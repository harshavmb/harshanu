---
author: "Harshanu"
title: "Wazuh SSO with Zitadel using SAML 2.0"
date: 2026-04-06
description: "A practical guide to integrating Wazuh Dashboard SSO with Zitadel as SAML Identity Provider, including role injection via Zitadel Actions"
tags: ["wazuh", "zitadel", "saml", "sso", "opensearch", "security", "idp", "siem"]
thumbnail: "https://photos.harshanu.space/api/v1/t/034f9c7b9d088f235cf02df615ca7bf7464931ed/081gaa0s/fit_2048"
---

## Introduction

Wazuh is a solid open-source SIEM, but when it comes to SSO the documentation is thin. The official docs cover Keycloak, OneLogin and a few others but not Zitadel. There is even an [open issue](https://github.com/wazuh/wazuh/issues/18630) requesting Zitadel support and it has been sitting there for a while.

I run Zitadel as my identity provider and needed SAML-based SSO for my Wazuh Dashboard. The biggest pain point was that Zitadel does NOT include project roles in SAML assertions by default. Unlike OIDC where you get `urn:zitadel:iam:org:project:roles` for free, SAML gets nothing. You have to create a Zitadel Action to inject roles into the SAML response. Many people get stuck at this exact point — SSO login works but the user lands with zero permissions because the roles never arrive.

This post covers the full integration end to end: generating the SP metadata XML, configuring Zitadel with the SAML app, injecting roles via Actions and Flows, mapping those roles to Wazuh permissions on both OpenSearch and the Wazuh Manager API layers, and the config files you need to touch.

## How It Works

The SAML flow between the browser, Wazuh and Zitadel looks like this:

```
User Browser
    │
    │  1. Access Wazuh Dashboard
    ▼
┌──────────────────────────┐
│  Wazuh Dashboard         │
│  (OpenSearch Security)   │
│                          │
│  Detects no session →    │
│  Sends SAML AuthnRequest │
└────────────┬─────────────┘
             │
             │  2. Redirect to Zitadel
             ▼
┌──────────────────────────┐
│  Zitadel IdP             │
│                          │
│  User authenticates      │
│  (password, MFA, etc.)   │
└────────────┬─────────────┘
             │
             │  3. SAML Response (HTTP POST)
             │     with roles in assertion
             ▼
┌──────────────────────────┐
│  Wazuh ACS endpoint      │
│  /_opendistro/           │
│    _security/saml/acs    │
│                          │
│  Validates assertion     │
│  Maps roles →            │
│    all_access or         │
│    kibana_read_only      │
│                          │
│  Creates session         │
└──────────────────────────┘
```

The critical piece in step 3 is that the SAML assertion must contain a `Roles` attribute with the user's project roles from Zitadel. Without that, OpenSearch has no idea what backend roles to assign and the user gets an empty session.

## Wazuh's Two Permission Layers

This is something that trips people up. Wazuh has **two separate permission layers** and both need to be configured.

| Layer | What It Controls | Where It's Configured |
|-------|------------------|-----------------------|
| **OpenSearch Security** (port 9200) | Index and cluster access — read/write data, dashboards | `roles_mapping.yml` |
| **Wazuh Manager API** (port 55000) | Application-level permissions — deploy agents, edit rules/decoders | Security rules via REST API |

If you only configure OpenSearch Security, users can log in and see dashboards but they cannot deploy agents, edit rules or do anything that hits the Wazuh Manager API. You need both.

### Role Mapping

| Zitadel Role | Backend Role | OpenSearch Role | Access Level |
|-------------|-------------|-----------------|-------------|
| `wazuh-admins` | `wazuh-admins` | `all_access` | Full index/cluster access |
| `wazuh-readonly` | `wazuh-readonly` | `kibana_read_only` | Read-only dashboards |

For the Wazuh Manager API layer:

| Backend Role | Wazuh API Role | Access Level |
|-------------|---------------|-------------|
| `wazuh-admins` | `administrator` | Full app access (agent:create, rules, decoders, etc.) |

## Step 1: Generate the SP Metadata XML

Wazuh does not expose a SAML metadata URL like some other service providers do. You have to create the metadata XML yourself and import it into Zitadel when creating the SAML application.

Here is the SP metadata XML. Replace the URLs with your actual Wazuh Dashboard URL:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
                     entityID="wazuh-saml">
  <md:SPSSODescriptor
      AuthnRequestsSigned="false"
      WantAssertionsSigned="true"
      protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <md:SingleLogoutService
        Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
        Location="https://wazuh.example.net/_opendistro/_security/saml/logout" />
    <md:NameIDFormat>urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified</md:NameIDFormat>
    <md:AssertionConsumerService
        Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
        Location="https://wazuh.example.net/_opendistro/_security/saml/acs"
        index="0"
        isDefault="true" />
    <md:AssertionConsumerService
        Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
        Location="https://wazuh.example.net/_opendistro/_security/saml/acs/idpinitiated"
        index="1" />
  </md:SPSSODescriptor>
</md:EntityDescriptor>
```

A few things to note:

- `entityID` is `wazuh-saml`. This is the SP identifier and it has to match exactly in the OpenSearch security config later.
- `WantAssertionsSigned="true"` means Wazuh expects Zitadel to sign the assertion.
- There are two `AssertionConsumerService` entries. The first is the standard ACS for SP-initiated login. The second is for IdP-initiated login (index 1).
- The `SingleLogoutService` is included but we will strip it from the IdP metadata later to avoid logout redirect issues.

Save this as `wazuh-sp-metadata.xml`.

## Step 2: Configure Zitadel

### 2a. Create Roles in Your Zitadel Project

Go to **Zitadel Console** → **Projects** → select your project → **Roles** tab.

Create two roles:

| Key | Display Name | Group |
|-----|-------------|-------|
| `wazuh-admins` | Wazuh Administrators | wazuh |
| `wazuh-readonly` | Wazuh Read Only | wazuh |

### 2b. Create the SAML Application

1. Go to **Projects** → your project → **Applications**
2. Click **New** → enter name: `Wazuh SIEM` → select **SAML** type → **Continue**
3. Choose **Upload metadata XML** and upload the `wazuh-sp-metadata.xml` from Step 1
4. Click **Continue** → **Create**

### 2c. Assign Roles to Users

Go to **Projects** → your project → **Authorizations** → **New** → search for the user → select `wazuh-admins` or `wazuh-readonly` → **Save**.

### 2d. Create a Zitadel Action for SAML Role Injection

This is the part that most people miss and where hours of debugging happen.

Zitadel does **not** send project roles in SAML assertions by default. With OIDC you add the `urn:zitadel:iam:org:project:roles` scope and roles show up in the token. SAML has no such mechanism. You must create a **Zitadel Action** that reads the user's granted roles and injects them as a custom SAML attribute.

Go to **Zitadel Console** → **Actions** → click **+ New**.

**Name:** `addRolesToSAML` (this must match the function name in the script)

**Script:**

```javascript
function addRolesToSAML(ctx, api) {
    if (ctx.v1.user.grants == undefined || ctx.v1.user.grants.count == 0) {
        return;
    }
    let roles = [];
    ctx.v1.user.grants.grants.forEach(grant => {
        grant.roles.forEach(role => {
            roles.push(role);
        });
    });
    api.v1.attributes.setCustomAttribute('Roles', '', ...roles);
}
```

Click **Save**.

What this does: It iterates over all project role grants for the authenticating user and pushes each role name into an array. Then it calls `setCustomAttribute('Roles', '', ...roles)` which adds a `Roles` attribute to the SAML assertion. The first argument is the attribute name, the second is the namespace (empty string works fine), and the rest are the values spread as individual arguments.

### 2e. Attach the Action to the SAML Flow

Still on the **Actions** page, go to the **Flows** section → click **+ New**.

| Setting | Value |
|---------|-------|
| **Flow Type** | `Complement SAMLResponse` |
| **Trigger Type** | `Pre SAMLResponse creation` |
| **Actions** | Check `addRolesToSAML` |

Click **Save**.

This tells Zitadel to run the `addRolesToSAML` function every time it builds a SAML response, right before sending it back to the service provider. The roles will now appear in the assertion as:

```xml
<saml:Attribute Name="Roles">
    <saml:AttributeValue>wazuh-admins</saml:AttributeValue>
</saml:Attribute>
```

### 2f. Verify Zitadel Configuration

After creating the app, make sure Zitadel's IdP metadata is accessible:

```bash
curl -s https://auth.example.net/saml/v2/metadata | head -5
```

You should see an XML document starting with `<md:EntityDescriptor>`. Download this metadata — you will need it in the next step.

## Step 3: Configure OpenSearch Security

This is the Wazuh side. You need to modify three files inside the Wazuh indexer node.

### config.yml — Add SAML Auth Domain

The OpenSearch security config lives at `/etc/wazuh-indexer/opensearch-security/config.yml`. You need to add a `saml_auth_domain` alongside the existing basic auth domain.

```yaml
---
_meta:
  type: "config"
  config_version: 2

config:
  dynamic:
    http:
      anonymous_auth_enabled: false
    authc:
      basic_internal_auth_domain:
        description: "Authenticate via HTTP Basic against internal users database"
        http_enabled: true
        transport_enabled: true
        order: 0
        http_authenticator:
          type: "basic"
          challenge: false
        authentication_backend:
          type: "intern"
      saml_auth_domain:
        http_enabled: true
        transport_enabled: false
        order: 1
        http_authenticator:
          type: saml
          challenge: true
          config:
            idp:
              metadata_file: "/etc/wazuh-indexer/opensearch-security/idp-metadata.xml"
              entity_id: "https://auth.example.net/saml/v2/metadata"
            sp:
              entity_id: "wazuh-saml"
              metadata_file: "/etc/wazuh-indexer/opensearch-security/sp-metadata.xml"
            kibana_url: "https://wazuh.example.net"
            roles_key: "Roles"
            exchange_key: "YOUR_64_CHAR_EXCHANGE_KEY_HERE"
        authentication_backend:
          type: noop
```

Key things:

- `challenge: false` on basic auth and `challenge: true` on SAML. This means the dashboard shows both login options but SAML is the one that triggers the redirect.
- `roles_key: "Roles"` must match exactly what the Zitadel Action sets via `setCustomAttribute('Roles', ...)`. If this does not match, roles will silently be empty.
- `exchange_key` is a random 64-character hex string. Generate one with `openssl rand -hex 32`. This is used for encrypting the SAML exchange between the dashboard and the indexer.
- `sp.entity_id: "wazuh-saml"` must match the `entityID` in your SP metadata XML.
- `idp.entity_id` must match the `entityID` in Zitadel's IdP metadata.

### Strip SLO Endpoints from IdP Metadata

Before placing the downloaded IdP metadata, remove the `SingleLogoutService` entries from it. If you leave them in, clicking logout in the Wazuh Dashboard will redirect you to Zitadel's logout page and you will not land back on the Wazuh login screen.

```bash
# Remove SLO endpoints from the downloaded IdP metadata
sed -i "s|<SingleLogoutService[^>]*>[^<]*</SingleLogoutService>||g" idp-metadata.xml
sed -i "s|<SingleLogoutService[^/]*/>[[:space:]]*||g" idp-metadata.xml
```

Place both metadata files in the OpenSearch security directory:

```bash
cp idp-metadata.xml /etc/wazuh-indexer/opensearch-security/idp-metadata.xml
cp wazuh-sp-metadata.xml /etc/wazuh-indexer/opensearch-security/sp-metadata.xml

chown wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/opensearch-security/idp-metadata.xml
chown wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/opensearch-security/sp-metadata.xml
chmod 0640 /etc/wazuh-indexer/opensearch-security/*.xml
```

### roles_mapping.yml — Map Zitadel Roles to OpenSearch Roles

You need to add your Zitadel backend roles to the `all_access` and `kibana_read_only` role mappings. Find the `all_access` section in `/etc/wazuh-indexer/opensearch-security/roles_mapping.yml` and add `wazuh-admins` to `backend_roles`:

```yaml
all_access:
  reserved: false
  hidden: false
  backend_roles:
  - "admin"
  - "wazuh-admins"
  description: "Maps admin to all_access"

kibana_read_only:
  reserved: false
  hidden: false
  backend_roles:
  - "wazuh-readonly"
  hosts: []
  users: []
  and_backend_roles: []
  description: "Maps wazuh-readonly to kibana_read_only"
```

**Important:** Set `reserved: false` on `all_access`. By default it is `reserved: true` and you cannot modify it.

### Apply Changes with securityadmin

After modifying the config files, you must run `securityadmin` to push the changes into the OpenSearch security index:

```bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk/

# Apply config.yml
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -f /etc/wazuh-indexer/opensearch-security/config.yml \
  -icl \
  -key /etc/wazuh-indexer/certs/admin-key.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -h localhost -nhnv

# Apply roles_mapping.yml
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -f /etc/wazuh-indexer/opensearch-security/roles_mapping.yml \
  -icl \
  -key /etc/wazuh-indexer/certs/admin-key.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -h localhost -nhnv
```

If `securityadmin` fails, check that the admin certificates exist and that the Wazuh indexer is running.

## Step 4: Configure the Wazuh Dashboard

Edit `/etc/wazuh-dashboard/opensearch_dashboards.yml` and add the following:

```yaml
# SAML SSO
opensearch_security.auth.multiple_auth_enabled: true
opensearch_security.auth.type: ["basicauth","saml"]
server.xsrf.allowlist: ["/_opendistro/_security/saml/acs", "/_opendistro/_security/saml/logout", "/_opendistro/_security/saml/acs/idpinitiated"]
opensearch_security.auth.logout_url: "https://wazuh.example.net"
```

- `multiple_auth_enabled: true` lets both basic auth and SAML appear on the login page.
- The `xsrf.allowlist` entries are required because the SAML ACS endpoint receives POST requests from the IdP and they would be blocked by XSRF protection otherwise.
- `logout_url` is where users land after clicking logout. Without this, logout sends you to the IdP.

Also edit `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` and set `run_as: true` on the Wazuh API host entry. This allows the dashboard to forward the authenticated user's backend roles to the Wazuh Manager API for role-based access.

Then restart the dashboard:

```bash
systemctl restart wazuh-dashboard
```

## Step 5: Create the Wazuh Manager API Security Rule

This is the second permission layer. Even if OpenSearch grants `all_access`, the Wazuh Manager API needs its own mapping from `wazuh-admins` to the `administrator` role. Without this, SSO users can see dashboards but cannot deploy agents, edit rules or access the API features.

```bash
# Authenticate to the Wazuh Manager API
TOKEN=$(curl -sk -u "wazuh-wui:YOUR_API_PASSWORD" \
  -X POST https://localhost:55000/security/user/authenticate \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['token'])")

# Create a security rule: wazuh-admins backend_role → administrator
RULE_ID=$(curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST https://localhost:55000/security/rules \
  -d '{"name": "wui_wazuh_admins", "rule": {"FIND": {"backend_roles": "wazuh-admins"}}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['affected_items'][0]['id'])")

# Link the rule to the administrator role (role ID 1)
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "https://localhost:55000/security/roles/1/rules?rule_ids=$RULE_ID"
```

Verify the rules:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  https://localhost:55000/security/rules | python3 -m json.tool
```

You should see `wui_elastic_admin`, `wui_opensearch_admin` and your new `wui_wazuh_admins` rule, all linked to role 1 (administrator).

## Step 6: Test

1. Open an incognito window
2. Navigate to your Wazuh Dashboard URL
3. You should see both a basic auth login form and a **Log in with single sign-on** button
4. Click the SSO button — Zitadel login page appears
5. Authenticate with your Zitadel credentials (password, MFA, etc.)
6. You should be redirected back to the Wazuh Dashboard, logged in with full permissions

If all went well, you are done. If not, read on.

## Files Modified — Summary

| File | Change |
|------|--------|
| `/etc/wazuh-indexer/opensearch-security/config.yml` | Added `saml_auth_domain`, set basic auth `challenge: false` |
| `/etc/wazuh-indexer/opensearch-security/roles_mapping.yml` | Added `wazuh-admins` → `all_access`, `wazuh-readonly` → `kibana_read_only` |
| `/etc/wazuh-indexer/opensearch-security/idp-metadata.xml` | Zitadel IdP metadata (downloaded, SLO stripped) |
| `/etc/wazuh-indexer/opensearch-security/sp-metadata.xml` | Wazuh SP metadata (generated) |
| `/etc/wazuh-dashboard/opensearch_dashboards.yml` | Added SAML auth type, XSRF allowlist, logout URL |
| `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` | Set `run_as: true` |

## Troubleshooting

### SSO button not showing on the login page

The dashboard config is missing the SAML settings. Check that `opensearch_dashboards.yml` has all four SAML lines from Step 4 and restart the dashboard.

```bash
grep -A2 'auth.type' /etc/wazuh-dashboard/opensearch_dashboards.yml
systemctl restart wazuh-dashboard
```

### "Invalid SAML response" or redirect loop

Entity ID mismatch. The `sp.entity_id` in `config.yml` must match the `entityID` in the SP metadata you imported into Zitadel. The `idp.entity_id` must match what Zitadel returns in its metadata. Check the OpenSearch logs:

```bash
tail -100 /var/log/wazuh-indexer/wazuh-cluster.log | grep -i saml
```

### SSO login works but user has no permissions (backend_roles=[])

This is the most common issue. It means the Zitadel Action is either not created, not attached to the SAML flow, or the user does not have a role grant in the project.

Checklist:
1. The Action function name `addRolesToSAML` matches the Action name in Zitadel
2. The Action is attached to Flow `Complement SAMLResponse` → Trigger `Pre SAMLResponse creation`
3. The user has a role authorization in **Projects** → your project → **Authorizations**
4. The `roles_key` in `config.yml` is set to `Roles` (capital R) matching the Action's `setCustomAttribute('Roles', ...)`

To debug, enable SAML logging temporarily:

```bash
echo "logger.com.amazon.dlic.auth.http.saml: debug" >> /etc/wazuh-indexer/opensearch.yml
systemctl restart wazuh-indexer

# Try an SSO login, then check logs
grep -i "SAMLResponse\|Token" /var/log/wazuh-indexer/wazuh-cluster.log | tail -20

# Clean up after debugging
sed -i "/^logger.com.amazon.dlic.auth.http.saml/d" /etc/wazuh-indexer/opensearch.yml
systemctl restart wazuh-indexer
```

### SSO login works but user cannot deploy agents or edit rules

This means OpenSearch Security (Layer 1) is fine but the Wazuh Manager API (Layer 2) is not configured. Go back to Step 5 and create the security rule mapping `wazuh-admins` to the administrator role.

You can verify by checking the API logs:

```bash
grep "run_as" /var/ossec/logs/api.log | tail -5
```

Look for `backend_roles: ["wazuh-admins"]` in the log entries. If `backend_roles` is empty, the roles are not arriving from the SAML assertion.

### Logout redirects to Zitadel and does not come back

The IdP metadata still has `SingleLogoutService` entries. Strip them as shown in Step 3, re-push the metadata and restart the indexer. Also make sure `opensearch_security.auth.logout_url` is set in the dashboard config.

### Basic auth (internal users) stopped working

It should not. The config preserves basic auth as order 0 with `challenge: false`. The dashboard is set to `["basicauth","saml"]` so both login methods appear. If basic auth broke, check that the `basic_internal_auth_domain` section is still present in `config.yml` and re-run `securityadmin`.

## Conclusion

The hardest part of this integration is not the SAML config itself — that is mostly standard OpenSearch Security stuff. The hard part is getting roles into the SAML assertion from Zitadel. The Action + Flow combination is not documented anywhere in the context of Wazuh, and without it the whole thing silently fails. Then the dual permission layer in Wazuh (OpenSearch Security + Manager API) adds another place where things can go wrong.

Once you get these two pieces right, it works well. Users log in via Zitadel, roles flow through the SAML assertion into OpenSearch backend roles, and from there into the Wazuh Manager API permissions. Both basic auth and SSO coexist on the login page so you always have a fallback.

## Disclaimer

Part of this post was assisted by AI. You may need to modify Wazuh/Zitadel config changes according to your current infrastructure
