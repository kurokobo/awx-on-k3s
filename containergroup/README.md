<!-- omit in toc -->
# Customize Pod Specification for Execution Environment

You can customize the specification of the Pod of the Execution Environment using **Container Group**.

In this example, we make the Execution Environment to work with the Pod with following specification.

- Run in a different namespace `ee-demo` instead of default one
- Have an additional label `app: ee-demo-pod`
- Have `requests` and `limits` for CPU and Memory resources
- Mount PVC as `/etc/demo`
- Run on the node with the label `awx-node-type: demo` using `nodeSelector`
- Have custom environment variable `MY_CUSTOM_ENV`
- Use custom DNS server `192.168.0.219` in addition to the default DNS servers

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare host and kubernetes](#prepare-host-and-kubernetes)
  - [Create Container Group](#create-container-group)
- [Quick Testing](#quick-testing)

## Procedure

### Prepare host and kubernetes

Prepare directories for Persistent Volumes defined in `containergroup/pv.yaml`.

```bash
sudo mkdir -p /data/demo
```

Create Namespace, PV, and PVC.

```bash
kubectl apply -k containergroup
```

Add label to the node.

```bash
$ kubectl label node kuro-awx01.kuro.lab awx-node-type=demo

$ kubectl get node --show-labels
NAME                  STATUS   ROLES                  AGE    VERSION        LABELS
kuro-awx01.kuro.lab   Ready    control-plane,master   3d7h   v1.21.2+k3s1   awx-node-type=demo,...
```

Copy `awx` role and `awx` rolebinding to new `ee-demo`, to assign `awx` role on `ee-demo` to `awx` serviceaccount on `awx` namespace.

```bash
$ kubectl -n awx get role awx -o json | jq '.metadata.namespace="ee-demo" | del(.metadata.ownerReferences)' | kubectl create -f -

$ kubectl -n ee-demo get role
NAME   CREATED AT
awx    2021-07-21T15:59:45Z

$ kubectl -n awx get rolebinding awx -o json | jq '.metadata.namespace="ee-demo" | del(.metadata.ownerReferences) | .subjects[0].namespace="awx"' | kubectl create -f -

$ kubectl -n ee-demo describe rolebinding awx
Name:         awx
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  awx
Subjects:
  Kind            Name  Namespace
  ----            ----  ---------
  ServiceAccount  awx   awx
```

Note that this is a little tricky but super useful way to duplicate resource between namespace. `jq` command is required.

### Create Container Group

You can create new Container Group by `Administration` > `Instance Group` > `Add`.

Enable `Customize pod specification` and define specification as following.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: ee-demo
  labels:
    app: ee-demo-pod
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
    - image: 'quay.io/ansible/awx-ee:latest'
      name: worker
      args:
        - ansible-runner
        - worker
        - '--private-data-dir=/runner'
      env:
        - name: MY_CUSTOM_ENV
          value: 'This is my custom environment variable'
      resources:
        requests:
          cpu: 500m
          memory: 100Mi
        limits:
          cpu: 1000m
          memory: 200Mi
      volumeMounts:
        - name: demo-volume
          mountPath: /etc/demo
  nodeSelector:
    awx-node-type: demo
  dnsConfig:
    nameservers:
      - 192.168.0.219
  volumes:
    - name: demo-volume
      persistentVolumeClaim:
        claimName: demo-claim
```

This is the customized manifest to achieve;

- Running in a different namespace `ee-demo` instead of default one
- Having an additional label `app: ee-demo-pod`
- Having `requests` and `limits` for CPU and Memory resources
- Mounting PVC as `/etc/demo`
- Running on the node with the label `awx-node-type: demo` using `nodeSelector`
- Having custom environment variable `MY_CUSTOM_ENV`
- Using custom DNS server `192.168.0.219` in addition to the default DNS servers

You can also change `image`, but it will be overridden by specifying Execution Environment for the Job Template, Project Default, or Global Default.

## Quick Testing

The Container Group that to be used can be specified as `Instance Groups` in the Job Template. After specifying and running the Job, you can see the result as follows.

The Pod for the Job is running in `ee-demo` namespace.

```bash
$ kubectl -n ee-demo get pod
NAME                      READY   STATUS    RESTARTS   AGE
automation-job-50-qsjbp   1/1     Running   0          17s
```

The Pod has your own specification as defined above. Note that the `image` in example output below has been overridden by the Execution Environment which defined in Job Template.

```bash
$ kubectl -n ee-demo get pod automation-job-50-qsjbp -o yaml
...
metadata:
  ...
  labels:
    ...
    app: ee-demo-pod
...
spec:
  containers:
    ...
    env:
    - name: MY_CUSTOM_ENV
      value: This is my custom environment variable
    image: registry.example.com/ansible/ee:2.12-custom
    ...
    resources:
      limits:
        cpu: "1"
        memory: 200Mi
      requests:
        cpu: 500m
        memory: 100Mi
    ...
    volumeMounts:
    - mountPath: /etc/demo
      name: demo-volume
    ...
  dnsConfig:
    nameservers:
    - 192.168.0.219
  nodeSelector:
    awx-node-type: demo
  ...
  volumes:
  - name: demo-volume
    persistentVolumeClaim:
      claimName: demo-claim
  ...
```
