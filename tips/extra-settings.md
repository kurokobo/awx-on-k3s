# Pass values from Secrets to `extra_settings`

Currently, AWX Operator does not ability to inject Secrets to `extra_settings`.

However, since Secrets are injectable for environment variables, we can indirectly inject values from Secrets into `extra_settings` by specifying values that refers to the environment variables.

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
