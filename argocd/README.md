<!-- omit in toc -->
# Enable Continuous Delivery (CD) for AWX using Argo CD

This guide shows minimal step-by-step examples to enable continuous delivery (CD) for AWX Operator and AWX using Argo CD, by covering following topics:

- Deploying Argo CD on K3s
- Deploying AWX by referring Git repository
- Putting GitOps into practice: Upgrading AWX

For more details about Argo CD, see [the official documentation](https://argo-cd.readthedocs.io/en/stable/).

<!-- omit in toc -->
## Table of Contents

- [Deploy Argo CD](#deploy-argo-cd)
- [Prepare Git repository](#prepare-git-repository)
- [Deploy AWX Operator and AWX](#deploy-awx-operator-and-awx)
- [Putting GitOps into practice: Upgrading AWX](#putting-gitops-into-practice-upgrading-awx)
- [Tips](#tips)
  - [Official Documentation](#official-documentation)
  - [Using the CLI](#using-the-cli)
  - [Declarative Setup](#declarative-setup)

## Deploy Argo CD

This repository contains customized required files to deploy latest stable Argo CD on K3s.

Modify hostname in `argocd/ingressroute,yaml`.

```yaml
...
spec:
  ...
  routes:
    - kind: Rule
      match: Host(`argocd.example.com`)                                                    👈👈👈
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
appproject.argoproj.io/default   28s
```

Now the Web UI is available on `https://argocd.example.com/` (or with the hostname you've specified). You can login to Argo CD using user `admin` and password `ArgoCD123!`.

## Prepare Git repository

Argo CD uses a Git repository as a _single source of truth_, so everything you want to deploy on K3s have to be stored as codes on Git repository.

This repository contains `kustomization.yaml` under [📂 example](example) directory to deploy AWX Operator and AWX with following spec.

> [!NOTE]
> This is for demonstrate purpose so the configuration under [📂 example](example) is simplified than the configuration used in [the main guide](../).

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

To proceed this tutorial, fork this repository on GitHub or create new repository on any Git repository and put the files under [📂 example](example) directory to it. Of course you can use [📂 Gitea on K3s](../git) to store manifests.

## Deploy AWX Operator and AWX

> [!TIP]
> If your repository requires authentication, add your repository and credential first by `Settings` > `Repositories` > `CONNECT REPO` on the Web UI. See [the official documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/) for detailed instructions.

Open the Web UI for Argo CD, then click `NEW APP` on `Applications` page. Fill the form as follows:

| Section | Key | Value |
| - | - | - |
| `GENERAL` | `Application Name` | `awx-argocd` |
| `GENERAL` | `Project Name` | `default` |
| `GENERAL` | `SYNC POLICY` | `Automatic` |
| `SOURCE` | `Repository URL` | The URL for your repository, e.g. `https://github.com/kurokobo/awx-on-k3s.git` |
| `SOURCE` | `Revision` | `HEAD`, branch, or tag, e.g. `HEAD` |
| `SOURCE` | `Path` | The path to the directory where contains `kustomization.yaml`, e.g. `./argocd/example` |
| `DESTINATION` | `Cluster URL` | `https://kubernetes.default.svc` |

Finally click `CREATE`.

Soon Argo CD starts synchronize all objects that defined in manifests in your `kustomization.yaml` to exist on your K3s, since you have specified `Automatic` for `SYNC POLICY`. You can review the resource tree and detailed deployment progress by `Applications` > `awx-argocd` on the Web UI.

Once the deployment is completed, you can access AWX on `http://awx-argocd.example.com/` (or with the hostname you've specified).

## Putting GitOps into practice: Upgrading AWX

Upgrading AWX can be done by upgrading AWX Operator.

To upgrade AWX Operator, simply change the version in `argocd/kustomization.yaml` and commit the change to your repository.

For example, to upgrade AWX Operator to 2.12.1, modify `argocd/kustomization.yaml` as follows:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: awx-argocd
...

resources:
  - github.com/ansible/awx-operator/config/default?ref=2.12.1     👈👈👈
  - awx.yaml

images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.12.1                                                👈👈👈
```

Then Argo CD will automatically detect the change and start the synchronization, since you have specified `Automatic` for `SYNC POLICY`.

> [!NOTE]
> By default, Argo CD polls the repositories every 3 minutes. See [the official documentation](https://argo-cd.readthedocs.io/en/stable/faq/#how-often-does-argo-cd-check-for-changes-to-my-git-or-helm-repository) for more details.

Your repository is a _single source of truth_ and your AWX is stored as codes in your repository, so all the steps to upgrade AWX are only update the codes in your repository. This is the power of GitOps.

## Tips

### Official Documentation

This guide only provides minimal examples to enable Continuous Delivery (CD) for AWX using Argo CD. Argo CD is a powerful tool and has many features. Refer to [the official documentation](https://argo-cd.readthedocs.io/en/stable/) for more details about Argo CD.

### Using the CLI

There is a CLI tool for Argo CD, which is useful to manage Argo CD and applications on Argo CD. Refer to [the CLI installation documentation](https://argo-cd.readthedocs.io/en/stable/cli_installation/) and [the command reference](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd/) for more details.

### Declarative Setup

Argo CD stores all configurations as resources on K3s, such as ConfigMaps, Secrets, Applications, and AppProjects. You can review created resources and its definitions as follows, for example:

```bash
$ kubectl -n argocd get appproject
NAME      AGE
default   29m

$ kubectl -n argocd get appproject default -o yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  ...
  name: default
  namespace: argocd
  ...
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: '*'
    server: '*'
  sourceRepos:
  - '*'
...
```

```bash
$ kubectl -n argocd get application
NAME         SYNC STATUS   HEALTH STATUS
awx-argocd   Synced        Healthy

$ kubectl -n argocd get application awx-argocd -o yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  ...
  name: awx-argocd
  namespace: argocd
  ...
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: ./argocd/example
    repoURL: https://github.com/kurokobo/awx-on-k3s.git
    targetRevision: HEAD
  syncPolicy:
    automated: {}
...
```

This means that you can manage all configurations as codes.

In this guide, The Web UI is used to define applications, but you can also create applications by applying YAML files. Refer to [the official documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/) for more details about declarative setup for Argo CD itself and applications on Argo CD.
