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
  - AWX Operator 0.15.0
  - AWX 19.5.0
  - PostgreSQL 12

## References

- [K3s - Lightweight Kubernetes](https://rancher.com/docs/k3s/latest/en/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/19.5.0/INSTALL.md) @19.5.0
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/0.15.0/README.md) @0.15.0

## Procedure

### Prepare CentOS 8 host

Disable Firewalld. This is [recommended by K3s](https://rancher.com/docs/k3s/latest/en/advanced/#additional-preparation-for-red-hat-centos-enterprise-linux).

```bash
sudo systemctl disable firewalld --now
```

Install required packages to deploy AWX Operator and AWX.

```bash
sudo dnf install -y git make
```

### Install K3s

Install K3s with `--write-kubeconfig-mode 644` to make config file (`/etc/rancher/k3s/k3s.yaml`) readable by non-root user.

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

### Install AWX Operator

Install specified version of AWX Operator. Note that this procedure is applicable only for AWX Operator `0.14.0` or later. If you want to deploy `0.13.0` or earlier version of AWX Operator, refer [📝Tips: Deploy older version of AWX Operator](tips/deploy-older-operator.md)

```bash
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout 0.15.0
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

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
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

Note that by default AWX can't be started unless your K3s node has at least 2 CPUs and 4 GB RAM available. If your K3s node is smaller than this and you want to remove this restriction, consider uncommenting the following three lines in `base/awx.yaml`.

```yaml
...
spec:
  ...
  # To run AWX on a node that does not meet resource requirements,
  # uncomment the following three lines
  web_resource_requirements: {}     👈👈👈
  task_resource_requirements: {}     👈👈👈
  ee_resource_requirements: {}     👈👈👈
```

### Deploy AWX

Deploy AWX, this takes few minutes to complete.

```bash
kubectl apply -k base
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0   
----------
```

Required objects has been deployed next to AWX Operator in `awx` namespace.

```bash
$ kubectl -n awx get awx,all,ingress,secrets
NAME                      AGE
awx.awx.ansible.com/awx   4m17s

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-operator-controller-manager-68d787cfbd-j6k7z   2/2     Running   0          7m43s
pod/awx-postgres-0                                     1/1     Running   0          4m6s
pod/awx-84d5c45999-h7xm4                               4/4     Running   0          3m59s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.134.67    <none>        8443/TCP   7m43s
service/awx-postgres                                      ClusterIP   None            <none>        5432/TCP   4m6s
service/awx-service                                       ClusterIP   10.43.232.137   <none>        80/TCP     4m

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           7m43s
deployment.apps/awx                               1/1     1            1           3m59s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-68d787cfbd   1         1         1       7m43s
replicaset.apps/awx-84d5c45999                               1         1         1       3m59s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     4m6s

NAME                                    CLASS    HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/awx-ingress   <none>   awx.example.com   192.168.0.100   80, 443   4m

NAME                                                 TYPE                                  DATA   AGE
secret/default-token-6tp55                           kubernetes.io/service-account-token   3      7m43s
secret/awx-operator-controller-manager-token-sz6wq   kubernetes.io/service-account-token   3      7m43s
secret/awx-admin-password                            Opaque                                1      4m17s
secret/awx-postgres-configuration                    Opaque                                6      4m17s
secret/awx-secret-tls                                kubernetes.io/tls                     2      4m17s
secret/awx-app-credentials                           Opaque                                3      4m2s
secret/awx-token-jfndh                               kubernetes.io/service-account-token   3      4m2s
secret/awx-secret-key                                Opaque                                1      4m13s
secret/awx-broadcast-websocket                       Opaque                                1      4m9s
```

Now your AWX is available at `https://awx.example.com/` or the hostname you specified.

At this point, however, AWX can be accessed via HTTP as well as HTTPS. If you want to redirect HTTP to HTTPS, see [📝Tips: Redirect HTTP to HTTPS](tips/https-redirection.md).

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

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWXBackup, awxbackup-2021-06-06/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0
----------
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

Note that if you are using AWX Operator `0.12.0` or earlier, the contents of the Secret that passed through `ingress_tls_secret` parameter will not be included in this backup files. If necessary, get a dump of this Secret, or keep original certificate file and key file. In `0.13.0` or later, this secret is included in the backup file therefore you can ignore this step.

```bash
kubectl get secret awx-secret-tls -n awx -o yaml > awx-secret-tls.yaml
```

### Restoring using AWX Operator

To perfom restoration, you need to have AWX Operator running on Kubernetes. If you are planning to restore to a new environment, first prepare Kubernetes and AWX Operator by referring to the instructions on this page.

It is strongly recommended that the version of AWX Operator is the same as the version when the backup was taken. This is because the structure of the backup files differs between versions and may not be compatible. If you have upgraded AWX Operator after taking the backup, it is recommended to downgrade it for the restore. To deploy `0.13.0` or earlier version of AWX Operator, refer [📝Tips: Deploy older version of AWX Operator](tips/deploy-older-operator.md)

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

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=56   changed=0    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
----------
```

This will create AWXRestore object in the namespace, and now your AWX is restored.

```bash
$ kubectl -n awx get awxrestore
NAME                    AGE
awxrestore-2021-06-06   137m
```

Note that if you are using AWX Operator `0.12.0` or earlier, the Secret for TLS should be manually restored (or create newly using original certificate and key file). This step is not required for `0.13.0` or later.

```bash
kubectl apply -f awx-secret-tls.yaml
```

## Additional Guides

- [📁 **Deploy Private Git Repository on Kubernetes**](git)
  - To use AWX with SCM, this repository includes the manifests to deploy [Gitea](https://gitea.io/en-us/).
  - See [📝`git/README.md`](git) for instructions.
- [📁 **Deploy Private Container Registry on Kubernetes**](registry)
  - To use Execution Environments in AWX (AWX-EE), we have to push the container image built with `ansible-builder` to the container registry.
  - If we don't want to push our container images to Docker Hub or other cloud services, we can deploy a private container registry on K3s.
  - See [📝`registry/README.md`](registry) for instructions.
- [📁 **Deploy Private Galaxy NG on Docker or Kubernetes** (Experimental)](galaxy)
  - Deploy our own Galaxy NG instance.
  - **Note that the containerized implementation of Galaxy NG is not supported at this time.**
  - **All information on the page is for development, testing and study purposes only.**
  - See [📝`galaxy/README.md`](galaxy) for instructions.
- [📁 **Use SSL Certificate from Public ACME CA**](acme)
  - To use a certificate from public ACME CA such as Let's Encrypt or ZeroSSL instead of Self-Signed certificate.
  - See [📝`acme/README.md`](acme) for instructions.
- [📁 **Use Ansible Builder**](builder)
  - Use Ansible Builder to build our own Execution Environment.
  - See [📝`builder/README.md`](builder) for instructions.
- [📁 **Use Ansible Runner**](runner)
  - Use Ansible Runner to run playbook using Execution Environment.
  - See [📝`runner/README.md`](runner) for instructions.
- [📁 **Use Customized Pod Specification for your Execution Environment**](containergroup)
  - We can customize the specification of the Pod of the Execution Environment using **Container Group**.
  - See [📝`containergroup/README.md`](containergroup) for instructions.
- [📁 **Tips**](tips)
  - [📝Expose `/etc/hosts` to Pods on K3s](tips/expose-hosts.md)
  - [📝Redirect HTTP to HTTPS](tips/https-redirection.md)
  - [📝Uninstall deployed resouces](tips/uninstall.md)
  - [📝Deploy older version of AWX Operator](tips/deploy-older-operator.md)
  - [📝Upgrade AWX Operator from 0.13.0 or earlier to 0.14.0 or later](tips/upgrade-operator.md)
