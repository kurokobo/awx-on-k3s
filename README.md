<!-- omit in toc -->
# 📚 AWX on Single Node K3s

An example implementation of AWX on single node K3s using AWX Operator, with easy-to-use simplified configuration with ownership of data and passwords.

- Accessible over HTTPS from remote host
- All data will be stored under `/data`
- Fixed (configurable) passwords for AWX and PostgreSQL
- Fixed (configurable) versions of AWX

**If you want to view the guide for the specific version of AWX Operator, switch the page to the desired tag instead of the `main` branch.**

<!-- omit in toc -->
## 📝 Table of Contents

- [📝 Environment](#-environment)
- [📝 References](#-references)
- [📝 Requirements](#-requirements)
- [📝 Deployment Instruction](#-deployment-instruction)
  - [✅ Prepare CentOS Stream 9 host](#-prepare-centos-stream-9-host)
  - [✅ Install K3s](#-install-k3s)
  - [✅ Install AWX Operator](#-install-awx-operator)
  - [✅ Prepare required files to deploy AWX](#-prepare-required-files-to-deploy-awx)
  - [✅ Deploy AWX](#-deploy-awx)
- [📝 Back up and Restore AWX using AWX Operator](#-back-up-and-restore-awx-using-awx-operator)
- [📝 Additional Guides](#-additional-guides)

## 📝 Environment

- Tested on:
  - CentOS Stream 9 (Minimal)
  - K3s v1.29.6+k3s2
- Products that will be deployed:
  - AWX Operator 2.19.1
  - AWX 24.6.1
  - PostgreSQL 15

## 📝 References

- [K3s - Lightweight Kubernetes](https://docs.k3s.io/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/24.6.1/INSTALL.md) @24.6.1
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/2.19.1/README.md) @2.19.1

## 📝 Requirements

- **Computing resources**
  - **2 CPUs minimum**.
    - Both **AMD64** (x86_64) with x86-64-v2 support, and **ARM64**  (aarch64) are supported.
  - **4 GiB RAM minimum**.
  - It's recommended to add more CPUs and RAM (like 4 CPUs and 8 GiB RAM or more) to avoid performance issue and job scheduling issue.
  - The files in this repository are configured to ignore resource requirements which specified by AWX Operator by default.
- **Storage resources**
  - At least **10 GiB for `/var/lib/rancher`** and **10 GiB for `/data`** are safe for fresh install.
    - `/var/lib/rancher` will be created and consumed by K3s and related data like container images and overlayfs.
    - `/data` will be created in this guide and used to store AWX-related databases and files.
  - **Both will be grown during lifetime** and **actual consumption highly depends on your environment and your use case**, so you should to pay attention to the consumption and add more capacity if required.

## 📝 Deployment Instruction

### ✅ Prepare CentOS Stream 9 host

Disable firewalld and nm-cloud-setup if enabled. This is [recommended by K3s](https://docs.k3s.io/installation/requirements?os=rhel#operating-systems).

```bash
# Disable firewalld
sudo systemctl disable firewalld --now

# Disable nm-cloud-setup if exists and enabled
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo reboot
```

Install the required packages to deploy AWX Operator and AWX.

```bash
sudo dnf install -y git curl
```

### ✅ Install K3s

Install a specific version of K3s with `--write-kubeconfig-mode 644` to make the config file (`/etc/rancher/k3s/k3s.yaml`) readable by non-root users.

<!-- shell: k3s: install -->
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.6+k3s2 sh -s - --write-kubeconfig-mode 644
```

### ✅ Install AWX Operator

Clone this repository and change directory.

If you want to use files suitable for a specific version of AWX Operator, [refer to tags in this repository](https://github.com/kurokobo/awx-on-k3s/tags) and specify the desired tag in `git checkout`. Especially for `0.13.0` or earlier versions of AWX Operator, refer to [📝Tips: Deploy older version of AWX Operator](tips/deploy-older-operator.md).

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
git checkout 2.19.1
```

Then invoke `kubectl apply -k operator` to deploy AWX Operator.

<!-- shell: operator: deploy -->
```bash
kubectl apply -k operator
```

The AWX Operator will be deployed to the namespace `awx`.

<!-- shell: operator: get resources -->
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

### ✅ Prepare required files to deploy AWX

Generate a Self-Signed certificate. Note that an IP address can't be specified. If you want to use a certificate from a public ACME CA such as Let's Encrypt or ZeroSSL instead of a Self-Signed certificate, follow the guide on [📁 **Use SSL Certificate from Public ACME CA**](acme) first and come back to this step when done.

<!-- shell: instance: generate certificates -->
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
  ingress_hosts:
    - hostname: awx.example.com   👈👈👈
      tls_secret: awx-secret-tls
...
```

Modify the two `password` entries in `base/kustomization.yaml`. Note that the `password` under `awx-postgres-configuration` should not contain single or double quotes (`'`, `"`) or backslashes (`\`) to avoid any issues during deployment, backup or restoration.

```yaml
...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=awx-postgres-15
      - port=5432
      - database=awx
      - username=awx
      - password=Ansible123!   👈👈👈
      - type=managed

  - name: awx-admin-password
    type: Opaque
    literals:
      - password=Ansible123!   👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `base/pv.yaml`. These directories will be used to store your databases and project files. Note that the size of the PVs and PVCs are specified in some of the files in this repository, but since their backends are `hostPath`, its value is just like a label and there is no actual capacity limitation.

<!-- shell: instance: create directories -->
```bash
sudo mkdir -p /data/postgres-15
sudo mkdir -p /data/projects
sudo chown 1000:0 /data/projects
```

### ✅ Deploy AWX

Deploy AWX, this takes few minutes to complete.

<!-- shell: instance: deploy -->
```bash
kubectl apply -k base
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

<!-- shell: instance: gather logs -->
```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

If the deployment completes successfully, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=90   changed=0    unreachable=0    failed=0    skipped=83   rescued=0    ignored=1
```

The required objects should now have been deployed next to AWX Operator in the `awx` namespace.

<!-- shell: instance: get resources -->
```bash
$ kubectl -n awx get awx,all,ingress,secrets
NAME                      AGE
awx.awx.ansible.com/awx   6m48s

NAME                                                  READY   STATUS      RESTARTS   AGE
pod/awx-operator-controller-manager-59b86c6fb-4zz9r   2/2     Running     0          7m22s
pod/awx-postgres-15-0                                 1/1     Running     0          6m33s
pod/awx-web-549f7fdbc5-htpl9                          3/3     Running     0          6m5s
pod/awx-migration-24.6.1-kglht                        0/1     Completed   0          4m36s
pod/awx-task-7d4fcdd449-mqkp2                         4/4     Running     0          6m4s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.58.194    <none>        8443/TCP   7m33s
service/awx-postgres-15                                   ClusterIP   None            <none>        5432/TCP   6m33s
service/awx-service                                       ClusterIP   10.43.180.226   <none>        80/TCP     6m7s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           7m33s
deployment.apps/awx-web                           1/1     1            1           6m5s
deployment.apps/awx-task                          1/1     1            1           6m4s

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-59b86c6fb   1         1         1       7m22s
replicaset.apps/awx-web-549f7fdbc5                          1         1         1       6m5s
replicaset.apps/awx-task-7d4fcdd449                         1         1         1       6m4s

NAME                               READY   AGE
statefulset.apps/awx-postgres-15   1/1     6m33s

NAME                             COMPLETIONS   DURATION   AGE
job.batch/awx-migration-24.6.1   1/1           2m4s       4m36s

NAME                                    CLASS     HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/awx-ingress   traefik   awx.example.com   192.168.0.221   80, 443   6m6s

NAME                                  TYPE                DATA   AGE
secret/redhat-operators-pull-secret   Opaque              1      7m33s
secret/awx-admin-password             Opaque              1      6m48s
secret/awx-postgres-configuration     Opaque              6      6m48s
secret/awx-secret-tls                 kubernetes.io/tls   2      6m48s
secret/awx-app-credentials            Opaque              3      6m9s
secret/awx-secret-key                 Opaque              1      6m41s
secret/awx-broadcast-websocket        Opaque              1      6m38s
secret/awx-receptor-ca                kubernetes.io/tls   2      6m14s
secret/awx-receptor-work-signing      Opaque              2      6m12s
```

Now your AWX is available at `https://awx.example.com/` or the hostname you specified.

Note that you have to access via the hostname that you specified in `base/awx.yaml`, instead of by IP address, since this guide uses Ingress. So you should configure your DNS or `hosts` file on your client where the browser is running.

At this point, AWX can be accessed via HTTP as well as HTTPS. If you want to force users to use HTTPS, see [📝Tips: Enable HTTP Strict Transport Security (HSTS)](tips/enable-hsts.md).

## 📝 Back up and Restore AWX using AWX Operator

The AWX Operator `0.10.0` or later has the ability to back up and restore AWX in easy way.

Refer [📁 **Back up AWX using AWX Operator**](backup) and [📁 **Restore AWX using AWX Operator**](restore) for details.

## 📝 Additional Guides

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
- [📁 **Integrate AWX with EDA Controller** (Experimental)](rulebooks)
  - The guide to deploy and use Event Driven Ansible Controller (EDA Controller) with AWX on K3s.
  - **Note that EDA Controller Operator that used in this guide is not a fully supported installation method for EDA Controller.**
- [📁 **Deploy Private Galaxy NG on Kubernetes** (Experimental)](galaxy)
  - The guide to deploy our own Galaxy NG instance.
- [📁 **Use SSL Certificate from Public ACME CA**](acme)
  - The guide to use a certificate from public ACME CA such as Let's Encrypt or ZeroSSL instead of Self-Signed certificate.
- [📁 **Use Ansible Runner**](runner)
  - The guide to use Ansible Runner to run playbook using Execution Environment.
- [📁 **Use Customized Pod Specification for your Execution Environment**](containergroup)
  - The guide to use customized Pod of the Execution Environment using **Container Group**.
- [📁 **Enable Continuous Delivery (CD) for AWX using Argo CD**](argocd)
  - The guide to deploy Argo CD and use it to enable continuous delivery for AWX.
- [📁 **Tips**](tips)
  - [📝Create "Manual" type project](tips/manual-project.md)
  - [📝Deploy AWX using external PostgreSQL database](tips/external-db.md)
  - [📝Trust custom Certificate Authority](tips/trust-custom-ca.md)
  - [📝Expose `/etc/hosts` to Pods on K3s](tips/expose-hosts.md)
  - [📝Enable HTTP Strict Transport Security (HSTS)](tips/enable-hsts.md)
  - [📝Pass values from Secrets to `extra_settings`](tips/extra-settings.md)
  - [📝Use HTTP proxy](tips/use-http-proxy.md)
  - [📝Uninstall deployed resources](tips/uninstall.md)
  - [📝Deploy older version of AWX Operator](tips/deploy-older-operator.md)
  - [📝Upgrade AWX Operator and AWX](tips/upgrade-operator.md)
  - [📝Workaround for the rate limit on Docker Hub](tips/dockerhub-rate-limit.md)
  - [📝Version Mapping for AWX Operator and AWX](tips/version-mapping.md)
  - [📝Use Kerberos authentication to connect to Windows hosts](tips/use-kerberos.md)
  - [📝Use Helm or Operator Lifecycle Manager to manage AWX Operator and AWX](tips/alternative-methods.md)
  - [📝Troubleshooting Guide](tips/troubleshooting.md)
