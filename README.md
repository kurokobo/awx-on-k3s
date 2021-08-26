<!-- omit in toc -->
# AWX on Single Node K3s

An example implementation of AWX on single node K3s using AWX Operator, with easy-to-use simplified configuration with ownership of data and passwords.

- Accesible over HTTPS from remote host
- All data will be stored under `/data`
- Fixed (configurable) passwords for AWX and PostgreSQL
- Fixed (configurable) versions of AWX and PostgreSQL

<!-- omit in toc -->
## Table of Contents

- [Environment](#environment)
- [References](#references)
- [Procedure](#procedure)
  - [Prepare CentOS 8 host](#prepare-centos-8-host)
  - [Install K3s](#install-k3s)
  - [Install AWX Operator](#install-awx-operator)
  - [Prepare required files](#prepare-required-files)
  - [Deploy AWX](#deploy-awx)
- [Backing up and Restoring using AWX Operator](#backing-up-and-restoring-using-awx-operator)
  - [Backing up using AWX Operator](#backing-up-using-awx-operator)
    - [Prepare for Backup](#prepare-for-backup)
    - [Invoke Manual Backup](#invoke-manual-backup)
  - [Restoring using AWX Operator](#restoring-using-awx-operator)
    - [Prepare for Restore](#prepare-for-restore)
    - [Invoke Manual Restore](#invoke-manual-restore)
- [Additional Guides](#additional-guides)

## Environment

- Tested on:
  - CentOS 8 (Minimal)
- Products that will be deployed:
  - AWX Operator 0.13.0
  - AWX 19.3.0
  - PostgreSQL 12

## References

- [K3s - Lightweight Kubernetes](https://rancher.com/docs/k3s/latest/en/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/19.3.0/INSTALL.md) @19.3.0
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/0.13.0/README.md) @0.13.0

## Procedure

### Prepare CentOS 8 host

Disable Firewalld. This is [recommended by K3s](https://rancher.com/docs/k3s/latest/en/advanced/#additional-preparation-for-red-hat-centos-enterprise-linux).

```bash
sudo systemctl disable firewalld --now
```

### Install K3s

Install K3s with `--write-kubeconfig-mode 644` to make config file (`/etc/rancher/k3s/k3s.yaml`) readable by non-root user.

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

### Install AWX Operator

Install specified version of AWX Operator.

```bash
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/0.13.0/deploy/awx-operator.yaml
```

### Prepare required files

Clone this repository and change directory.

```bash
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
```

Generate a Self-Signed Certificate. Note that IP address can't be specified.

```bash
AWX_HOST="awx.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./base/tls.crt -keyout ./base/tls.key -subj "/CN=${AWX_HOST}/O=${AWX_HOST}" -addext "subjectAltName = DNS:${AWX_HOST}"
```

Modify `hostname` in `base/awx.yaml`.

```yaml
...
spec:
  ingress_type: ingress
  ingress_tls_secret: awx-secret-tls
  hostname: awx.example.com     👈👈👈
...
```

Modify two `password`s in `base/kustomization.yaml`.

```yaml
...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=awx-postgres
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

Prepare directories for Persistent Volumes defined in `base/pv.yaml`.

```bash
sudo mkdir -p /data/postgres
sudo mkdir -p /data/projects
sudo chown 1000:0 /data/projects
```

### Deploy AWX

Deploy AWX, this takes few minutes to complete.

```bash
kubectl apply -k base
```

Once this completed, the logs of `deployment/awx-operator` end with:

```txt
$ kubectl logs -f deployment/awx-operator
...
--------------------------- Ansible Task Status Event StdOut  -----------------
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0 
-------------------------------------------------------------------------------
```

Required objects has been deployed in `awx` namespace.

```bash
$ kubectl -n awx get awx,all,ingress,secrets
NAME                      AGE
awx.awx.ansible.com/awx   4m19s

NAME                      READY   STATUS    RESTARTS   AGE
pod/awx-postgres-0        1/1     Running   0          4m27s
pod/awx-59ff55b5b-qdk9p   4/4     Running   0          4m19s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-postgres   ClusterIP   None            <none>        5432/TCP   4m27s
service/awx-service    ClusterIP   10.43.209.222   <none>        80/TCP     4m21s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx   1/1     1            1           4m19s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-59ff55b5b   1         1         1       4m19s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     7m27s

NAME                                    CLASS    HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/awx-ingress   <none>   awx.example.com   192.168.0.100   80, 443   4m20s

NAME                                TYPE                                  DATA   AGE
secret/default-token-lxj9h          kubernetes.io/service-account-token   3      5m36s
secret/awx-admin-password           Opaque                                1      4m45s
secret/awx-broadcast-websocket      Opaque                                1      4m45s
secret/awx-secret-tls               kubernetes.io/tls                     2      4m45s
secret/awx-postgres-configuration   Opaque                                6      4m45s
secret/awx-secret-key               Opaque                                1      4m45s
secret/awx-app-credentials          Opaque                                3      4m23s
secret/awx-token-6s7rj              kubernetes.io/service-account-token   3      4m22s
```

Now AWX is available at `https://<awx-host>/`.

## Backing up and Restoring using AWX Operator

The AWX Operator `0.10.0` or later has the ability to backup and restore AWX in easy way.

### Backing up using AWX Operator

#### Prepare for Backup

Prepare directories for Persistent Volumes to store backup files that defined in `backup/pv.yaml`.

```bash
sudo mkdir -p /data/backup
```

Then deploy Persistent Volume and Persistent Volume Claim.

```bash
kubectl apply -k backup
```

#### Invoke Manual Backup

Modify the name of the AWXBackup object in `backup/awxbackup.yaml`.

```yaml
...
kind: AWXBackup
metadata:
  name: awxbackup-2021-06-06     👈👈👈
  namespace: awx
...
```

Then invoke backup by applying this manifest file.

```bash
kubectl apply -f backup/awxbackup.yaml
```

Once this completed, the logs of `deployment/awx-operator` end with:

```txt
$ kubectl logs -f deployment/awx-operator
--------------------------- Ansible Task Status Event StdOut  -----------------
PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0
-------------------------------------------------------------------------------
```

This will create AWXBackup object in the namespace and also create backup files in the Persistent Volume. In this example those files are available at `/data/backup`.

```bash
$ kubectl -n awx get awxbackup
NAME                   AGE
awxbackup-2021-06-06   6m47s
```

```bash
$ ls -l /data/backup/
total 0
drwxr-xr-x. 2 root root 59 Jun  5 06:51 tower-openshift-backup-2021-06-06-10:51:49

$ ls -l /data/backup/tower-openshift-backup-2021-06-06-10\:51\:49/
total 736
-rw-r--r--. 1 root             root    749 Jun  6 06:51 awx_object
-rw-r--r--. 1 root             root    482 Jun  6 06:51 secrets.yml
-rw-------. 1 systemd-coredump root 745302 Jun  6 06:51 tower.db
```

### Restoring using AWX Operator

#### Prepare for Restore

If your PV, PVC, and Secret still exist, no preparation is required.

If you are restoring the entire AWX to a new environment, create the PVs and PVCs first to be restored.

```bash
sudo mkdir -p /data/postgres
sudo mkdir -p /data/projects
sudo chown 1000:0 /data/projects
```

Then deploy Persistent Volume and Persistent Volume Claim.

```bash
kubectl apply -k restore
```

#### Invoke Manual Restore

Modify the name of the AWXRestore object in `restore/awxrestore.yaml`.

```yaml
...
kind: AWXRestore
metadata:
  name: awxrestore-2021-06-06     👈👈👈
  namespace: awx
...
```

If you want to restore from AWXBackup object, specify its name in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from AWXBackup object
  backup_pvc_namespace: awx
  backup_name: awxbackup-2021-06-06     👈👈👈
...
```

If the AWXBackup object no longer exists, place the backup files and specify the name of the PVC and directory in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from existing files on PVC (without AWXBackup object)
  backup_pvc_namespace: awx
  backup_pvc: awx-backup-claim     👈👈👈
  backup_dir: /backups/tower-openshift-backup-2021-06-06-10:51:49     👈👈👈
...
```

Then invoke restore by applying this manifest file.

```bash
kubectl apply -f restore/awxrestore.yaml
```

Once this completed, the logs of `deployment/awx-operator` end with:

```txt
$ kubectl logs -f deployment/awx-operator
--------------------------- Ansible Task Status Event StdOut  -----------------
PLAY RECAP *********************************************************************
localhost                  : ok=56   changed=0    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
-------------------------------------------------------------------------------
```

This will create AWXRestore object in the namespace, and now your AWX is restored.

```bash
$ kubectl -n awx get awxrestore
NAME                    AGE
awxrestore-2021-06-06   137m
```

## Additional Guides

- [📁 **Deploy Private Git Repository on Kubernetes**](git)
  - To use AWX with SCM, this repository includes the manifests to deploy [Gitea](https://gitea.io/en-us/).
  - See [📝`git/README.md`](git) for instructions.
- [📁 **Deploy Private Container Registry on Kubernetes**](registry)
  - To use Execution Environments in AWX (AWX-EE), we have to push the container image built with `ansible-builder` to the container registry.
  - If we don't want to push our container images to Docker Hub or other cloud services, we can deploy a private container registry on K3s.
  - See [📝`registry/README.md`](registry) for instructions.
- [📁 **Use Ansible Builder**](builder)
  - Use Ansible Builder to build our own Execution Environment.
  - See [📝`builder/README.md`](builder) for instructions.
- [📁 **Use Ansible Runner**](runner)
  - Use Ansible Runner to run playbook using Execution Environment.
  - See [📝`runner/README.md`](runner) for instructions.
- [📁 **Use Customized Pod Specification for your Execution Environment**](containergroup)
  - We can customize the specification of the Pod of the Execution Environment using **Container Group**.
  - See [📝`containergroup/README.md`](containergroup) for instructions.
- [📁 **Deploy Private Galaxy NG on Docker or Kubernetes** (Experimental)](galaxy)
  - Deploy our own Galaxy NG instance.
  - **Note that the containerized implementation of Galaxy NG is not supported at this time.**
  - **All information on the page is for development, testing and study purposes only.**
  - See [📝`galaxy/README.md`](galaxy) for instructions.
- [📁 **Tips**](tips)
  - [📝Expose `/etc/hosts` to Pods on K3s](tips/expose-hosts.md)
