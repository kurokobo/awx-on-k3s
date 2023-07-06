<!-- omit in toc -->
# Enable Continuous Delivery (CD) for AWX using Argo CD

This guide shows minimal step-by-step examples to enable continuous delivery (CD) for AWX Operator and AWX using Argo CD, by covering following topics:

- Deploying Argo CD on K3s
- Deploying AWX by referring public Git repository
- Putting GitOps into practice: Upgrading AWX

<!-- omit in toc -->
## Table of Contents

- [Deploy Argo CD](#deploy-argo-cd)
- [Prepare Git repository](#prepare-git-repository)
- [Deploy AWX Operator and AWX](#deploy-awx-operator-and-awx)
- [Install CLI](#install-cli)

## Deploy Argo CD

This repository contains customized required files to deploy latest stable Argo CD on K3s.

Modify hostname in `ingressroute,yaml`.

```yaml
...
spec:
  ...
  routes:
    - kind: Rule
      match: Host(`argocd.example.com`)     👈👈👈
      priority: 10
      ...
    - kind: Rule
      match: Host(`argocd.example.com`) && Headers(`Content-Type`, `application/grpc`)     👈👈👈
      priority: 11
      ...
```

Create self-signed certificate file with your hostname for Argo CD. This certificate will be used for Web UI and API endpoint over HTTPS.

```bash
ARGOCD_HOST="argocd.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./argocd/tls.crt -keyout ./argocd/tls.key -subj "/CN=${ARGOCD_HOST}/O=${ARGOCD_HOST}" -addext "subjectAltName = DNS:${ARGOCD_HOST}"
```

Then deploy Argo CD. Wait few minutes and ensure all pods in the namespace `argocd` are `Running`.

```bash
$ kubectl apply -k argocd
...

$ kubectl -n argocd get all,appproject
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-applicationset-controller-55d9c78dc8-pzgdt   1/1     Running   0          31s
pod/argocd-notifications-controller-cbd4cfb6f-d7nsk     1/1     Running   0          31s
pod/argocd-redis-77bf5b886-94gts                        1/1     Running   0          31s
pod/argocd-dex-server-87d49c7-wzq9k                     1/1     Running   0          31s
pod/argocd-repo-server-59f4c89dc9-hzzcm                 1/1     Running   0          31s
pod/argocd-application-controller-0                     1/1     Running   0          31s
pod/argocd-server-78f8c6c456-w6gwx                      1/1     Running   0          31s

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.43.47.209    <none>        7000/TCP,8080/TCP            31s
service/argocd-dex-server                         ClusterIP   10.43.11.133    <none>        5556/TCP,5557/TCP,5558/TCP   31s
service/argocd-metrics                            ClusterIP   10.43.178.232   <none>        8082/TCP                     31s
service/argocd-notifications-controller-metrics   ClusterIP   10.43.241.129   <none>        9001/TCP                     31s
service/argocd-redis                              ClusterIP   10.43.8.222     <none>        6379/TCP                     31s
service/argocd-repo-server                        ClusterIP   10.43.161.251   <none>        8081/TCP,8084/TCP            31s
service/argocd-server                             ClusterIP   10.43.125.115   <none>        80/TCP,443/TCP               31s
service/argocd-server-metrics                     ClusterIP   10.43.208.232   <none>        8083/TCP                     31s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           31s
deployment.apps/argocd-notifications-controller    1/1     1            1           31s
deployment.apps/argocd-redis                       1/1     1            1           31s
deployment.apps/argocd-dex-server                  1/1     1            1           31s
deployment.apps/argocd-repo-server                 1/1     1            1           31s
deployment.apps/argocd-server                      1/1     1            1           31s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-55d9c78dc8   1         1         1       31s
replicaset.apps/argocd-notifications-controller-cbd4cfb6f     1         1         1       31s
replicaset.apps/argocd-redis-77bf5b886                        1         1         1       31s
replicaset.apps/argocd-dex-server-87d49c7                     1         1         1       31s
replicaset.apps/argocd-repo-server-59f4c89dc9                 1         1         1       31s
replicaset.apps/argocd-server-78f8c6c456                      1         1         1       31s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     31s

NAME                             AGE
appproject.argoproj.io/awx       29s
appproject.argoproj.io/default   28s
```

Now the Web UI is available on `https://argocd.example.com/` (or with the hostname you've specified). You can login to Argo CD using user `admin` and password `ArgoCD123!`.

## Prepare Git repository

Argo CD uses a Git repository as a _single source of truth_, so everything you want to deploy on K3s have to be stored as codes on Git repository.

This repository contains `kustomization.yaml` under [📂 example](example) directory to deploy AWX Operator and AWX with following spec.

Note that these configuration is a bit different from the deployment that described in [the main guide](../).

- AWX Operator
  - Use `awx-argocd` namespace
  - Old version is specified, since this will be upgraded by GitOps in the later step
- AWX
  - Use `awx-argocd` namespace
  - Use pre-configured passwords for AWX and PostgreSQL (`Ansible123!`)
  - Use Ingress to access AWX with hostname `awx-argocd.example.com`, over plain HTTP instead of HTTPS
    - To avoid storing certificate and key on Git repository, this example does not enable HTTPS.
  - Use PVCs with `local-path` storage class to persist projects and PostgreSQL
    - Since ArgoCD cannot make directories on K3s host directly, this example uses local path provisioner which is the default provisioner for K3s instead of `hostPath` based PV with explicit path.

To proceed this tutorial, fork this repository on GitHub and keep forked repository public. Alternatively you can create new repository on any Git repository and put the files under [📂 example](example) directory to it. Of course you can use [📂 Gitea on K3s](../git) to store manifests.

## Deploy AWX Operator and AWX

Open the Web UI for Argo CD, then click `NEW APP` on `Applications` page. Fill the form as follows:

| Section | Key | Value |
| - | - | - |
| `GENERAL` | `Application Name` | `awx-argocd` |
| `GENERAL` | `Project Name` | `default` |
| `GENERAL` | `SYNC POLICY` | `Automatic` |
| `SOURCE` | `Repository URL` | The URL for your repository, e.g. `https://github.com/kurokobo/awx-on-k3s.git` |
| `SOURCE` | `Revision` | `HEAD`, branch, or tag, e.g. `argocd` |
| `SOURCE` | `Path` | The path to the directory where contains `kustomization.yaml`, e.g. `./argocd/example` |
| `DESTINATION` | `Cluster URL` | `https://kubernetes.default.svc` |

Finally click `CREATE`.

Soon Argo CD starts synchronize all objects that defined in manifests in your `kustomization.yaml` to exist on your K3s, since you have specified `Automatic` for `SYNC POLICY`.

## Install CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v2.9.0/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```
