<!-- omit in toc -->
# Trust custom Certificate Authority

If your AWX has to trust custom Certificate Authority, you can pass the CA certificates to AWX. This is helpful in cases:

- Use private Git repository via SSL, without ignoring SSL verification.
- Use LDAPS to authenticate users.

Refer [the official documentation](https://github.com/ansible/awx-operator#trusting-a-custom-certificate-authority) for more information.

<!-- omit in toc -->
## Table of Contents

- [Prepare required CA certificatess](#prepare-required-ca-certificatess)
- [Modify `base/kustomization.yaml`](#modify-basekustomizationyaml)
- [Modify `base/awx.yaml`](#modify-baseawxyaml)
- [Apply configuration](#apply-configuration)

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
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```
