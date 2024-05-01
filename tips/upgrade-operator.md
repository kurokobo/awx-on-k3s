<!-- omit in toc -->
# Upgrade AWX Operator and AWX

This guide provides the procedure for the following three types of upgrading AWX Operator.

- Upgrade from `0.14.0` or later (e.g. from `0.14.0` to `0.15.0`)
- Upgrade from `0.13.0` (e.g. from `0.13.0` to `0.14.0`)
- Upgrade from `0.12.0` or earlier (e.g. from `0.12.0` to `0.13.0`)

Note that once you upgrade AWX Operator, your AWX will also be upgraded automatically to the version bundled with the upgraded AWX Operator as shown in [📝Version Mapping for AWX Operator and AWX](version-mapping.md).

[There is `image_version` parameter for AWX resource to change which image will be used](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/deploying-a-specific-version-of-awx.html), but it appears that using a version of AWX other than the one bundled with the AWX Operator [is currently not supported](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/deploying-a-specific-version-of-awx.html). Conversely, if you want to upgrade AWX, you need to plan to upgrade AWX Operator first.

<!-- omit in toc -->
## Table of Contents

- [✅ Take a backup of the old AWX instance](#-take-a-backup-of-the-old-awx-instance)
- [📝 Upgrade from `0.14.0` or later (e.g. from `0.14.0` to `0.15.0`)](#-upgrade-from-0140-or-later-eg-from-0140-to-0150)
  - [📝 Procedure](#-procedure)
- [📝 Upgrade from `0.13.0` (e.g. from `0.13.0` to `0.14.0`)](#-upgrade-from-0130-eg-from-0130-to-0140)
  - [📝 Procedure](#-procedure-1)
- [📝 Upgrade from `0.12.0` or earlier (e.g. from `0.12.0` to `0.13.0`)](#-upgrade-from-0120-or-earlier-eg-from-0120-to-0130)
- [❓ Troubleshooting](#-troubleshooting)
  - [New Pod gets stuck in `Pending` state](#new-pod-gets-stuck-in-pending-state)

## ✅ Take a backup of the old AWX instance

Before performing the upgrade, make sure that you have a backup of your old AWX.

Refer [📝README: Backing up using AWX Operator](../README.md#backing-up-using-awx-operator) to take backup using AWX Operator.

## 📝 Upgrade from `0.14.0` or later (e.g. from `0.14.0` to `0.15.0`)

If you are using AWX Operator `0.14.0` or later and want to upgrade to newer version, basically upgrade is done by deploying the new version of AWX Operator to the same namespace where the old AWX Operator is running.

> [!WARNING]
> If you are planning to upgrade AWX Operator **from `2.12.2` or earlier to `2.13.1` or later**, note that since the bundled PostgreSQL version will be changed to 15, so the following additional tasks are required.
>
> ```bash
> # Required only when upgrading from 2.12.2 or earlier to 2.13.1 or later
> sudo mkdir -p /data/postgres-15/data
> sudo chown 26:0 /data/postgres-15/data
> sudo chmod 700 /data/postgres-15/data
> cat <<EOF > pv-postgres-15.yaml
> ---
> apiVersion: v1
> kind: PersistentVolume
> metadata:
>   name: awx-postgres-15-volume
> spec:
>   accessModes:
>     - ReadWriteOnce
>   persistentVolumeReclaimPolicy: Retain
>   capacity:
>     storage: 8Gi
>   storageClassName: awx-postgres-volume
>   hostPath:
>     path: /data/postgres-15
> EOF
> kubectl apply -f pv-postgres-15.yaml
> ```

> [!WARNING]
> If you are planning to upgrade AWX Operator **from `2.0.0` to `2.0.1` or later**, note that [the `extra_volumes` and `extra_volumes` in `base/awx.yaml` for `2.0.0` as a workaround for specific issue](https://github.com/kurokobo/awx-on-k3s/blob/2.0.0/base/awx.yaml#L42-L51) causes failure of upgrading.
>
> To avoid this, follow these steps before upgrading AWX Operator. Steps 1 and 2 can also be achieved by `kubectl -n awx edit awx awx`.
>
> 1. Remove the definition of the volume and volume mount that named `awx-projects-web` in `extra_volumes` and `web_extra_volume_mounts` in your `base/awx.yaml`.
>    - If there are no other volumes or volume mounts, you can remove whole `extra_volumes` and `web_extra_volume_mounts`.
> 2. Apply modified `base/awx.yaml` by `kubectl apply -k base`
> 3. Wait for deployment for AWX to be completed
>
> Once your AWX has been deployed without volume `awx-projects-web`, your AWX can be safely upgraded. Proceed to [the next step](#-procedure).

> [!WARNING]
> If you are planning to upgrade AWX Operator **from `0.25.0` or earlier to `0.26.0` or later**, note that since the bundled PostgreSQL version will be changed to 13, so the following additional tasks are required.
>
> ```bash
> # Required only when upgrading from 0.25.0 or earlier to 0.26.0 or later
> sudo mkdir -p /data/postgres-13
> sudo chmod 755 /data/postgres-13
> cat <<EOF > pv-postgres-13.yaml
> ---
> apiVersion: v1
> kind: PersistentVolume
> metadata:
>   name: awx-postgres-13-volume
> spec:
>   accessModes:
>     - ReadWriteOnce
>   persistentVolumeReclaimPolicy: Retain
>   capacity:
>     storage: 8Gi
>   storageClassName: awx-postgres-volume
>   hostPath:
>     path: /data/postgres-13
> EOF
> kubectl apply -f pv-postgres-13.yaml
> ```

### 📝 Procedure

To upgrade your AWX Operator, perform following steps. If your target version is earlier than 2.3.0, switch this page to the desired tag instead of the `main` branch.

```bash
# Prepare required files
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
git checkout 2.3.0  # Checkout the version to upgrade to

# Deploy AWX Operator
kubectl apply -k operator
```

This will upgrade the AWX Operator first, after that, AWX will be also upgraded as well.

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
localhost                  : ok=56   changed=0    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
```

> [!NOTE]
> If your AWX Operator has upgraded from `2.12.2` or earlier to `2.13.1` or later, old PVC and PV for PostgreSQL 13 can be removed since new AWX is running with new PV for PostgreSQL 15.
>
> ```bash
> # Recommended only when upgraded from 2.12.2 or earlier to 2.13.1 or later
> kubectl -n awx delete pvc postgres-13-awx-postgres-13-0
> kubectl delete pv awx-postgres-13-volume
> sudo rm -rf /data/postgres-13
> ```

> [!NOTE]
> If your AWX Operator has upgraded from `0.25.0` or earlier to `0.26.0` or later, old PV for PostgreSQL 12 can be removed since new AWX is running with new PV for PostgreSQL 13.
>
> ```bash
> # Recommended only when upgraded from 0.25.0 or earlier to 0.26.0 or later
> kubectl delete pv awx-postgres-volume
> sudo rm -rf /data/postgres
> ```

## 📝 Upgrade from `0.13.0` (e.g. from `0.13.0` to `0.14.0`)

If you are using AWX Operator `0.13.0` and want to upgrade to newer version, you should consider the big changes in AWX Operator in `0.14.0`. [As described in the documentation](https://ansible.readthedocs.io/projects/awx-operator/en/latest/upgrade/upgrading.html#v0140), in `0.14.0`, AWX Operator changed from cluster scope to namespace scope. Also, the Operator SDK `1.x` is used.

This means that upgrading from `0.13.0` to `0.14.0` or later requires a bit of finesse, such as cleaning the old AWX Operator. **If you are using `0.12.0` or earlier and want to upgrade to `0.14.0` or later, I recommend you to [upgrade to `0.13.0` first](#-upgrade-from-0120-or-earlier-eg-from-0120-to-0130) and then come back to here to avoid unintended issue.**

In this guide, for example, perform upgrading from `0.13.0` to `0.14.0`. The AWX Operator `0.13.0` or earlier resides in the `default` namespace by default and the related AWX instance resides in the `awx` namespace, as described in this repository. After the upgrade, everything related to the AWX Operator `0.14.0` will reside in the `awx` namespace.

| Phase            | AWX Operator                    | AWX Instance                |
| ---------------- | ------------------------------- | --------------------------- |
| Before Upgrade   | `0.13.0` in `default` namespace | `19.3.0` in `awx` namespace |
| After Upgrade    | `0.14.0` in `awx` namespace     | `19.4.0` in `awx` namespace |

### 📝 Procedure

To upgrade AWX Operator, remove the old AWX Operator that is running in the `default` namespace first. In addition, remove Service Account, Cluster Role, and Cluster Role Binding that are required for old AWX Operator to work.

```bash
kubectl -n default delete deployment awx-operator
kubectl -n default delete serviceaccount awx-operator
kubectl -n default delete clusterrolebinding awx-operator
kubectl -n default delete clusterrole awx-operator
```

Since we removed only old AWX Operator, the old CRDs are still exist. Therefore, the old `awx` resource which means old AWX instance is still running in the `awx` namespace.

Finally, deploy the new AWX Operator to the `awx` namespace.

```bash
# Prepare required files
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout 0.14.0  # Checkout the version to upgrade to

# Deploy AWX Operator
export NAMESPACE=awx  # Specify the namespace where the old AWX instance exists
make deploy
```

This will update the CRDs in the cluster and create the required Service Account, Roles, etc. in the `awx` namespace. Also, AWX Operator will start working. Once AWX Operator is up and running, it will start rolling out a new version of the AWX instance automatically based on the old `awx` resource definition.

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
localhost                  : ok=56   changed=0    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
```

## 📝 Upgrade from `0.12.0` or earlier (e.g. from `0.12.0` to `0.13.0`)

If you are using `0.12.0` or earlier and want to upgrade to newer version, simply, deploy the new version of AWX Operator. This procedure can be applicable for upgrading to up to `0.13.0`. **If you want to upgrade to `0.14.0` or later, I recommend you to upgrade to `0.13.0` by following this procedure first and then [perform upgrading to `0.14.0` or later](#-upgrade-from-0130-eg-from-0130-to-0140).**

```bash
# Specify the version to upgrade to in the URL
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/0.13.0/deploy/awx-operator.yaml
```

This will upgrade the AWX Operator first, after that, AWX will be also upgraded as well.

To monitor the progress of the deployment, check the logs of `deployment/awx-operator`:

```bash
kubectl logs -f deployment/awx-operator
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl logs -f deployment/awx-operator
...
--------------------------- Ansible Task Status Event StdOut  -----------------
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0 
-------------------------------------------------------------------------------
```

## ❓ Troubleshooting

Some hints for when you got stuck during upgrade.

### New Pod gets stuck in `Pending` state

If the K3s node does not have enough free resources to deploy a new AWX instance, the new Pod for AWX gets stuck in `Pending` state.

```bash
$ kubectl -n awx get pods
NAME                                               READY   STATUS    RESTARTS   AGE
awx-7d74496d7d-d66dw                               4/4     Running   0          19d
awx-84d5c45999-55gb4                               0/4     Pending   0          10s   👈👈👈
```

Try running `kubectl -n awx describe pod <Pod Name>` and check the `Events` section at the end for the cause.

```bash
$ kubectl -n awx describe pod awx-84d5c45999-55gb4
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  106s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
  Warning  FailedScheduling  105s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
```

This means that the node does not have enough CPU or memory resources to start the Pod.

During the AWX upgrade, a rollout of the Deployment resource will be performed and temporarily two AWX Pods will be running. This means that the required Resource Requests for CPU and memory will be doubled.

For this reason, if we do not have enough free resources on our K3s node, we can manually delete the old AWX instance beforehand in order to free up resources. Note that the rollout history will be lost with this step.

```bash
$ kubectl -n awx delete deployment awx
deployment.apps "awx" deleted

$ kubectl -n awx get all
NAME                    READY   STATUS    RESTARTS   AGE
pod/awx-postgres-13-0   1/1     Running   0          8m57s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-postgres   ClusterIP   None            <none>        5432/TCP   8m57s
service/awx-service    ClusterIP   10.43.248.150   <none>        80/TCP     8m51s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     8m58s
```

Ensure that it is not the `awx` resource that should be deleted, but the `deployment` resource. If we accidentally delete the `awx` resource or any Secrets, we will not be able to upgrade successfully.

After a few minutes of waiting, our AWX Operator will successfully launch the new Deployment and the Pod for AWX.
