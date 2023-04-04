<!-- omit in toc -->
# Trust custom Certificate Authority

If your AWX has to trust custom Certificate Authority, you can pass the CA certificates to AWX. This is helpful in cases:

- Use private Git repository via SSL, without ignoring SSL verification.
- Use LDAPS to authenticate users.

Refer [the official documentation](https://github.com/ansible/awx-operator#trusting-a-custom-certificate-authority) for more information.

<!-- omit in toc -->
## Table of Contents

- [Overview](#overview)
- [Prepare required CA certificatess](#prepare-required-ca-certificatess)
- [Modify `base/kustomization.yaml`](#modify-basekustomizationyaml)
- [Modify `base/awx.yaml`](#modify-baseawxyaml)
- [Apply configuration](#apply-configuration)
- [Troubleshooting](#troubleshooting)

## Overview

Trusting custom Certificate Authority can be achieved by following steps:

1. Creating new Secret which includes your certificates
2. Passing it to your AWX by specifying the name of the Secret as your AWX's specification

There are two kinds of certificate, one is used to trust LDAP server, and the other is used as the CA bundle.

| Fields in the specification for AWX | Keys in Secret | Containers that the certificate will be mounted | Paths that the certificate will be mounted as |
|-|-|-|-|
| `ldap_cacert_secret` | `ldap-ca.crt` | `awx-web` | `/etc/openldap/certs/ldap-ca.crt` |
| `bundle_cacert_secret` | `bundle-ca.crt` | `awx-web`, `awx-task`, and `awx-ee` | `/etc/pki/ca-trust/source/anchors/bundle-ca.crt` |

Note that the `awx-ee` container is used to run management jobs only, not EE which runs your playbooks. If the EE running your playbook needs a certificate, you will need to [customize the pod specification](../containergroup).

If you want to mount the certificate to the additional containers in AWX pod or the additional path other than above, you shoud add extra volumes and extra mounts using `extra_volumes` and `_extra_volume_mounts` field, but this is not covered in this guide. Refer to [the official documentation](https://github.com/ansible/awx-operator#custom-volume-and-volume-mount-options).

## Prepare required CA certificatess

Place your certificates under `base` directory.

```bash
$ ls -l base
total 32
-rw-rw-r--. 1 kuro kuro  801 Feb 27 00:23 awx.yaml
-rw-rw-r--. 1 kuro kuro 1339 Feb 27 00:44 cacert.pem     👈👈👈
-rw-rw-r--. 1 kuro kuro  610 Feb 27 00:23 kustomization.yaml
...
```

Note that **your certificates have to have PEM format**. You can check the format of the certificates depending on which of the following commands succeeds.

```bash
# Works for PEM format
openssl x509 -in cacert.crt -text

# Works for DER format
openssl x509 -in cacert.crt -inform DER -text

# Works for PKCS #7 format
openssl pkcs7 -in cacert.crt -text

# Works for PKCS #12 format
openssl pkcs12 -in cacert.crt -info
```

If your certificate doesn't have PEM format, you can convert it by followings:

```bash
# Convert DER to PEM
openssl x509 -in cacert.crt -inform DER -out cacert.pem -outform PEM

# Convert PKCS #7 to PEM
openssl pkcs7 -print_certs -in cacert.crt -out cacert.pem -outform PEM

# Convert PKCS #12 to PEM
openssl pkcs12 -in cacert.crt -out cacert.pem -nokeys -nodes
```

## Modify `base/kustomization.yaml`

Add following lines under `secretGenerator` in `base/kustomization.yaml`.

Note that this example provides both `ldap-ca.crt` and `bundle-ca.crt`, but you can remove unnecessary line if you don't need both of them. `ldap-ca.crt` will be used as the CA certificate for LDAP server, and `bundle-ca.crt` will be used as the CA bundle.

```yaml
...
secretGenerator:
  ...
  - name: awx-custom-certs     👈👈👈
    type: Opaque     👈👈👈
    files:     👈👈👈
      - ldap-ca.crt=<Name Of Your Certificate File>     👈👈👈
      - bundle-ca.crt=<Name Of Your Certificate File>     👈👈👈
  ...
```

## Modify `base/awx.yaml`

Add following lines under `secretGenerator` in `base/kustomization.yaml`.

Note that this example provides both `ldap_cacert_secret` (should have `ldap-ca.crt`) and `bundle_cacert_secret` (should have `bundle-ca.crt`), but you can remove unnecessary line if you don't need both of them.

```yaml
...
spec:
  ...
  ldap_cacert_secret: awx-custom-certs     👈👈👈
  bundle_cacert_secret: awx-custom-certs     👈👈👈
  ...
```

## Apply configuration

Invoke `apply` command. This will start re-deployment of your AWX.

```base
kubectl apply -k base
```

You can monitor the progress of the re-deployment by following command:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

## Troubleshooting

If you have problem with SSL connection such as LDAPS, you can verify your certificates inside the pod.

```bash
# Open Bash shell of the "awx-web" container
$ kubectl -n awx exec -it deployment/awx-web -c awx-web -- bash
bash-5.1$
```

First of all, you should ensure your CA certificate is mounted and has PEM format. The certificate should be be dumped as readable plain text by following command, without any error.

```bash
# The secret ldap_cacert_secret is mounted as /etc/openldap/certs/ldap-ca.crt
bash-5.1$ openssl x509 -in /etc/openldap/certs/ldap-ca.crt -text

# The secret bundle_cacert_secret is mounted as /etc/pki/ca-trust/source/anchors/bundle-ca.crt
bash-5.1$ openssl x509 -in /etc/pki/ca-trust/source/anchors/bundle-ca.crt
```

Note that your certificate file should contain both intermediate CA and root CA, if your server certificate is signed by intermediate CA.

```bash
# Example output of concatenated CA cert; one for intermediate CA, one for root CA
bash-5.1$ cat /etc/openldap/certs/ldap-ca.crt
-----BEGIN CERTIFICATE-----
MIIDizCCAnOgAwIBAgIUftINZYmeHvcovY0qBHp+SqZWrlswDQYJKoZIhvcNAQEL
...
3Eyhv0l7mJw/86twDMFFax+cKOCRFV6NoPOpzK1mzAXmxth6vk8DeRm0ipVpQVQ=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDizCCAnOgAwIBAgIUftINZYmeHvcovY0qBHp+SqZWrlwwDQYJKoZIhvcNAQEL
...
lVsDxZfbZVpRGkDr8odNurNmz0Xcttr+ZVRkoTy5KUxqIZhQuS6ySJj7yoLawWY=
-----END CERTIFICATE-----
```

Now you can test SSL connection.

```bash
# This is an example to test connection to LDAP server over SSL using /etc/openldap/certs/ldap-ca.crt
bash-5.1$ openssl s_client -connect ldap.example.com:636 -no-CAfile -CAfile /etc/openldap/certs/ldap-ca.crt
CONNECTED(00000003)
depth=2 C = JP, ST = Example State, O = EXAMPLE.COM, CN = rca.example.com
verify return:1
depth=1 C = JP, ST = Example State, O = EXAMPLE.COM, CN = ica.example.com
verify return:1
depth=0 C = JP, ST = Example State, O = EXAMPLE.COM, CN = ldap.example.com
verify return:1
---
Certificate chain     👈👈👈 Ensure that the full certificate chain is recognized
 0 s:C = JP, ST = Example State, O = EXAMPLE.COM, CN = ldap.example.com
   i:C = JP, ST = Example State, O = EXAMPLE.COM, CN = ica.example.com
   ...
 1 s:C = JP, ST = Example State, O = EXAMPLE.COM, CN = ica.example.com
   i:C = JP, ST = Example State, O = EXAMPLE.COM, CN = rca.example.com
   ...
 2 s:C = JP, ST = Example State, O = EXAMPLE.COM, CN = rca.example.com
   i:C = JP, ST = Example State, O = EXAMPLE.COM, CN = rca.example.com
   ...
---
...
---
SSL handshake has read 3210 bytes and written 413 bytes
Verification: OK     👈👈👈 Ensure there is no verification error
---
...
SSL-Session:
    ...
    Verify return code: 0 (ok)     👈👈👈 Ensure there is no verification error
    ...
```
