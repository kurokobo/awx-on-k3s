<!-- omit in toc -->
# Alternative methods to manage AWX Operator and AWX

While [the main guide](../README.md) covers the most straightforward steps to deploy AWX Operator and AWX, but AWX Operator can also be managed with Helm and Operator Lifecycle Manager.

<!-- omit in toc -->
## Table of Contents

- [Helm](#helm)
  - [Deployment](#deployment)
    - [Install `helm` command](#install-helm-command)
    - [Add new repository](#add-new-repository)
    - [Prepare required files and resources](#prepare-required-files-and-resources)
    - [Deploy AWX Operator and AWX](#deploy-awx-operator-and-awx)
  - [Management](#management)
    - [Modify `spec` for AWX](#modify-spec-for-awx)
    - [Upgrade AWX Operator and AWX](#upgrade-awx-operator-and-awx)
    - [Gather history of upgrades](#gather-history-of-upgrades)
    - [Revert a release to a previous revision](#revert-a-release-to-a-previous-revision)
  - [Undeployment](#undeployment)
- [Operator Lifecycle Manager (OLM)](#operator-lifecycle-manager-olm)
  - [Deployment](#deployment-1)
    - [Install OLM](#install-olm)
    - [Prepare required files](#prepare-required-files)
    - [Deploy AWX Operator](#deploy-awx-operator)
    - [Deploy AWX](#deploy-awx)
  - [Management](#management-1)
    - [Upgrade AWX Operator and AWX](#upgrade-awx-operator-and-awx-1)
    - [Approve deployments and upgrades](#approve-deployments-and-upgrades)
  - [Undeployment](#undeployment-1)

## Helm

[Helm](https://helm.sh/) is the package manager for Kubernetes. Refer to [the official documentation](https://helm.sh/docs/) for details.

Recent versions of AWX Operator have also released Helm charts.

- [awx-operator Helm charts | awx-operator](https://ansible.github.io/awx-operator/)
- [Glossary - Helm | Docs](https://helm.sh/docs/glossary/)

This chart includes AWX Operator as well as template for AWX, so both AWX Operator and AWX can be managed in single release.

### Deployment

#### Install `helm` command

Refer to [the official documentation](https://helm.sh/docs/intro/install/) for details.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### Add new repository

Add a repository for downloading charts, and then update the repository to make sure you get the latest list of charts.

```bash
$ helm repo add awx-operator https://ansible.github.io/awx-operator/
"awx-operator" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "awx-operator" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Available charts by the repository can be listed with the `search` command.

```bash
$ helm search repo awx-operator
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
awx-operator/awx-operator       1.0.0           1.0.0           A Helm chart for the AWX Operator
```

#### Prepare required files and resources

This chart can deploy not only AWX Operator, but also AWX instance at once by specifying `spec` for AWX instance as same as specified in `base/awx.yaml` described in [the main guide](../README.md). However, since `spec` for AWX instance in this guide requires some resources like PVs and Secrets that can't be deployed by this chart. So required resources have to be deployed in advance. To achieve this, follow the instructions in ["Prepare required files" section in the main guide](../README.md#prepare-required-files). Note that you don't need to modify `base/awx.yaml`, however.

Then, comment out or delete reference to `awx.yaml` in `base/kustomization.yaml`.

```yaml
...
resources:
  - pv.yaml
  - pvc.yaml
  # - awx.yaml   👈👈👈
```

Then create Namespace, PVs, and Secrets. Now all required resources that will be referenced in the `spec` of the AWX instance have been created.

```bash
cat <<EOF > base/namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: awx
EOF

kubectl apply -f base/namespace.yaml
kubectl apply -k base
```

Now let's get to preparing the chart. Create a file `values.yaml` to pass configurations to the chart to deploy AWX Operator and AWX.
To deploy both AWX Operator and AWX instance at once, `AWX.enabled` have to be `true`. `AWX.spec` is also important, this is the equivalent of `base/awx.yaml` in [the main guide](../README.md). Note that this is just an example so `AWX.spec.hostname` have to be modified.

```bash
cat <<EOF > base/values.yaml
AWX:
  enabled: true

  name: awx
  spec:
    admin_user: admin
    admin_password_secret: awx-admin-password

    ingress_type: ingress
    ingress_hosts:
      - hostname: awx.example.com
        tls_secret: awx-secret-tls

    postgres_configuration_secret: awx-postgres-configuration

    postgres_storage_class: awx-postgres-volume
    postgres_storage_requirements:
      requests:
        storage: 8Gi

    projects_persistence: true
    projects_existing_claim: awx-projects-claim

    web_replicas: 1
    task_replicas: 1

    web_resource_requirements: {}
    task_resource_requirements: {}
    ee_resource_requirements: {}
    init_container_resource_requirements: {}
    postgres_resource_requirements: {}
    redis_resource_requirements: {}
    rsyslog_resource_requirements: {}
EOF
```

#### Deploy AWX Operator and AWX

To deploy chart as a release, run the `install` or `update -i` command. `upgrade` is a command to upgrade a deployed release, but adding `-i` (`--install`) makes it equivalent to `install` if that release does not exist. This is convenient because the same command can be used for both installation and upgrade.

```bash
$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
$ helm -n awx upgrade -i awx awx-operator/awx-operator -f base/values.yaml
Release "awx" does not exist. Installing it now.
NAME: awx
LAST DEPLOYED: Fri Nov 18 07:13:42 2022
NAMESPACE: awx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWX Operator installed with Helm Chart version 1.0.0
```

This will deploy AWX Operator and then AWX will be deployed by AWX Operator. In a short while, you will be able to access AWX. You can review the logs and deployed resources by following commands. Refer to [the main guide](../README.md) for details.

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
kubectl -n awx get awx,all,ingress,secrets
```

Deployed release can be listed by `list` command.

```bash
$ helm -n awx list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
awx     awx             1               2022-11-18 07:13:42.953184844 +0900 JST deployed        awx-operator-1.0.0      1.0.0
```

### Management

#### Modify `spec` for AWX

To modify the `spec` for AWX, modify `values.yaml` and run the `upgrade` command again. You can use the exact same command that you used for initial deployment.

```bash
$ helm -n awx upgrade -i awx awx-operator/awx-operator -f base/values.yaml
Release "awx" has been upgraded. Happy Helming!
NAME: awx
LAST DEPLOYED: Fri Nov 18 07:25:41 2022
NAMESPACE: awx
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
AWX Operator installed with Helm Chart version 1.0.0
```

#### Upgrade AWX Operator and AWX

If you want to upgrade AWX Operator or AWX to the newer version, `update` the repository and run `upgrade` command again.

```bash
helm repo update
helm -n awx upgrade -i awx awx-operator/awx-operator -f base/values.yaml
```

If the upgraded AWX Operator has been released with CRD changes, you may need to run following command to apply the changes.

```bash
# Specify the version of the upgraded AWX Operator
VERSION="1.0.1"
kubectl apply --server-side --force-conflicts -k github.com/ansible/awx-operator/config/crd?ref=${VERSION}
```

#### Gather history of upgrades

The history of the upgrades for the release can be listed with `history` command.

```bash
$ helm -n awx history awx
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION
1               Fri Nov 18 07:13:42 2022        superseded      awx-operator-1.0.0      1.0.0           Install complete
2               Fri Nov 18 07:25:41 2022        superseded      awx-operator-1.0.0      1.0.0           Upgrade complete
3               Fri Nov 18 07:30:51 2022        deployed        awx-operator-1.0.0      1.0.0           Upgrade complete
```

#### Revert a release to a previous revision

You can `rollback` to any revision listed in `history` command. Note, however, that only the version of AWX Operator and `spec` for AWX are rolled back, not the files in the PVs. This may unintentionally break your instance in some cases.

```bash
$ helm -n awx rollback awx 2
Rollback was a success! Happy Helming!
```

### Undeployment

You can `uninstall` deployed release. But the resources created without Helm are not deleted by `uninstall` command, so they have to be deleted manually.

```bash
$ helm -n awx uninstall awx
release "awx" uninstalled

$ kubectl delete -k base
secret "awx-admin-password" deleted
secret "awx-postgres-configuration" deleted
secret "awx-secret-tls" deleted
persistentvolume "awx-postgres-15-volume" deleted
persistentvolume "awx-projects-volume" deleted
persistentvolumeclaim "awx-projects-claim" deleted

$ kubectl delete namespace awx
namespace "awx" deleted
```

## Operator Lifecycle Manager (OLM)

[Operator Lifecycle Manager (OLM)](https://olm.operatorframework.io/) is a management framework for Operators for Kubernetes. Refer to [the official documentation](https://olm.operatorframework.io/docs/) for details.

Recent versions of AWX Operator is available on [OperatorHub.io](https://operatorhub.io/) (and also on [Artifact Hub](https://artifacthub.io/packages/olm/community-operators/awx-operator)).

- [OperatorHub.io | The registry for Kubernetes Operators](https://operatorhub.io/operator/awx-operator)
- [Glossary | Operator Lifecycle Manager](https://olm.operatorframework.io/docs/glossary/)

In this method, the lifecycle of AWX Operator is managed by OLM. In the default configuration, AWX Operator is automatically upgraded when a new version is released. Also, as AWX Operator is upgraded, AWX is automatically upgraded as well unless you have set `auto_upgrade` for your AWX to `false`.

### Deployment

#### Install OLM

Refer to [the first step of the instruction that appears by `Install` button](https://operatorhub.io/operator/awx-operator) for details.

```bash
OLM_RELEASE="v0.28.0"
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${OLM_RELEASE}/install.sh | bash -s ${OLM_RELEASE}
```

Required resources are deployed on your Kubernetes cluster.

```bash
$ kubectl get olm -A
NAMESPACE   NAME                                                   AGE
olm         operatorcondition.operators.coreos.com/packageserver   76s

NAMESPACE   NAME                                     AGE
            olmconfig.operators.coreos.com/cluster   83s

NAMESPACE   NAME                                                  AGE
operators   operatorgroup.operators.coreos.com/global-operators   83s
olm         operatorgroup.operators.coreos.com/olm-operators      83s

NAMESPACE   NAME                                                       DISPLAY               TYPE   PUBLISHER        AGE
olm         catalogsource.operators.coreos.com/operatorhubio-catalog   Community Operators   grpc   OperatorHub.io   83s

NAMESPACE   NAME                                                       DISPLAY          VERSION   REPLACES   PHASE
olm         clusterserviceversion.operators.coreos.com/packageserver   Package Server   0.26.0               Succeeded


$ kubectl -n olm get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/olm-operator-f56666c56-ggqp2        1/1     Running   0          100s
pod/catalog-operator-5bb75dd968-mkrc6   1/1     Running   0          100s
pod/packageserver-5d9568694-6lxwg       1/1     Running   0          93s
pod/packageserver-5d9568694-7qnlk       1/1     Running   0          93s
pod/operatorhubio-catalog-hxvcx         1/1     Running   0          93s

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
service/packageserver-service   ClusterIP   10.43.59.22   <none>        5443/TCP    93s
service/operatorhubio-catalog   ClusterIP   10.43.102.2   <none>        50051/TCP   92s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalog-operator   1/1     1            1           100s
deployment.apps/olm-operator       1/1     1            1           100s
deployment.apps/packageserver      2/2     2            2           93s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/catalog-operator-5bb75dd968   1         1         1       100s
replicaset.apps/olm-operator-f56666c56        1         1         1       100s
replicaset.apps/packageserver-5d9568694       2         2         2       93s
```

Available package can be gathered by getting `PackageManifest` resource.

```bash
$ kubectl get packagemanifest awx-operator
NAME           CATALOG               AGE
awx-operator   Community Operators   2m
```

#### Prepare required files

To deploy AWX Operator using OLM, the Namespace, OperatorGroup, and Subscription resources are required.

The **Subscription** is an important resource for specifying how to upgrade AWX Operator by `installPlanApproval` with value `Automatic` or `Manual`. The default value is `Automatic`, which automatically upgrades AWX Operator when a new version is available at [OperatorHub.io](https://operatorhub.io/operator/awx-operator).

```bash
cat <<EOF > subscription.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: awx

---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: awx
spec:
  targetNamespaces:
    - awx

---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: awx
  namespace: awx
spec:
  channel: alpha
  name: awx-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
  # Specify "Manual" to prevent automatic upgrades
  installPlanApproval: Automatic
EOF
```

#### Deploy AWX Operator

AWX Operator is deployed by `apply`ing the manifests for Subscription.

```bash
kubectl apply -f subscription.yaml
```

If you specify `Manual` for `installPlanApproval`, approving **InstallPlan** is required. Refer to the "[Approve deployments and upgrades](#approve-deployments-and-upgrades)" section in this guide.

After a short wait, AWX Operator will be deployed in the `awx` namespace.

```bash
$ kubectl -n awx get all
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-operator-controller-manager-646965ff5b-fvjrc   2/2     Running   0          15s

NAME                                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.118.95   <none>        8443/TCP   16s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           15s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-646965ff5b   1         1         1       15s
```

You can gather metadata of deployed operator by `get`ting `ClusterServiceVersion` (`csv`).

```bash
$ kubectl -n awx get csv
NAME                  DISPLAY   VERSION   REPLACES              PHASE
awx-operator.v2.8.0   AWX       2.8.0     awx-operator.v2.7.2   Succeeded
```

#### Deploy AWX

OLM only manages AWX Operator, not AWX. Therefore, AWX deployment is required separately. Follow [the main guide](../README.md) for instructions.

### Management

#### Upgrade AWX Operator and AWX

If you have specified `Automatic` for `installPlanApproval` for your Subscription, AWX Operator is automatically upgraded when a new version is available at [OperatorHub.io](https://operatorhub.io/operator/awx-operator). Also, as AWX Operator is upgraded, AWX is automatically upgraded as well unless you have set `auto_upgrade` for your AWX to `false`.

#### Approve deployments and upgrades

If you have specified `Manual` for `installPlanApproval` for your Subscription, initial deployment and any upgrades are pending until you manually approve them.

All installations requiring approval can be listed by `get`ting InstallPlan resource. If `APPROVED` is `false`, the installation for the InstallPlan is pending.

```bash
$ kubectl -n awx get installplan
NAME            CSV                   APPROVAL   APPROVED
install-jfh85   awx-operator.v2.8.0   Manual     false
```

To approve pending `InstallPlan`, `patch` it.

```bash
kubectl -n awx patch installplan install-jfh85 --type merge -p '{"spec":{"approved":true}}'
```

This will start the pending installation.

Note that sometimes duplicated InstallPlans are generated. In this case, just approve one of them and the installation for InstallPlan will begin. It does not matter if you approve both since duplicated InstallPlans are idempotent. It makes little sense, but if you want to know which of the two is the more appropriate InstallPlan to approve, you can get it with the following command.

```bash
kubectl -n awx get subscription awx -o=jsonpath='{.status.installplan.name}'
```

### Undeployment

To uninstall deployed package, simply delete Subscription using the same file that used to deploy it.

```bash
kubectl delete -f subscription.yaml
```

To uninstall OLM, some resources should be deleted.

```bash
OLM_RELEASE="v0.28.0"
kubectl delete apiservices.apiregistration.k8s.io v1.packages.operators.coreos.com
kubectl delete -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${OLM_RELEASE}/crds.yaml
kubectl delete -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${OLM_RELEASE}/olm.yaml
```
