<!-- omit in toc -->
# Upgrade AWX Operator to 0.14.0 or later

[As described in the documentation](https://github.com/ansible/awx-operator/blob/0.14.0/README.md#v0140), AWX Operator changed from cluster scope to namespace scope in `0.14.0`. Also, the Operator SDK `1.x` is used.

This means that upgrading from `0.13.0` or earlier to `0.14.0` or later requires a bit of finesse, such as cleaning the old AWX Operator.

<!-- omit in toc -->
## Table of Contents

- [Environment](#environment)
- [Procedure](#procedure)
  - [Take a backup of the old AWX instance](#take-a-backup-of-the-old-awx-instance)
  - [Delete the old AWX Operator](#delete-the-old-awx-operator)
  - [(Optional) Delete the old AWX instance](#optional-delete-the-old-awx-instance)
  - [Deploy the new AWX Operator](#deploy-the-new-awx-operator)
  - [Wait for the AWX instance to be upgraded](#wait-for-the-awx-instance-to-be-upgraded)

## Environment

In this case, we will upgrade `0.13.0` to `0.14.0`.

The AWX Operator `0.13.0` resides in the `default` namespace and the related AWX instance resides in the `awx` namespace, as described in this repository prior to `0.13.0`. After the upgrade, everything related to the AWX Operator `0.14.0` will reside in the `awx` namespace.

| Phase / Resource | AWX Operator                    | AWX Instance                |
| ---------------- | ------------------------------- | --------------------------- |
| Before Upgrade   | `0.13.0` in `default` namespace | `19.3.0` in `awx` namespace |
| After Upgrade    | `0.14.0` in `awx` namespace     | `19.4.0` in `awx` namespace |

## Procedure

This is an overview of the procedures to be described in this case.

1. Take a backup of the old AWX instance
2. Delete the old AWX Operator
3. (Optional) Delete the old AWX instance
4. Deploy the new AWX Operator
5. Wait for the AWX instance to be upgraded

### Take a backup of the old AWX instance

Before performing the upgrade, make sure that you have a backup of your old AWX.

Refer [📝README: Backing up using AWX Operator](../README.md#backing-up-using-awx-operator) to take backup using AWX Operator.

### Delete the old AWX Operator

First, remove the old AWX Operator that is running in the `default` namespace. In addition, remove Service Account, Cluster Role, and Cluster Role Binding that are required for old AWX Operator to work.

```bash
kubectl delete deployment awx-operator
kubectl delete serviceaccount awx-operator
kubectl delete clusterrolebinding awx-operator
kubectl delete clusterrole awx-operator
```

Since we removed only old AWX Operator, the old CRDs are still exist. Therefore, the old `awx` resource which means old AWX instance is still running in the `awx` namespace.

### (Optional) Delete the old AWX instance

This step should be performed if the K3s node does not have enough free resources to deploy a new AWX instance.

During the AWX upgrade, a rollout of the Deployment resource will be performed and temporarily two AWX Pods will be running. This means that the required Resource Requests for CPU and memory will be doubled.

For this reason, if we do not have enough free resources on our K3s node, we can manually delete the old AWX instance beforehand in order to free up resources. Note that the rollout history will be lost with this step.

```bash
kubectl -n awx delete deployment awx
```

Ensure that it is not the `awx` resource that should be deleted, but the `deployment` resource. If we accidentally delete the `awx` resource or any Secrets, we will not be able to upgrade successfully.

Now only PostgreSQL exists in our `awx` namespace.

```bash
$ kubectl -n awx get all
NAME                 READY   STATUS    RESTARTS   AGE
pod/awx-postgres-0   1/1     Running   0          8m57s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-postgres   ClusterIP   None            <none>        5432/TCP   8m57s
service/awx-service    ClusterIP   10.43.248.150   <none>        80/TCP     8m51s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     8m58s
```

### Deploy the new AWX Operator

Finally, deploy the new AWX Operator to the awx namespace.

```bash
# Prepare required files
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout 0.14.0

# Deploy AWX Operator
export NAMESPACE=awx
make deploy
```

This will update the CRDs and create the required Service Account, Roles, etc. in the `awx` namespace. Also, AWX Operator will start working.

### Wait for the AWX instance to be upgraded

Once AWX Operator is up and running, it will start rolling out a new version of the AWX instance automatically based on the old `awx` resource definition.

We can monitor the progress in the logs of `deployments/awx-operator-controller-manager`. Once this completed, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=56   changed=0    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
----------
```

Now your new AWX instance and AWX Operator exist in `awx` namespace.

```bash
$ kubectl -n awx get awx,all,ingress,secrets
NAME                      AGE
awx.awx.ansible.com/awx   13m

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-postgres-0                                     1/1     Running   0          13m
pod/awx-operator-controller-manager-68d787cfbd-59wr8   2/2     Running   0          3m42s
pod/awx-84d5c45999-xdspl                               4/4     Running   0          3m23s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.81.63     <none>        8443/TCP   3m42s
service/awx-postgres                                      ClusterIP   None            <none>        5432/TCP   13m
service/awx-service                                       ClusterIP   10.43.248.150   <none>        80/TCP     13m

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           3m42s
deployment.apps/awx                               1/1     1            1           3m23s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-68d787cfbd   1         1         1       3m42s
replicaset.apps/awx-84d5c45999                               1         1         1       3m23s

NAME                            READY   AGE
statefulset.apps/awx-postgres   1/1     13m

NAME                                    CLASS    HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/awx-ingress   <none>   awx.example.com   192.168.0.100   80, 443   13m

NAME                                                 TYPE                                  DATA   AGE
secret/default-token-gq4k7                           kubernetes.io/service-account-token   3      13m
secret/awx-admin-password                            Opaque                                1      13m
secret/awx-broadcast-websocket                       Opaque                                1      13m
secret/awx-secret-tls                                kubernetes.io/tls                     2      13m
secret/awx-postgres-configuration                    Opaque                                6      13m
secret/awx-secret-key                                Opaque                                1      13m
secret/awx-token-vpc22                               kubernetes.io/service-account-token   3      13m
secret/awx-operator-controller-manager-token-6m4k9   kubernetes.io/service-account-token   3      3m42s
secret/awx-app-credentials                           Opaque                                3      13m
```
