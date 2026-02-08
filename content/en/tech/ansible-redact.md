---
author: "Harshanu"
title: "Hide sensitive data in Ansible verbose logs"
date: 2023-11-17
description: "A Guide to redact sensitive data in Ansible logs even when verbose flags are turned on"
tags: ["ansible", "secrets", "logs", "verbose", "sensitive", "redact", "debug", "callback"]
thumbnail: https://photos.harshanu.space/api/v1/t/25d379c065d5e00dffea28754ac7330271fa0b64/081gaa0s/fit_1280
---

## Introduction
In the realm of infrastructure automation, Ansible has emerged as a powerful tool, enabling users to manage and configure complex IT environments with ease. However, one of the persistent challenges in automation is the secure handling of sensitive information, such as passwords and API keys. When verbose logging is enabled in Ansible playbooks, these secrets could potentially be exposed in the logs, leading to significant security risks.

To address this concern, callback plugins offer a viable solution for masking sensitive information in Ansible playbook logs. Callback plugins are custom modules that intercept and modify Ansible task output, providing a mechanism to sanitize logs before they are displayed or stored.

## Why can't Ansible mask all secrets from the logs?
In ideal scenarios we expecte Ansible to mask secrets such as tokens/passwords in the logs. Unfortunately, it's only applicable to limited module attributes such as `url_password` in `get_url`/`uri` modules. If you pass tokens/passwords in `Authorization`/`body` of HTTP requests, it is shown in plain text. 

You could find a discussion [here](https://github.com/ansible/ansible/issues/73048) and the response from Ansible team below::

{{< highlight go >}}
There is no mechanism for optional no_log as dictated by a module. Instead, if you know you are passing sensitive data by a means not explicitly marked as no_log, you should use no_log: true on your task.

If you have further questions please stop by IRC or the mailing list:

    IRC: #ansible on irc.freenode.net
    mailing list: https://groups.google.com/forum/#!forum/ansible-project

{{< /highlight >}}

## Implementing a Callback Plugin for Secret Masking

Designing a callback plugin for secret masking involves several key steps:

1. #### Identifying Sensitive Data: 
The first step is to identify the specific data elements that need to be protected, such as passwords, API keys, and other credentials. This may involve reviewing the playbook tasks and identifying variables or data structures that contain sensitive information.

2. #### Defining Masking Rules: 
Once sensitive data elements are identified, clear masking rules should be established. These rules define how the sensitive data should be transformed to conceal its original value. Common masking techniques include replacing sensitive characters with asterisks (********) or with some text to indicate it is redacted or completely removing the sensitive data from the log output.

3. #### Developing the Callback Plugin: 
The core of the callback plugin lies in its ability to intercept and modify task output. The plugin should implement methods that are triggered when specific task events occur, such as task execution or task completion. These methods should parse the task output, identify sensitive data based on the defined masking rules, and apply the appropriate masking transformations.

4. #### Integrating the Callback Plugin: 
To utilize the callback plugin, it needs to be integrated into the Ansible environment. This typically involves placing the plugin code in a designated plugin directory and configuring Ansible to use the plugin.

## An example snippet of callback plugin redacting Authorization headers

Create a new Python script for your callback plugin. Let's call it redactsensitivedata.py. This script will contain the logic to filter or mask sensitive data of Authorization headers and `http` body which also sends secrets with `HTTP POST` method.

```shell
from ansible.plugins.callback.default import CallbackModule as CallbackModule_default
import os, collections

class CallbackModule(CallbackModule_default):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'stdout'
    CALLBACK_NAME = 'redactsensitivedata'

    def __init__(self, display=None):
        super(CallbackModule, self).__init__()

    def redact_sensitive_data(self, result):
        ret = {}
        ## result.iteritems() for python2.x
        for key, value in result.items():
            if isinstance(value, collections.Mapping):
                ret[key] = self.redact_sensitive_data(value)
            else:
                ## this can keep on growing or we can have an array of secrets as constants
                ## which can REDACT the sensitive info. 
                if "Authorization" in key or "body" in key:
                    ret[key] = "REDACTED"
                else:
                    ret[key] = value
        return ret

    def _dump_results(self, result, indent=None, sort_keys=True, keep_invocation=False):
        return super(CallbackModule, self)._dump_results(self.redact_sensitive_data(result), indent, sort_keys, keep_invocation)
```

Configure Ansible to Use the Callback Plugin

```shell
[defaults]
callback_whitelist = redactsensitivedata
```

Test run of ansible redacting Authorization headers and HTTP body..
```shell
HTTP BODY info redacted:
"module_args": {
            "attributes": null, 
            "backup": null, 
            "body": " REDACTED", 
            "body_format": " REDACTED", 
            "client_cert": null, 
            "client_key": null, 
            "content": null, 
            "creates": null, 
            "delimiter": null, 
 
Authorization headers redacted:
  "group": null, 
            "headers": {
                "Authorization": "Bearer REDACTED"
            }, 
```

## Benefits of Using Callback Plugins for Secret Masking

Utilizing callback plugins for secret masking offers several advantages:

1. #### Enhanced Security: 
By intercepting and modifying task output, callback plugins can effectively prevent sensitive information from being exposed in Ansible logs, significantly reducing the risk of accidental disclosure.

2. #### Granular Control: 
Callback plugins provide granular control over the masking process, allowing users to define specific masking rules for different types of sensitive data. This ensures that only the necessary information is masked, while maintaining the usefulness of log data for debugging and troubleshooting purposes.

3. #### Non-Intrusive Approach: 
Callback plugins integrate seamlessly into the Ansible workflow without modifying the core Ansible code. This makes them easy to implement and maintain, minimizing the impact on existing playbooks and configurations.


## Another approach using no_log

while callback plugin is a neat approach, it doesn't redact all the secrets. It needs to be updated reactively for each identified secret & compatibility with Ansible and Python versions. In addition to this, Ansible Tower/AWX uses it's own stdout callback plugin. One way you could overcome this with `no_log` flag set to `true` part of module. 

```shell
- name: A Rest service
  ansible.builtin.uri:
    url: "https://example.com/rest/latest/apples"
    body_format: json
    headers:
        Authorization: "Bearer {{ secret_token }}"
  register: api_response
  no_log: true
  ignore_errors: true  

- name: Print the output of api_response
  debug:
    msg: "{{ ( api_response ~ '') | replace(secret_token, 'REDACTED') }}"
  failed_when: api_response.failed == true
```

We suppress the output of `uri` module & print in the following `debug` task by redacting `secret_token` in the logs. 

While this is an easy workaround compared to `callback`, it needs to be repeated for each module printing secrets. 

## Conclusion

Callback plugins offer a powerful and flexible approach to safeguarding sensitive information in Ansible playbook logs. By implementing custom masking rules and integrating the plugin into the Ansible environment, users can effectively protect their secrets while maintaining the functionality of verbose logging for system monitoring and troubleshooting.