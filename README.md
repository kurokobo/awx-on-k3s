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
- [Deploy Private Git Repository](#deploy-private-git-repository)
- [Deploy Private Container Registry](#deploy-private-container-registry)
- [Use Ansible Builder](#use-ansible-builder)
- [Use Ansible Runner](#use-ansible-runner)
- [Additional Configuration for AWX](#additional-configuration-for-awx)
  - [Configure AWX to use Git Repository with Self-Signed Certificate](#configure-awx-to-use-git-repository-with-self-signed-certificate)
  - [Expose your /etc/hosts to K3s](#expose-your-etchosts-to-k3s)

## Environment

- Tested on:
  - CentOS 8 (Minimal)
- Products that will be deployed:
  - AWX-Operator 0.12.0
  - AWX Version 19.2.2
  - PostgreSQL 12

## References

- [K3s - Lightweight Kubernetes](https://rancher.com/docs/k3s/latest/en/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/19.2.2/INSTALL.md) @19.2.2
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/0.12.0/README.md) @0.12.0

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
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/0.12.0/deploy/awx-operator.yaml
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
      - password=Ansible123!!     👈👈👈
      - type=managed

  - name: awx-admin-password
    type: Opaque
    literals:
      - password=Ansible123!!     👈👈👈
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
localhost                  : ok=51   changed=2    unreachable=0    failed=0    skipped=32   rescued=0    ignored=0
-------------------------------------------------------------------------------
```

Required objects has been deployed in `awx` namespace.

```bash
$ kubectl get all -n awx
NAME                      READY   STATUS    RESTARTS   AGE
pod/awx-postgres-0        1/1     Running   0          4m30s
pod/awx-b47fd55cd-d8dqj   4/4     Running   0          4m22s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-postgres   ClusterIP   None            <none>        5432/TCP   4m30s
service/awx-service    ClusterIP   10.43.159.187   <none>        80/TCP     4m24s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx   1/1     1            1           4m22s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-b47fd55cd   1         1         1       4m22s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     4m30s
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
$ kubectl get awxbackup -n awx
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

Note that the contents of the Secret that passed through `ingress_tls_secret` parameter will not be included in this backup files. If necessary, get a dump of this Secret, or keep original certificate file and key file.

```bash
kubectl get secret awx-secret-tls -n awx -o yaml > awx-secret-tls.yaml
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
localhost                  : ok=53   changed=2    unreachable=0    failed=0    skipped=30   rescued=0    ignored=0
-------------------------------------------------------------------------------
```

This will create AWXRestore object in the namespace.

```bash
$ kubectl get awxrestore -n awx
NAME                    AGE
awxrestore-2021-06-06   137m
```

Then restore the Secret for TLS manually (or create newly using original certificate and key file).

```bash
kubectl apply -f awx-secret-tls.yaml
```

## Deploy Private Git Repository

To use AWX with SCM, this repository includes the manifests to deploy [Gitea](https://gitea.io/en-us/).

See [📝`git/README.md`](git) for instructions.

## Deploy Private Container Registry

To use Execution Environments in AWX (AWX-EE), we have to push the container image built with `ansible-builder` to the container registry.

If we don't want to push our container images to Docker Hub or other cloud services, we can deploy a private container registry on K3s.

See [📝`registry/README.md`](registry) for instructions.

## Use Ansible Builder

See [📝`builder/README.md`](builder) for instructions.

## Use Ansible Runner

See [📝`runner/README.md`](runner) for instructions.

## Additional Configuration for AWX

### Configure AWX to use Git Repository with Self-Signed Certificate

1. Add Credentials for SCM
2. Allow Self-Signed Certificate such as this Gitea
   - Open `Settings` > `Jobs settings` in AWX
   - Press `Edit` and scroll down to `Extra Environment Variables`, then add `"GIT_SSL_NO_VERIFY": "True"` in `{}`
   - Press `Save`

### Expose your /etc/hosts to K3s

If we don't have a DNS server and are using `/etc/hosts`, we will need to do some additional tasks to get the Pods on K3s to resolve names according to `/etc/hosts`.

This is necessary for AWX to resolve the hostname for your Private Git Repository or pull images from the Container Registry.

One easy way to do this is to use `dnsmasq`.

1. Add entries to `/etc/hosts` on your K3s host. Note that the IP addresses have to be replaced with your K3s host's one.

   ```bash
   sudo echo "192.168.0.100 awx.example.com" >> /etc/hosts
   sudo echo "192.168.0.100 registry.example.com" >> /etc/hosts
   sudo echo "192.168.0.100 git.example.com" >> /etc/hosts
   ```

2. Install and start `dnsmasq` with default configuration.

   ```bash
   sudo dnf install dnsmasq
   sudo systemctl enable dnsmasq --now
   ```

3. Create new `resolv.conf` to use K3s. Note that the IP addresses have to be replaced with your K3s host's one.

   ```bash
   sudo echo "nameserver 192.168.0.100" > /etc/rancher/k3s/resolv.conf
   ```

4. Add `--resolv-conf /etc/rancher/k3s/resolv.conf` as an argument for `k3s server` command.

   ```bash
   $ cat /etc/systemd/system/k3s.service
   ...
   ExecStart=/usr/local/bin/k3s \
       server \
           '--write-kubeconfig-mode' \
           '644' \
           '--resolv-conf' \     👈👈👈
           '/etc/rancher/k3s/resolv.conf' \     👈👈👈
   ```

5. Restart K3s.

   ```bash
   sudo systemctl restart k3s
   ```
