---
author: "Harshanu"
title: "Supabase Blocked in India: Why Routing Your Traffic Through a Random Proxy Is a Terrible Idea"
date: 2026-03-01
description: "Supabase is blocked in India. Desperate vibecoders are routing all their API traffic through third-party proxies like JioBase. Here is why that is a security disaster and what you should do instead."
tags: ["supabase", "security", "proxy", "jiobase", "india", "pocketbase", "self-hosted", "vibecoding", "dns", "tls"]
thumbnail: https://photos.harshanu.space/api/v1/t/524846017ea5a4d144fd424b8c6b9d1a24a1d206/081gaa0s/fit_2048
---

## Introduction

On February 24, 2026, Indian ISPs including Jio, Airtel, and ACT Fibernet started DNS-blocking `*.supabase.co` following a government order under Section 69A of the IT Act. Every Supabase-powered app in India broke overnight. REST queries, auth flows, file uploads, realtime WebSocket connections, all started timing out with `ERR_CONNECTION_TIMED_OUT`. The Supabase dashboard at `supabase.com` still works. The API domain `supabase.co` does not.

Thousands of developers panicked. Many of them are vibecoders, people who build apps by prompting AI to generate code without fully understanding what happens under the hood. They know how to `createClient()` but not how DNS resolution or TLS handshakes work.

Within days, a proxy service called [JioBase](https://jiobase.com/) appeared. Built by an indie developer named Sunith VS from Kerala, it promises to fix the problem with a one-line code change. Replace `xyz.supabase.co` with `xyz.jiobase.com` and your app works again. It even has a polished landing page, docs, a dashboard, and 15 blog posts about DNS poisoning. The [codebase](https://github.com/sunithvs/jiobase) is open source. Sounds great.

It is not great. It is a security disaster waiting to happen.

This post explains why routing your production Supabase traffic through any third-party proxy is dangerous, what exactly is at risk, and what you should do instead.

## How the Block Works

Before diving into the proxy problem, it helps to understand the block itself. Indian ISPs are using DNS poisoning to block Supabase. When your app tries to resolve `yourproject.supabase.co`, the ISP's DNS resolver intercepts the query and returns a sinkhole IP (commonly `49.44.79.236`) instead of Supabase's actual IP address. The request goes to a dead-end and times out.

```shell
# On an affected ISP (poisoned DNS)
$ nslookup yourproject.supabase.co
Address: 49.44.79.236        ← sinkhole, NOT Supabase

# On Cloudflare DNS (correct)
$ nslookup yourproject.supabase.co 1.1.1.1
Address: 104.18.x.x          ← real Supabase IP
```

Some ISPs also perform Deep Packet Inspection (DPI), inspecting the SNI field in the TLS handshake. Even if you bypass DNS poisoning by using `1.1.1.1`, the ISP sees `supabase.co` in the SNI and drops the connection. This is why simply changing DNS does not reliably fix the problem for end users.

A reverse proxy solves this because the client connects to `yourapp.jiobase.com` (or your own domain), and the ISP never sees `supabase.co` in any DNS query or TLS handshake. The proxy forwards the request to Supabase on the server side, outside the ISP's reach.

The technical approach is sound. The problem is *who* runs the proxy.

## How JioBase Works

JioBase is a managed reverse proxy running on Cloudflare Workers. The [source code](https://github.com/sunithvs/jiobase) shows three apps: a web frontend (Svelte), an API (Hono on Cloudflare Workers), and the proxy worker itself. Here is the flow:

```shell
[Your App / Browser]
        |
        | HTTPS (SNI: yourapp.jiobase.com)
        |
   [Cloudflare Edge]
        |
        | TLS terminated here
        |
   [JioBase Proxy Worker]
        |
        | HTTPS (new connection to Supabase)
        |
   [Supabase API]
        | (*.supabase.co)
```

You sign up, paste your Supabase project URL, pick a slug, and get `yourapp.jiobase.com`. You replace the URL in your Supabase client initialization. The proxy forwards everything: REST, Auth, Storage, Realtime WebSockets, Edge Functions, GraphQL.

Looking at the [handler.ts](https://github.com/sunithvs/jiobase/blob/main/apps/proxy/src/handler.ts), the proxy:

1. Receives the incoming request
2. Rewrites the URL to point at your Supabase project
3. Copies all headers (including `Authorization` with your JWT)
4. Forwards the request body verbatim
5. Returns the response with an `X-Proxied-By: JioBase` header
6. Writes analytics data (app ID, HTTP method, path, country, status code)

The [websocket.ts](https://github.com/sunithvs/jiobase/blob/main/apps/proxy/src/websocket.ts) does the same for WebSocket connections, relaying messages bidirectionally between client and Supabase.

The service claims it does not log, inspect, or store transit data. Their [privacy policy](https://jiobase.com/privacy) says so. Their [terms page](https://jiobase.com/terms) says so. The open source code appears to confirm it.

Now let me explain why none of that matters.

## The Security Problems

### 1. TLS Termination: Your Data Is Decrypted at the Proxy

This is the most fundamental issue and the one most vibecoders do not understand. When your app sends an HTTPS request to `yourapp.jiobase.com`, TLS is terminated at Cloudflare's edge. This means the request is **decrypted in memory** at the proxy layer before being re-encrypted and forwarded to Supabase.

JioBase's own privacy policy admits this:

> *"TLS disclosure: Cloudflare terminates TLS at the edge as part of its standard infrastructure. This means Transit Data is briefly decrypted in Cloudflare's memory during forwarding."*

This is how all reverse proxies work. It is not unique to JioBase. But it means that **everything your app sends to Supabase passes through the proxy in plaintext**. Every database query, every auth token, every file upload, every WebSocket message. The proxy operator (and Cloudflare) have technical access to all of it at the moment of forwarding.

### 2. Your Supabase JWT Tokens Flow Through the Proxy

Every authenticated request to Supabase includes an `Authorization: Bearer <JWT>` header. This JWT contains user identity, role, and session information. The proxy forwards this header verbatim to Supabase:

```typescript
// From JioBase's handler.ts
const headers = new Headers(request.headers);
headers.set('Host', upstreamUrl.hostname);
// ... Authorization header is NOT stripped
// It passes straight through to Supabase
```

Anyone with the ability to log or inspect these headers at the proxy layer can:

- **Impersonate any user** of your application by replaying their JWT
- **Access any data** the user has permission to read or write
- **Perform actions** as the user (create, update, delete records)
- **Exfiltrate refresh tokens** to maintain long-term access

JWTs typically expire in 1 hour for Supabase, but refresh tokens last much longer. If a refresh token is captured, the attacker can mint new access tokens indefinitely until the user's session is explicitly revoked.

### 3. Your Supabase Anon Key Is Exposed

When you initialize the Supabase client, you pass the anon key:

```typescript
const supabase = createClient(
  'https://myapp.jiobase.com',  // proxy URL
  'your-anon-key'               // this key is sent with EVERY request
)
```

The anon key is sent as the `apikey` header with every request. It passes through the proxy in plaintext. While the anon key is designed to be public-facing, it defines the baseline permissions for your project. Combined with the ability to capture user JWTs, an attacker has everything needed to interact with your Supabase project as any user.

If you use the **service role key** anywhere in client-side code (which you should never do, but vibecoders sometimes do because the AI told them to), that key also passes through the proxy. The service role key bypasses Row Level Security entirely.

### 4. All Your Users' PII Flows Through the Proxy

Think about what your Supabase queries contain:

- **Auth endpoints**: Email addresses, passwords (during signup/signin), phone numbers, OAuth tokens from Google/GitHub/etc.
- **Database queries**: Names, addresses, payment info, health data, whatever your app stores
- **Storage uploads**: Profile photos, documents, ID cards, anything users upload
- **Realtime messages**: Chat messages, presence data, live notifications

All of this data passes through JioBase's Cloudflare Worker in plaintext during TLS termination. Even if JioBase's code does not log it today, there is no technical guarantee it will not log it tomorrow.

### 5. Open Source Does Not Mean Safe

JioBase's code is [on GitHub](https://github.com/sunithvs/jiobase). That is good. But here is the problem: **you have no way to verify that the code on GitHub is the same code running on Cloudflare Workers**.

There is no reproducible build. There is no deployment attestation. There is no way for you to verify what is actually executing when your traffic hits `jiobase.com`. The operator could deploy a version with an extra line that logs every `Authorization` header to a private endpoint, and you would never know.

This is not a JioBase-specific problem. It applies to every hosted service. But it is especially dangerous when that service sees all your authentication tokens and user data, while having no audit trail, no SOC 2 certification, no bug bounty program, and no legal entity beyond a single individual developer.

### 6. Analytics Already Track Request Metadata

The proxy code already writes analytics data on every request:

```typescript
env.ANALYTICS.writeDataPoint({
  blobs: [
    config.appId,
    request.method,
    url.pathname,
    request.headers.get('cf-ipcountry') || 'unknown',
  ],
  doubles: [upstreamResponse.status],
  indexes: [config.appId],
});
```

This logs your app ID, the HTTP method, the full URL path (which often contains table names and query parameters), the user's country, and the response status code. The path alone can reveal which tables your users are querying and with what filters. This is metadata, not payload, but it is still far more information than most developers realize they are sharing.

### 7. Single Point of Failure and Trust

JioBase is operated by one person. From the [terms of service](https://jiobase.com/terms):

> *"These Terms of Service constitute a legally binding agreement between you and Sunith VS, an individual developer operating the project JioBase."*

There is no company, no team, no legal entity. If Sunith decides to shut down the service, your app breaks. If his Cloudflare account gets compromised, every JioBase user's traffic is exposed. If he gets a legal notice from the Indian government to log traffic (which is entirely possible under the IT Act), he must comply or shut down.

The terms explicitly say:

> *"Our total aggregate liability... shall not exceed... one thousand Indian Rupees (INR 1,000)."*

That is roughly $12. If your user data gets leaked through a compromised proxy, your maximum recovery from JioBase is $12.

### 8. OAuth Flows Through the Proxy Are Especially Dangerous

The proxy rewrites `Location` headers to keep OAuth redirects working through the proxy:

```typescript
if (locUrl.hostname === supabaseHost) {
  locUrl.hostname = url.hostname;
  responseHeaders.set('Location', locUrl.toString());
}
```

This means OAuth callbacks (Google, GitHub, etc.) flow through the proxy. OAuth tokens, authorization codes, and state parameters all pass through the proxy in plaintext during the redirect chain. A compromised proxy could capture OAuth authorization codes and exchange them for access tokens to your users' Google or GitHub accounts.

### 9. The Domain Could Be Blocked Too

If the Indian government blocked `supabase.co`, there is nothing stopping them from blocking `jiobase.com` once it gains visibility. It is literally called "JioBase", and Jio is Reliance's brand. The [terms of service](https://jiobase.com/terms) even acknowledge this:

> *"ISPs or government authorities may at any time block or restrict access to JioBase domains (including `*.jiobase.com`)."*

You would be migrating from one blocked domain to another blocked domain, having given a third party access to all your traffic in the process.

## The Bigger Problem: Vibecoding Without Understanding Infrastructure

The Supabase block exposed a deeper issue. A huge number of Indian developers built production apps where the client (browser/mobile) connects directly to Supabase endpoints. They used Supabase as both the backend and the database with no server of their own. When Supabase got blocked, they had no fallback. Zero resilience.

This is the vibecoding trap. AI tools make it trivially easy to scaffold a full-stack app with Supabase in an afternoon. Cursor, Windsurf, Lovable, Bolt, or any other AI IDE will happily generate a Next.js app with Supabase Auth, database queries, and realtime subscriptions. It works. It ships. And then one day your cloud provider gets blocked in your primary market and you have no idea what to do because you never learned how hosting works.

The panic-driven rush to JioBase is a symptom. The disease is building production infrastructure you do not understand.

## What You Should Do Instead

### Option 1: Self-Hosted Cloudflare Worker (Free, 15 Minutes)

JioBase themselves offer a [worker generator tool](https://jiobase.com/tools/worker-generator) that creates a Cloudflare Worker script you can deploy on your own Cloudflare account. This is the same proxy concept, but **you** control the Cloudflare account, **you** deploy the code, and **you** own the DNS. No third party sees your traffic.

```shell
# The flow becomes:
[Your App] → [Your Cloudflare Worker] → [Supabase]
#                  ↑
#          YOU control this
```

The Cloudflare Workers free tier gives you 100,000 requests per day. That is 3 million per month. For most apps, this is more than enough. You get the same DNS-bypass benefit with zero trust delegation.

### Option 2: Your Own Reverse Proxy on a VPS ($3-5/month)

Spin up a cheap VPS on Hetzner ($3.29/month for CX22), DigitalOcean, or any provider and run nginx as a reverse proxy. You control the server, the DNS, and the TLS certificates.

```nginx
# /etc/nginx/sites-available/supabase-proxy
server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/api.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomain.com/privkey.pem;

    location / {
        proxy_pass https://yourproject.supabase.co;
        proxy_set_header Host yourproject.supabase.co;
        proxy_set_header X-Real-IP $remote_addr;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Point your domain's DNS to the VPS, get a Let's Encrypt certificate with `certbot`, and you have a proxy that nobody else controls.

### Option 3: PocketBase, the Self-Hosted Alternative

If you are building a new project or willing to migrate, consider [PocketBase](https://pocketbase.io/). It is an open-source backend that gives you most of what Supabase offers:

- SQLite database with a REST API
- Authentication (email/password, OAuth2)
- File storage
- Realtime subscriptions
- Admin dashboard

The key difference: **PocketBase is a single binary**. It is written in Go, compiles to a ~30MB executable, and runs on anything. A $3 VPS, a Raspberry Pi, a free-tier Oracle Cloud instance. It needs no Docker, no Kubernetes, no managed database.

```shell
# Download and run PocketBase
wget https://github.com/pocketbase/pocketbase/releases/download/v0.25.x/pocketbase_0.25.x_linux_amd64.zip
unzip pocketbase_0.25.x_linux_amd64.zip
./pocketbase serve --http=0.0.0.0:8090
```

That is it. You now have a backend with auth, database, file storage, and realtime. Put nginx or Caddy in front of it for TLS and you have a production-ready setup that no ISP can block because it runs on your own domain.

PocketBase is not a 1:1 Supabase replacement. It uses SQLite instead of PostgreSQL, it does not have Edge Functions, and its ecosystem is smaller. But for most apps that vibecoders are building (SaaS dashboards, chat apps, CRUD tools), it is more than enough.

And here is the important part: **you own the infrastructure**. No government order can block your app by blocking someone else's domain. If your VPS provider gets blocked, you move to another one. You have the data, you have the binary, you have control.

### Option 4: Add a Backend Server

The architecturally correct fix is to not expose your database to the client at all. Instead of your browser talking directly to Supabase, put a backend server in between:

```shell
[Browser] → [Your API Server] → [Supabase]
```

Your API server can run on any VPS, any cloud provider, or any serverless platform. It talks to Supabase using the service role key on the server side (where it belongs). Your client only talks to your API server. The ISP never sees `supabase.co` in any request.

This is how production apps are supposed to work. The "connect directly from the browser to the database" pattern that Supabase encourages is convenient for prototyping but creates exactly this kind of fragility.

## The Math on Risk

Let me lay this out plainly:

| Aspect | JioBase (SaaS Proxy) | Self-Hosted Proxy | PocketBase on VPS |
|---|---|---|---|
| Setup time | 5 minutes | 15-30 minutes | 30-60 minutes |
| Monthly cost | Free | Free (CF Workers) or $3-5 | $3-5 |
| Who sees your tokens | JioBase + Cloudflare | Cloudflare (your account) | Nobody |
| Who sees your data | JioBase + Cloudflare | Cloudflare (your account) | Nobody |
| Can operator log traffic | Yes (technically) | No (you are the operator) | No |
| Resilience to blocking | Low (jiobase.com can be blocked) | Medium (your domain) | High (your domain + server) |
| Vendor lock-in | Supabase + JioBase | Supabase | None |
| Liability cap | ₹1,000 (~$12) | N/A | N/A |

Saving 15 minutes of setup is not worth giving a stranger access to all your users' authentication tokens and personal data.

## Conclusion

The Supabase block in India is real and frustrating. Developers who depended on direct client-to-Supabase connections are stuck. But the answer is not to route all your traffic through a random proxy built by a stranger in 2 days and run on their personal Cloudflare account.

To be clear, I am not accusing JioBase's developer of malicious intent. The code is open source, the terms are transparent about limitations, and the project appears to be a genuine attempt to help. But good intentions do not make a service secure. The architecture of any SaaS proxy means your data is exposed at the proxy layer, and no privacy policy can change that technical reality.

If you are a vibecoder who got burned by this, use it as a learning moment. Spend a weekend understanding how DNS works, what TLS termination means, how to deploy a VPS, and how to run a reverse proxy. These are fundamental skills. You do not need to become a sysadmin, but you need to understand enough to make informed decisions about where your users' data flows.

Deploy your own Cloudflare Worker. Spin up a Hetzner VPS with nginx. Try PocketBase. Build something you control. The next time a cloud service gets blocked, you will be the one who has a fix in 15 minutes instead of the one desperately feeding your traffic to strangers.

## Disclaimer

This post is an educational analysis of the security implications of third-party proxy services. It is not legal advice. The author has no affiliation with Supabase, JioBase, Cloudflare, or PocketBase. The technical analysis is based on publicly available source code and documentation. Part of this post was assisted by AI.
