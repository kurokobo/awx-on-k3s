<!-- omit in toc -->
# Deploy older version of AWX Operator

The installation method for AWX Operator has been changed to `make` since version `0.14.0`. If you want to deploy `0.13.0` or earlier version of AWX Operator, the old procedure must be followed.

<!-- omit in toc -->
## Table of Contents

- [Install AWX Operator](#install-awx-operator)
- [Monitor the logs of AWX Operator](#monitor-the-logs-of-awx-operator)

## Install AWX Operator

If you want to deploy `0.13.0` or earlier version of AWX Operator, you can directly invoke `kubectl apply` using the manifest file on GitHub instead of using `make` command. [Official old `README.md` on `ansible/awx-operator`](https://github.com/ansible/awx-operator/blob/0.13.0/README.md) is also helpful.

```bash
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/0.13.0/deploy/awx-operator.yaml
```

The AWX Operator will be deployed to the `default` namespace.

```bash
$ kubectl -n default get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/awx-operator-69c646c48f-jmtrs   1/1     Running   0          93s

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/kubernetes             ClusterIP   10.43.0.1     <none>        443/TCP             5m57s
service/awx-operator-metrics   ClusterIP   10.43.183.1   <none>        8383/TCP,8686/TCP   70s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator   1/1     1            1           93s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-69c646c48f   1         1         1       93s
```

Once you have AWX Operator, the rest of the steps are the same as in `0.14.0` and later.

## Monitor the logs of AWX Operator

You can monitor the logs of AWX Operator by following command.

```bash
kubectl logs -f deployment/awx-operator
```
