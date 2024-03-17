<!-- omit in toc -->
# Trust custom Certificate Authority

If your AWX has to trust custom Certificate Authority, you can pass the CA certificates to AWX. This is helpful in cases:

- Use private Git repository via SSL, without ignoring SSL verification.
- Use LDAPS to authenticate users.

Refer to [the official documentation](https://github.com/ansible/awx-operator#trusting-a-custom-certificate-authority) for more information.

Note that if you want to trust custom CA for jobs (running playbooks or accessing inventory sources), a different approach is required. Refer to [Appendix: Trust custom CA for jobs](#appendix-trust-custom-ca-for-jobs).

<!-- omit in toc -->
## Table of Contents

- [Overview](#overview)
- [Prepare required CA certificates](#prepare-required-ca-certificates)
- [Modify `base/kustomization.yaml`](#modify-basekustomizationyaml)
- [Modify `base/awx.yaml`](#modify-baseawxyaml)
- [Apply configuration](#apply-configuration)
- [Troubleshooting](#troubleshooting)
- [Appendix: Trust custom CA for jobs](#appendix-trust-custom-ca-for-jobs)
  - [Method 1: Use Container Group](#method-1-use-container-group)
    - [Create Secret](#create-secret)
    - [Create Container Group on AWX](#create-container-group-on-awx)
    - [Specify Container Group](#specify-container-group)
  - [Method 2: Mount host filesystem](#method-2-mount-host-filesystem)
    - [Update certificate store](#update-certificate-store)
    - [Modify options on AWX](#modify-options-on-awx)
    - [Add environment variables to jobs](#add-environment-variables-to-jobs)
  - [Method 3: Add certificate file to EE image](#method-3-add-certificate-file-to-ee-image)
    - [Add files to the existing image](#add-files-to-the-existing-image)
    - [Build new EE using Ansible Builder](#build-new-ee-using-ansible-builder)

## Overview

Trusting custom Certificate Authority can be achieved by following steps:

1. Creating new Secret which includes your certificates
2. Passing it to your AWX by specifying the name of the Secret as your AWX's specification

There are two kinds of certificate, one is used to trust LDAP server, and the other is used as the CA bundle.

| Fields in the specification for AWX | Keys in Secret | Containers that the certificate will be mounted | Paths that the certificate will be mounted as |
|-|-|-|-|
| `ldap_cacert_secret` | `ldap-ca.crt` | `awx-web` | `/etc/openldap/certs/ldap-ca.crt` |
| `bundle_cacert_secret` | `bundle-ca.crt` | `awx-web`, `awx-task`, and `awx-ee` | `/etc/pki/ca-trust/source/anchors/bundle-ca.crt` |

Note that the `awx-ee` container is used to run management jobs only, this is not EE which runs your playbooks. If the EE running your playbook needs certificates, refer [Appendix: Trust custom CA for jobs](#appendix-trust-custom-ca-for-jobs) on the bottom.

## Prepare required CA certificates

Place your certificates under `base` directory.

```bash
$ ls -l base
total 32
-rw-rw-r--. 1 kuro kuro  801 Feb 27 00:23 awx.yaml
-rw-rw-r--. 1 kuro kuro 1339 Feb 27 00:44 cacert.pem   👈👈👈
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
  - name: awx-custom-certs                              👈👈👈
    type: Opaque                                        👈👈👈
    files:                                              👈👈👈
      - ldap-ca.crt=<Name Of Your Certificate File>     👈👈👈
      - bundle-ca.crt=<Name Of Your Certificate File>   👈👈👈
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
  bundle_cacert_secret: awx-custom-certs   👈👈👈
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
bash-5.1$ openssl x509 -in /etc/pki/ca-trust/source/anchors/bundle-ca.crt -text
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
bash-5.1$ echo | openssl s_client -connect ldap.example.com:636 -no-CAfile -CAfile /etc/openldap/certs/ldap-ca.crt
CONNECTED(00000003)
depth=2 C = JP, ST = Example State, O = EXAMPLE.COM, CN = rca.example.com
verify return:1
depth=1 C = JP, ST = Example State, O = EXAMPLE.COM, CN = ica.example.com
verify return:1
depth=0 C = JP, ST = Example State, O = EXAMPLE.COM, CN = ldap.example.com
verify return:1
---
Certificate chain                👈👈👈 Ensure that the full certificate chain is recognized
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
Verification: OK                 👈👈👈 Ensure there is no verification error
---
...
SSL-Session:
    ...
    Verify return code: 0 (ok)   👈👈👈 Ensure there is no verification error
    ...
```

## Appendix: Trust custom CA for jobs

The `bundle_cacert_secret` stated above does not add any certificates to automation job pods, so it can't be used to trust custom CA to run jobs e.g. running playbooks, accessing inventory sources.

To trust custom CA for your jobs, there are several solutions depend on your modules or plugins that used in your jobs.

- Update system certificate store with custom CA certificate on automation job pods.
- Place custom CA certificates on automation job pods and pass its path through environment variable or parameters for modules or plugins.

This guide provides some ways as examples to achieve above solutions by invoking `update-ca-trust` and adding environment variable `REQUESTS_CA_BUNDLE` for `requests` python module (since `requests` does not refer system certificate store, custom CA certificates should be passed through this environment variable).

<!-- no toc -->
- [Method 1: Use Container Group](#method-1-use-container-group)
  - Add required certificates and environment variable on the automation job pods by mounting it using [Container Group](../containergroup) with custom pod specification.
- [Method 2: Mount host filesystem](#method-2-mount-host-filesystem)
  - Add required certificates and environment variable on the automation job pods by mounting host filesystem.
- [Method 3: Add certificate file to EE image](#method-3-add-certificate-file-to-ee-image)
  - Add required certificates and environment variable on the automation job pods by adding it to the EE image.

In this example, the value of the environment variable `REQUESTS_CA_BUNDLE` is specified as `/etc/pki/tls/cert.pem`. This file is a symbolic link to `/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem` that updated by `update-ca-trust` by combining and includes all CA certificates in the system certificate store into one file.

### Method 1: Use Container Group

This method can be used for standard jobs, such as the case that some modules in your playbook require custom CA certificates. In addition, this method can also be used to access inventory sources.

#### Create Secret

Prepare CA certificate file with PEM format (refer to [Prepare required CA certificates](#prepare-required-ca-certificates)) first, then create Secret with the certificate.

```bash
kubectl -n awx create secret generic awx-custom-job-certs \
    --from-file=job-ca.crt=cacert.pem
```

#### Create Container Group on AWX

Create new Container Group by `Administration` > `Instance Group` > `Add` > `Add container group` on AWX, then enable `Customize pod specification` and put following specification.

This specification supports both jobs that refers system certificate store and the store for `requests`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: awx
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  initContainers:
    - name: init
      image: quay.io/ansible/awx-ee:latest
      command:
        - /bin/sh
        - -c
        - |
          mkdir -p /etc/pki/ca-trust/extracted/{java,pem,openssl,edk2}
          update-ca-trust
      volumeMounts:
        - name: ca-trust-extracted
          mountPath: /etc/pki/ca-trust/extracted
        - name: awx-custom-job-certs
          mountPath: /etc/pki/ca-trust/source/anchors/job-ca.crt
          subPath: job-ca.crt
          readOnly: true
  containers:
    - image: quay.io/ansible/awx-ee:latest
      name: worker
      args:
        - ansible-runner
        - worker
        - '--private-data-dir=/runner'
      env:
        - name: REQUESTS_CA_BUNDLE
          value: /etc/pki/tls/cert.pem
      resources:
        requests:
          cpu: 250m
          memory: 100Mi
      volumeMounts:
        - name: ca-trust-extracted
          mountPath: /etc/pki/ca-trust/extracted
        - name: awx-custom-job-certs
          mountPath: /etc/pki/ca-trust/source/anchors/job-ca.crt
          subPath: job-ca.crt
          readOnly: true
  volumes:
    - name: ca-trust-extracted
      emptyDir: {}
    - name: awx-custom-job-certs
      secret:
        secretName: awx-custom-job-certs
        items:
          - key: job-ca.crt
            path: job-ca.crt
```

This specification has following changes from the default one:

- Init container (`initContainers`) is added to invoke `update-ca-trust` to update certificate store using CA certificate that mounted through Secret resource.
- Worker container mounts certificate store that updated by init container (`volumeMounts`).
- Worker container has environment variable `REQUESTS_CA_BUNDLE` (`env`) that has path for updated certificate file.
- Volumes (`volumes`) are added to achieve above changes.

If you just want to place CA certificates somewhere on automation job pods and `update-ca-trust` is not required for you, you can ignore `initContainers` and `ca-trust-extracted` (under `volumes` and `volumeMounts`), and change `mountPath` for `awx-custom-job-certs`.

#### Specify Container Group

The Container Group can be specified as `Instance Groups` for the Job Template or Organization.

If you want to use this Container Group to access inventory sources, specify it as `Instance Groups` for the Inventory or Organization.

### Method 2: Mount host filesystem

This method can be used for both standard jobs and inventory sync.

Note that this method changes the global configuration of your K3s host and AWX, so whole system and other applications on your K3s host, and all automation job pods running on the AWX are also affected.

#### Update certificate store

Prepare CA certificate file with PEM format (refer to [Prepare required CA certificates](#prepare-required-ca-certificates)) first, then place the certificate file under `/etc/pki/ca-trust/source/anchors` on your K3s host, and invoke `update-ca-trust`.

```bash
$ ls -l /etc/pki/ca-trust/source/anchors
total 4
-rw-r--r--. 1 root root 1489 Apr 27 22:49 cacert.pem

$ sudo update-ca-trust
```

If your Kubernetes cluster has multiple nodes, this step have to be invoked on all nodes where your automation job pods will be launched on.

If you just want to place CA certificates somewhere on automation job pods and `update-ca-trust` is not required for you, you can ignore `update-ca-trust`.

#### Modify options on AWX

On `Settings` > `Jobs settings` > `Edit` page, modify following two options.

- Enable `Expose host paths for Container Groups`.
- Ensure `"/etc/pki/ca-trust:/etc/pki/ca-trust:O"` is specified in `Paths to expose to isolated jobs`.

#### Add environment variables to jobs

To add environment variables to your jobs, on `Settings` > `Jobs settings` > `Edit` page, modify `Extra Environment Variables` with following value. If the environment variable is not required for you, this step can be ignored.

```json
{
  "REQUESTS_CA_BUNDLE": "/etc/pki/tls/cert.pem"
}
```

### Method 3: Add certificate file to EE image

This method can be used for both standard jobs and inventory sync, and can be achieved by following two ways.

- [Add files to the existing image](#add-files-to-the-existing-image)
- [Build new EE using Ansible Builder](#build-new-ee-using-ansible-builder)

#### Add files to the existing image

Prepare CA certificate file with PEM format (refer to [Prepare required CA certificates](#prepare-required-ca-certificates)), then you can add the certificate to existing image using following Dockerfile.

This example shows the way to add local certificate file (`cacert.pem`) and environment variable to `quay.io/ansible/awx-ee:latest`.

```dockerfile
FROM quay.io/ansible/awx-ee:latest

USER root
COPY cacert.pem /etc/pki/ca-trust/source/anchors/cacert.pem
RUN update-ca-trust
ENV REQUESTS_CA_BUNDLE=/etc/pki/tls/cert.pem

USER 1000
```

Build and push image to container registry, then define new EE on AWX, and specify it for your jobs.

#### Build new EE using Ansible Builder

Prepare CA certificate file with PEM format (refer to [Prepare required CA certificates](#prepare-required-ca-certificates)), then you can add `additional_build_files` and `additional_build_steps.append_final` to your `execution-environment.yml` to add the certificate and environment variable to your EE.

```yaml
...
additional_build_files:
  - src: certs/cacert.pem
    dest: certs

additional_build_steps:
  append_final:
    - COPY _build/certs/cacert.pem /etc/pki/ca-trust/source/anchors/cacert.pem
    - RUN update-ca-trust
    - ENV REQUESTS_CA_BUNDLE=/etc/pki/tls/cert.pem
```

Build and push image to container registry, then define new EE on AWX, and specify it for your jobs.
