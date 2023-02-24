<!-- omit in toc -->
# AWX on Single Node K3s

An example implementation of AWX on single node K3s using AWX Operator, with easy-to-use simplified configuration with ownership of data and passwords.

- Accessible over HTTPS from remote host
- All data will be stored under `/data`
- Fixed (configurable) passwords for AWX and PostgreSQL
- Fixed (configurable) versions of AWX and PostgreSQL

**If you want to view the guide for the specific version of AWX Operator, switch the page to the desired tag instead of the `main` branch.**

<!-- omit in toc -->
## Table of Contents

- [Environment](#environment)
- [References](#references)
- [Requirements](#requirements)
- [Deployment Instruction](#deployment-instruction)
  - [Prepare CentOS Stream 8 host](#prepare-centos-stream-8-host)
  - [Install K3s](#install-k3s)
  - [Install AWX Operator](#install-awx-operator)
  - [Prepare required files](#prepare-required-files)
  - [Deploy AWX](#deploy-awx)
- [Back up and Restore AWX using AWX Operator](#back-up-and-restore-awx-using-awx-operator)
- [Additional Guides](#additional-guides)

## Environment

- Tested on:
  - CentOS Stream 8 (Minimal)
  - K3s v1.25.6+k3s1
- Products that will be deployed:
  - AWX Operator 1.2.0
  - AWX 21.12.0
  - PostgreSQL 13

## References

- [K3s - Lightweight Kubernetes](https://docs.k3s.io/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/21.12.0/INSTALL.md) @21.12.0
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/1.2.0/README.md) @1.2.0

## Requirements

- **Computing resources**
  - **2 CPUs and 4 GiB RAM minimum**.
  - It's recommended to add more CPUs and RAM (like 4 CPUs and 8 GiB RAM or more) to avoid performance issue and job scheduling issue.
  - The files in this repository are configured to ignore resource requirements which specified by AWX Operator by default.
- **Storage resources**
  - At least **10 GiB for `/var/lib/rancher`** and **10 GiB for `/data`** are safe for fresh install.
  - **Both will be grown during lifetime** and **actual consumption highly depends on your environment and your use case**, so you should to pay attention to the consumption and add more capacity if required.
  - `/var/lib/rancher` will be created and consumed by K3s and related data like container images and overlayfs.
  - `/data` will be created in this guide and used to store AWX-related databases and files.

## Deployment Instruction

### Prepare CentOS Stream 8 host

Disable firewalld and nm-cloud-setup if enabled. This is [recommended by K3s](https://docs.k3s.io/advanced#red-hat-enterprise-linux--centos).

```bash
# Disable firewalld
sudo systemctl disable firewalld --now

# Disable nm-cloud-setup if exists and enabled
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo reboot
```

Install required packages to deploy AWX Operator and AWX.

```bash
sudo dnf install -y git make curl
```

### Install K3s

Install K3s with `--write-kubeconfig-mode 644` to make config file (`/etc/rancher/k3s/k3s.yaml`) readable by non-root user.

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

### Install AWX Operator

Install specified version of AWX Operator. Note that this procedure is applicable only for AWX Operator `0.14.0` or later. If you want to deploy `0.13.0` or earlier version of AWX Operator, refer [📝Tips: Deploy older version of AWX Operator](tips/deploy-older-operator.md).

```bash
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout 1.2.0
```

Export the name of the namespace where you want to deploy AWX Operator as the environment variable `NAMESPACE` and run `make deploy`. The default namespace is `awx`.

```bash
export NAMESPACE=awx
make deploy
```

The AWX Operator will be deployed to the namespace you specified.

```bash
$ kubectl -n awx get all
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-operator-controller-manager-68d787cfbd-kjfg7   2/2     Running   0          16s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.150.245   <none>        8443/TCP   16s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           16s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-68d787cfbd   1         1         1       16s
```

### Prepare required files

Clone this repository and change directory.

If you want to use files suitable for the specific version of AWX Operator, [refer tags in this repository](https://github.com/kurokobo/awx-on-k3s/tags) and specify desired tag in `git checkout`.

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
git checkout 1.2.0
```

Generate a Self-Signed certificate. Note that IP address can't be specified. If you want to use a certificate from public ACME CA such as Let's Encrypt or ZeroSSL instead of Self-Signed certificate, follow the guide on [📁 **Use SSL Certificate from Public ACME CA**](acme) first and come back to this step when done.

```bash
AWX_HOST="awx.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./base/tls.crt -keyout ./base/tls.key -subj "/CN=${AWX_HOST}/O=${AWX_HOST}" -addext "subjectAltName = DNS:${AWX_HOST}"
```

Modify `hostname` in `base/awx.yaml`.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: awx-secret-tls
  hostname: awx.example.com     👈👈👈
...
```

Modify two `password`s in `base/kustomization.yaml`. Note that the `password` under `awx-postgres-configuration` should not contain single or double quotes (`'`, `"`) or backslashes (`\`) to avoid any issues during deployment, backup or restoration.

```yaml
...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=awx-postgres-13
      - port=5432
      - database=awx
      - username=awx
      - password=Ansible123!     👈👈👈
      - type=managed

  - name: awx-admin-password
    type: Opaque
    literals:
      - password=Ansible123!     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `base/pv.yaml`. These directories will be used to store your databases and project files. Note that the size of the PVs and PVCs are specified in some of the files in this repository, but since their backends are `hostPath`, its value is just like a label and there is no actual capacity limitation.

```bash
sudo mkdir -p /data/postgres-13
sudo mkdir -p /data/projects
sudo chmod 755 /data/postgres-13
sudo chown 1000:0 /data/projects
```

### Deploy AWX

Deploy AWX, this takes few minutes to complete.

```bash
kubectl apply -k base
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=77   changed=0    unreachable=0    failed=0    skipped=71   rescued=0    ignored=1
```

Required objects has been deployed next to AWX Operator in `awx` namespace.

```bash
$ kubectl -n awx get awx,all,ingress,secrets
NAME                      AGE
awx.awx.ansible.com/awx   5m

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-operator-controller-manager-5d5d58758c-7xcrl   2/2     Running   0          5m35s
pod/awx-postgres-13-0                                  1/1     Running   0          4m46s
pod/awx-5b859c644-zp6x5                                4/4     Running   0          4m26s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.229.20    <none>        8443/TCP   5m45s
service/awx-postgres-13                                   ClusterIP   None            <none>        5432/TCP   4m46s
service/awx-service                                       ClusterIP   10.43.135.205   <none>        80/TCP     4m28s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           5m45s
deployment.apps/awx                               1/1     1            1           4m26s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-5d5d58758c   1         1         1       5m35s
replicaset.apps/awx-5b859c644                                1         1         1       4m26s

NAME                               READY   AGE
statefulset.apps/awx-postgres-13   1/1     4m46s

NAME                                    CLASS    HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/awx-ingress   <none>   awx.example.com   192.168.0.219   80, 443   4m27s

NAME                                  TYPE                DATA   AGE
secret/awx-admin-password             Opaque              1      5m
secret/awx-postgres-configuration     Opaque              6      5m
secret/awx-secret-tls                 kubernetes.io/tls   2      3m54s
secret/redhat-operators-pull-secret   Opaque              1      4m30s
secret/awx-app-credentials            Opaque              3      4m30s
secret/awx-secret-key                 Opaque              1      4m55s
secret/awx-broadcast-websocket        Opaque              1      4m52s
secret/awx-receptor-ca                Opaque              2      4m26s
secret/awx-receptor-work-signing      Opaque              2      4m29s
```

Now your AWX is available at `https://awx.example.com/` or the hostname you specified.

Note that you have to access via hostname that you specified in `base/awx.yaml`, instead of IP address, since this guide uses Ingress. So you should configure your DNS or `hosts` file on your client where the browser is running.

At this point, AWX can be accessed via HTTP as well as HTTPS. If you want to redirect HTTP to HTTPS, see [📝Tips: Redirect HTTP to HTTPS](tips/https-redirection.md).

## Back up and Restore AWX using AWX Operator

The AWX Operator `0.10.0` or later has the ability to back up and restore AWX in easy way.

Refer [📁 **Back up AWX using AWX Operator**](backup) and [📁 **Restore AWX using AWX Operator**](restore) for details.

## Additional Guides

- [📁 **Back up AWX using AWX Operator**](backup)
  - The guide to make backup of your AWX using AWX Operator.
  - This guide includes not only the way to make backup manually, but also an example simple playbook for Ansible, which can be use with scheduling feature on AWX.
- [📁 **Restore AWX using AWX Operator**](restore)
  - The guide to restore your AWX using AWX Operator.
- [📁 **Build and Use your own Execution Environment**](builder)
  - The guide to use Ansible Builder to build our own Execution Environment.
- [📁 **Deploy Private Git Repository on Kubernetes**](git)
  - The guide to use AWX with SCM. This repository includes the manifests to deploy [Gitea](https://gitea.io/en-us/).
- [📁 **Deploy Private Container Registry on Kubernetes**](registry)
  - The guide to use Execution Environments in AWX (AWX-EE).
  - If we want to use our own Execution Environment built with Ansible Builder and don't want to push it to the public container registry e.g. Docker Hub, we can deploy a private container registry on K3s.
- [📁 **Deploy Private Galaxy NG on Docker or Kubernetes** (Experimental)](galaxy)
  - The guide to deploy our own Galaxy NG instance.
  - **Note that the containerized implementation of Galaxy NG is not officially supported at this time.**
  - **All information on the guide is for development, testing and study purposes only.**
- [📁 **Use SSL Certificate from Public ACME CA**](acme)
  - The guide to use a certificate from public ACME CA such as Let's Encrypt or ZeroSSL instead of Self-Signed certificate.
- [📁 **Use Ansible Runner**](runner)
  - The guide to use Ansible Runner to run playbook using Execution Environment.
- [📁 **Use Customized Pod Specification for your Execution Environment**](containergroup)
  - The guide to use customized Pod of the Execution Environment using **Container Group**.
- [📁 **Tips**](tips)
  - [📝Create "Manual" type project](tips/manual-project.md)
  - [📝Deploy AWX using external PostgreSQL database](tips/external-db.md)
  - [📝Trust custom Certificate Authority](tips/trust-custom-ca.md)
  - [📝Expose `/etc/hosts` to Pods on K3s](tips/expose-hosts.md)
  - [📝Redirect HTTP to HTTPS](tips/https-redirection.md)
  - [📝Use HTTP proxy](tips/use-http-proxy.md)
  - [📝Uninstall deployed resources](tips/uninstall.md)
  - [📝Deploy older version of AWX Operator](tips/deploy-older-operator.md)
  - [📝Upgrade AWX Operator and AWX](tips/upgrade-operator.md)
  - [📝Workaround for the rate limit on Docker Hub](tips/dockerhub-rate-limit.md)
  - [📝Version Mapping between AWX Operator and AWX](tips/version-mapping.md)
  - [📝Use Kerberos authentication to connect to Windows hosts](tips/use-kerberos.md)
  - [📝Use Helm or Operator Lifecycle Manager to manage AWX Operator and AWX](tips/alternative-methods.md)
  - [📝Troubleshooting Guide](tips/troubleshooting.md)
