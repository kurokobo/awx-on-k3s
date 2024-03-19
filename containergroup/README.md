<!-- omit in toc -->
# Customize Pod Specification for Execution Environment

You can customize the specification of the Pod of the Execution Environment using **Container Group**.

<!-- omit in toc -->
## Table of Contents

- [Case 1: Read and write files on K3s host during Jobs](#case-1-read-and-write-files-on-k3s-host-during-jobs)
  - [Prepare host and Kubernetes](#prepare-host-and-kubernetes)
  - [Create Container Group](#create-container-group)
  - [Quick testing](#quick-testing)
- [Case 2: Achieve complex requirements](#case-2-achieve-complex-requirements)
  - [Prepare host and Kubernetes](#prepare-host-and-kubernetes-1)
  - [Create Container Group](#create-container-group-1)
  - [Quick testing](#quick-testing-1)

## Case 1: Read and write files on K3s host during Jobs

One typical use case to use Container Group is to read or write files on K3s host during Jobs.

In this example, we make Container Group that works with the Pod with following specification.

- Mount PVC as `/data/work` that is the exact `/data/work` on the K3s host

By using this Container Group, you can read and write any files under `/data/work` on the K3s host during Jobs. Of course, these files are persisted since they are on the host file system, so you can read and write the same files in any Jobs.

> [!NOTE]
> Almost the same thing can be achieved by enabling `Expose host paths for Container Groups` and adding `"/data/work:/data/work:O"` to `Paths to expose to isolated jobs` on `Administration` > `Settings` > `Jobs settings`, but using Container Group is more flexible and recommended.
> The configuration `Paths to expose to isolated jobs` is a global setting and all Jobs mount the host file system even if it's not required. Whereas, any number of container groups can be created, with different settings for each job template.

### Prepare host and Kubernetes

Prepare directories for Persistent Volumes defined in `containergroup/case1/pv.yaml`.

```bash
sudo mkdir -p /data/work
sudo chown 1000:0 /data/work
sudo chmod 700 /data/work
```

Create PV and PVC.

```bash
kubectl apply -k containergroup/case1
```

### Create Container Group

You can create new Container Group by `Administration` > `Instance Group` > `Add` > `Add container group`.

Enable `Customize pod specification` and define specification as following.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: awx
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
    - image: quay.io/ansible/awx-ee:latest
      name: worker
      args:
        - ansible-runner
        - worker
        - '--private-data-dir=/runner'
      resources:
        requests:
          cpu: 250m
          memory: 100Mi
      volumeMounts:
        - name: awx-work-volume
          mountPath: /data/work
  volumes:
    - name: awx-work-volume
      persistentVolumeClaim:
        claimName: awx-work-claim
```

This is the customized manifest to achieve mounting PVC as `/data/work` that is the exact `/data/work` on the K3s host.

### Quick testing

The Container Group that to be used can be specified as `Instance Groups` in the Job Template.

Before proceeding the testing, create new file under `/data/work` on the K3s host.

```bash
$ echo "This file is written from K3s host." | sudo tee /data/work/demo_from_host.txt
This file is written from K3s host.

$ ls -l /data/work
total 4
-rw-r--r--. 1 root root 36 Feb 25 14:09 demo_from_host.txt
```

Then by running any playbooks on AWX that includes following tasks for example, you can confirm that the files under `/data/work` on the K3s host are readable and writable. Note some tasks that read or write files should be delegated to `localhost` (with `delegate_to: localhost`) if your playbook is not for `localhost`.

```yaml
---
- hosts: localhost
  gather_facts: false

  tasks:
    - name: Gather files under /data/work
      ansible.builtin.find:
        paths: /data/work
      register: existing_files

    - name: List files under /data/work
      ansible.builtin.debug:
        var: existing_files.files

    - name: Read a file under /data/work
      ansible.builtin.debug:
        var: lookup("ansible.builtin.file", "/data/work/demo_from_host.txt")

    - name: Write a file under /data/work
      ansible.builtin.copy:
        content: "This file is written during the task in the Job.\n"
        dest: /data/work/demo_from_job.txt

    - name: Read a file under /data/work
      ansible.builtin.debug:
        var: lookup("ansible.builtin.file", "/data/work/demo_from_job.txt")
```

This is the example output of the Job. You can see that the file written from the K3s host is readable in the Job, and writing a file in the Job is completed successfully.

```bash
TASK [Gather files under /data/work] *******************************************
ok: [localhost]

TASK [List files under /data/work] *********************************************
ok: [localhost] => {
    "existing_files.files": [
        {
            ...
            "path": "/data/work/demo_from_host.txt",
            ...
        }
    ]
}

TASK [Read a file under /data/work] ********************************************
ok: [localhost] => {
    "lookup('ansible.builtin.file', '/data/work/demo_from_host.txt')": "This file is written from K3s host."
}

TASK [Write a file under /data/work] *******************************************
changed: [localhost]

TASK [Read a file under /data/work] ********************************************
ok: [localhost] => {
    "lookup('ansible.builtin.file', '/data/work/demo_from_job.txt')": "This file is written during the task in the Job."
}
```

You can also see that the file written in the task is actually available on the K3s host.

```bash
$ ls -l /data/work
total 8
-rw-r--r--. 1 root root 36 Feb 25 14:09 demo_from_host.txt
-rw-r--r--. 1 1000 root 49 Feb 25 14:13 demo_from_job.txt

$ cat /data/work/demo_from_job.txt
This file is written during the task in the Job.
```

## Case 2: Achieve complex requirements

In this example, we make the Execution Environment to work with the Pod with following specification.

- Run in a different namespace `ee-demo` instead of default one
- Have an additional label `app: ee-demo-pod`
- Have `requests` and `limits` for CPU and Memory resources
- Mount PVC as `/etc/demo`
- Run on the node with the label `awx-node-type: demo` using `nodeSelector`
- Have custom environment variable `MY_CUSTOM_ENV`
- Use custom DNS server `192.168.0.221` in addition to the default DNS servers

### Prepare host and Kubernetes

Prepare directories for Persistent Volumes defined in `containergroup/case2/pv.yaml`.

```bash
sudo mkdir -p /data/demo
sudo chown 1000:0 /data/demo
sudo chmod 700 /data/demo
```

Create Namespace, PV, and PVC.

```bash
kubectl apply -k containergroup/case2
```

Add label to the node.

```bash
$ kubectl label node kuro-c9s01.kuro.lab awx-node-type=demo

$ kubectl get node --show-labels
NAME                  STATUS   ROLES                  AGE    VERSION        LABELS
kuro-c9s01.kuro.lab   Ready    control-plane,master   3d7h   v1.21.2+k3s1   awx-node-type=demo,...
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

You can create new Container Group by `Administration` > `Instance Group` > `Add` > `Add container group`.

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
      - 192.168.0.221
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
- Using custom DNS server `192.168.0.221` in addition to the default DNS servers

You can also change `image`, but it will be overridden by specifying Execution Environment for the Job Template, Project Default, or Global Default.

### Quick testing

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
    image: registry.example.com/ansible/ee:2.15-custom
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
    - 192.168.0.221
  nodeSelector:
    awx-node-type: demo
  ...
  volumes:
  - name: demo-volume
    persistentVolumeClaim:
      claimName: demo-claim
  ...
```
