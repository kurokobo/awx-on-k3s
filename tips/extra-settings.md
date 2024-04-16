<!-- omit in toc -->
# Pass values from Secrets to `extra_settings`

Currently, AWX Operator does not ability to inject Secrets to `extra_settings`.

However, since Secrets are injectable for environment variables, we can indirectly inject values from Secrets into `extra_settings` by specifying values that refers to the environment variables.

<!-- omit in toc -->
## Table of Contents

- [Concept](#concept)
- [Simple examples](#simple-examples)
- [Complex examples](#complex-examples)
- [Appendix: Add extra settings by mounting settings files instead of using `extra_settings`](#appendix-add-extra-settings-by-mounting-settings-files-instead-of-using-extra_settings)
  - [Add extra settings by mounting ConfigMap](#add-extra-settings-by-mounting-configmap)
  - [Refer sensitive values from Secrets from the settings files](#refer-sensitive-values-from-secrets-from-the-settings-files)
  - [Embed sensitive values directly into the settings files](#embed-sensitive-values-directly-into-the-settings-files)

## Concept

Since the values for `extra_settings` will be embedded as Python code into `settings.py`, we can use any python code as values for `extra_settings`. So the key concepts are:

- Add environment variables for both task and web pod with `valueFrom` to inject values from Secrets to environment variables.
- Specify `os.getenv()` in values for `extra_settings`.

## Simple examples

This is a simple example to inject values from Secrets to `extra_settings`.

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...

  # Add new environment variables with "valueFrom" to task pod
  task_extra_env: |
    - name: AZUREAD_OAUTH2_KEY
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: key
    - name: AZUREAD_OAUTH2_SECRET
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: secret

  # Completely the same environment variables should be added to web pod
  web_extra_env: |
    - name: AZUREAD_OAUTH2_KEY
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: key
    - name: AZUREAD_OAUTH2_SECRET
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: secret

  # Specify "os.getenv()" that refers environment variables that newly added above
  extra_settings:
    - setting: SOCIAL_AUTH_AZUREAD_OAUTH2_KEY
      value: os.getenv("AZUREAD_OAUTH2_KEY")
    - setting: SOCIAL_AUTH_AZUREAD_OAUTH2_SECRET
      value: os.getenv("AZUREAD_OAUTH2_SECRET")
```

## Complex examples

This is an example to inject values from Secrets as a part of Dictionary. The Dictionary should be passed as multi-line strings.

> [!IMPORTANT]
> If we want to pass multi-line strings including `os.getenv()` as values for `extra_settings`, indentation have to be noted since the values will be embedded into ConfigMap that templated by Jinja, and ConfigMap has 4 spaces indentation per lines. We must not destroy the indentation of the ConfigMap.

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...

  # Add new environment variables with "valueFrom" to task pod
  task_extra_env: |
    - name: SAML_IDP_SIGNING_CERT
      valueFrom:
        secretKeyRef:
          name: awx-saml-idp-signing-cert
          key: x509cert

  # Completely the same environment variables should be added to web pod
  web_extra_env: |
    - name: SAML_IDP_SIGNING_CERT
      valueFrom:
        secretKeyRef:
          name: awx-saml-idp-signing-cert
          key: x509cert

  # Specify "os.getenv()" that refers environment variables that newly added above
  extra_settings:
    - setting: SOCIAL_AUTH_SAML_ENABLED_IDPS 
      # Define value as multi-line strings with 2 spaces indentation (by "|2"),
      # then pass the actual value with 6 spaces indentation
      # to add 4 leading spaces for each lines to keep indentation in ConfigMap correctly
      value: |2
            {
              "saml_ms_adfs": {
                "x509cert": os.getenv("SAML_IDP_SIGNING_CERT"),
                "attr_last_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
                "attr_first_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
                "attr_email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
                "attr_username": "name_id",
                "attr_user_permanent_id": "name_id",
                "entity_id": "https://adfs.domain.com/adfs/services/trust",
                "url": "https://adfs.domain.com/adfs/ls",
              }
            }
```

This will generate following ConfigMap. As you can see, to include all lines into under `settings`, 4 spaces indentation is important.

```yaml
---
apiVersion: v1
kind: ConfigMap
...
data:
  ...
  settings: |
    import os
    import socket
    ...
    SOCIAL_AUTH_SAML_ENABLED_IDPS =     {
      "saml_ms_adfs": {
        "x509cert": os.getenv("SAML_IDP_SIGNING_CERT"),
        "attr_last_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
        "attr_first_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
        "attr_email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
        "attr_username": "name_id",
        "attr_user_permanent_id": "name_id",
        "entity_id": "https://adfs.domain.com/adfs/services/trust",
        "url": "https://adfs.domain.com/adfs/ls",
      }
    }
```

> [!TIP]
> If you don't want to care about indentation, alternatively it is also possible to pass a Dictionary on a single line, as long as it follows Python syntax.

> [!TIP]
> The above example assumes that the certificate is stored in secret as a single line of text data, not including line breaks, headers, or footers.
>
> ```bash
> kubectl -n awx create secret generic awx-saml-idp-signing-cert \
>   --from-literal=x509cert=MIIDZTCC........6xGPxz26
> ```
>
> If your secrets contains whole the certificate, you can use `replace()` after `getenv()`, to remove line breaks, headers, and footers.
>
> ```python
> "x509cert": os.getenv("SAML_IDP_SIGNING_CERT").replace("\n", "").replace("-----BEGIN CERTIFICATE-----", "").replace("-----END CERTIFICATE----", ""),
> ```

## Appendix: Add extra settings by mounting settings files instead of using `extra_settings`

If you have a lot of settings to add, or want to pass dictionaries or lists, it might be hard to maintain them in `extra_settings`.

In that case, you can create settings files as pure Python code and mount them under `/etc/tower/conf.d` directory on the task and web pods, since AWX reads any `*.py` files in `/etc/tower/conf.d` directory as settings files on its start up.

### Add extra settings by mounting ConfigMap

This is an example `extra_settings.py` file that contain multiple extra settings.

```python
AD_HOC_COMMANDS = [
    "command",
    "shell",
    "yum",
    "apt",
    "apt_key",
    "apt_repository",
    "apt_rpm",
    "service",
    "group",
    "user",
    "mount",
    "ping",
    "selinux",
    "setup",
    "win_ping",
    "win_service",
    "win_updates",
    "win_group",
    "win_user",
]
AWX_ISOLATION_SHOW_PATHS = [
    "/etc/pki/ca-trust:/etc/pki/ca-trust:O",
    "/usr/share/pki:/usr/share/pki:O",
]
AWX_TASK_ENV = {
    "HTTPS_PROXY": "http://proxy.example.com:3128",
    "HTTP_PROXY": "http://proxy.example.com:3128",
    "NO_PROXY": "127.0.0.1,localhost,.example.com",
}
GALAXY_TASK_ENV = {
    "ANSIBLE_FORCE_COLOR": "false",
    "GIT_SSH_COMMAND": "ssh -o StrictHostKeyChecking=no"
}
```

Create a ConfigMap with above `extra_settings.py` file.

```bash
kubectl -n awx create configmap awx-extra-settings-configmap --from-file=extra_settings.py
```

Then, mount the ConfigMap to the task and web pods.

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...

  extra_volumes: |
    - name: awx-extra-settings
      configMap:
        name: awx-extra-settings-configmap
        items:
          - key: extra_settings.py
            path: extra_settings.py

  task_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py

  web_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py
```

### Refer sensitive values from Secrets from the settings files

If you want to embed the values from Secrets into the settings files, you can use `os.getenv()` in `extra_settings.py` as well.

```python
import os

...
SOCIAL_AUTH_AZUREAD_OAUTH2_KEY = os.getenv("AZUREAD_OAUTH2_KEY")
SOCIAL_AUTH_AZUREAD_OAUTH2_SECRET = os.getenv("AZUREAD_OAUTH2_SECRET")
SOCIAL_AUTH_SAML_ENABLED_IDPS = {
    "saml_ms_adfs": {
        "x509cert": os.getenv("SAML_IDP_SIGNING_CERT"),
        ...
    }
}
```

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...

  task_extra_env: |
    - name: AZUREAD_OAUTH2_KEY
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: key
    - name: AZUREAD_OAUTH2_SECRET
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: secret
    - name: SAML_IDP_SIGNING_CERT
      valueFrom:
        secretKeyRef:
          name: awx-saml-idp-signing-cert
          key: x509cert

  web_extra_env: |
    - name: AZUREAD_OAUTH2_KEY
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: key
    - name: AZUREAD_OAUTH2_SECRET
      valueFrom:
        secretKeyRef:
          name: awx-azuread-oauth2-secret
          key: secret
    - name: SAML_IDP_SIGNING_CERT
      valueFrom:
        secretKeyRef:
          name: awx-saml-idp-signing-cert
          key: x509cert

  extra_volumes: |
    - name: awx-extra-settings
      configMap:
        name: awx-extra-settings-configmap
        items:
          - key: extra_settings.py
            path: extra_settings.py

  task_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py

  web_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py
```

### Embed sensitive values directly into the settings files

Alternatively, if you want to embed sensitive values into the settings files directly, you can store `extra_settings.py` as a Secret instead of a ConfigMap.

```python
import os

...
SOCIAL_AUTH_AZUREAD_OAUTH2_KEY = "****************"
SOCIAL_AUTH_AZUREAD_OAUTH2_SECRET = "****************"
SOCIAL_AUTH_SAML_ENABLED_IDPS = {
    "saml_ms_adfs": {
        "x509cert": "MIIDZTCC........6xGPxz26",
        ...
    }
}
```

```bash
kubectl -n awx create secret generic awx-extra-settings-secret --from-file=extra_settings.py
```

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...

  extra_volumes: |
    - name: awx-extra-settings
      secret:
        secretName: awx-extra-settings-secret
        items:
          - key: extra_settings.py
            path: extra_settings.py

  task_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py

  web_extra_volume_mounts: |
    - name: awx-extra-settings
      mountPath: /etc/tower/conf.d/extra_settings.py
      subPath: extra_settings.py
```
