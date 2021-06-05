# AWX on Single Node K3s

An example implementation of AWX on single node K3s using AWX Operator, with easy-to-use simplified configuration with ownership of data and passwords.

- Accesible over HTTPS from remote host
- All data will be stored under `/data`
- Fixed (configurable) passwords for AWX and PostgreSQL
- Fixed (configurable) versions of AWX and PostgreSQL

## Environment

- Tested on:
  - CentOS 8 (Minimal)
- Products that will be deployed:
  - AWX-Operator 0.10.0
  - AWX Version 19.2.0
  - PostgreSQL 12

## References

- [K3s - Lightweight Kubernetes](https://rancher.com/docs/k3s/latest/en/)
- [INSTALL.md on ansible/awx](https://github.com/ansible/awx/blob/19.2.0/INSTALL.md) @19.2.0
- [README.md on ansible/awx-operator](https://github.com/ansible/awx-operator/blob/0.10.0/README.md) @0.10.0

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
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/0.10.0/deploy/awx-operator.yaml
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

Modify `hostname` in `base\awx.yaml`.

```yaml
...
spec:
  ingress_type: ingress
  ingress_tls_secret: awx-secret-tls
  hostname: awx.example.com     👈👈👈
...
```

Modify two `password`s in `base\kustomization.yaml`.

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
