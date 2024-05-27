<!-- omit in toc -->
# Troubleshooting Guide

Some hints and guides for when you got stuck during deployment and daily use of AWX.

<!-- omit in toc -->
## Table of Contents

- [Troubles during Deployment](#troubles-during-deployment)
  - [First Step: Investigate your Situation](#first-step-investigate-your-situation)
    - [Investigate Status and Events of the Pods](#investigate-status-and-events-of-the-pods)
    - [Investigate Logs of the Containers inside the Pods](#investigate-logs-of-the-containers-inside-the-pods)
  - [Reveal "censored" output in the AWX Operator's log](#reveal-censored-output-in-the-awx-operators-log)
  - [The Pod is `ErrImagePull` with "429 Too Many Requests"](#the-pod-is-errimagepull-with-429-too-many-requests)
  - [The Pod is `Pending` with "1 Insufficient cpu, 1 Insufficient memory." event](#the-pod-is-pending-with-1-insufficient-cpu-1-insufficient-memory-event)
  - [The Pod is `Pending` with "1 pod has unbound immediate PersistentVolumeClaims." event](#the-pod-is-pending-with-1-pod-has-unbound-immediate-persistentvolumeclaims-event)
  - [The Pod is `Running` but stucked with "wait-for-migrations" message](#the-pod-is-running-but-stucked-with-wait-for-migrations-message)
  - [The Pod for PostgreSQL is in `CrashLoopBackOff` state and shows "Permission denied" log](#the-pod-for-postgresql-is-in-crashloopbackoff-state-and-shows-permission-denied-log)
- [Troubles during Daily Use](#troubles-during-daily-use)
  - [Job failed with no output](#job-failed-with-no-output)
  - [Provisioning Callback does not work](#provisioning-callback-does-not-work)
  - [The job failed and I got "ERROR! couldn't resolve module/action" or "Failed to import the required Python library" message](#the-job-failed-and-i-got-error-couldnt-resolve-moduleaction-or-failed-to-import-the-required-python-library-message)

## Troubles during Deployment

### First Step: Investigate your Situation

You can start investigating troubles during deployment with following two things.

- **Status** and **Events** of the Pods
- **Logs** of the Containers inside the Pods

#### Investigate Status and Events of the Pods

First, check the `STATUS` for the Pods by this command.

```bash
kubectl -n awx get pod
```

If the Pods are working properly, its `STATUS` are `Running`. If your Pods are not in `Running` state e.g. `Pending`, `ImagePullBackOff` or `CrashLoopBackOff` etc., the Pods might have some problems. In the following example, the Pod `awx-84d5c45999-h7xm4` is in `Pending` state.

```bash
$ kubectl -n awx get pod
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-57867569c4-ggl29   2/2     Running   0          8m20s
awx-postgres-15-0                                  1/1     Running   0          7m26s
awx-task-5d8cd9b6b9-8ptjt                          0/4     Pending   0          6m55s
awx-web-66f89bc9cf-6zck5                           0/3     Pending   0          6m9s
```

If you have the Pods which has the unexpected state instead of `Running`, the next step is checking `Events` for the Pod. The command to get `Events` for the pod is:

```bash
kubectl -n awx describe pod <Pod Name>
```

By this command, you can get the `Events` for the Pod you specified at the end of the output.

```bash
$ kubectl -n awx describe pod awx-task-5d8cd9b6b9-8ptjt
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  106s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
  Warning  FailedScheduling  105s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
```

In most cases, you can find the reason why the Pod is not `Running` from `Events`. In the example above, I can see that it is due to lack of CPU or memory.

#### Investigate Logs of the Containers inside the Pods

The logs also helpful to get the reason why something went wrong. In particular, if the status of the Pod is `Running` but the Pod does not works as expected, you should check the logs.

The commands to get the logs are following. `-f` is optional, useful to watch the logs as well as get the logs.

```bash
# Get the logs of specific Pod.
# If the Pod includes multiple containers, container name has to be specified.
kubectl -n awx logs -f <POD>
kubectl -n awx logs -f <POD> -c <CONTAINER>

# Get the logs of specific Pod which is handled by Deployment resource.
# If the Pod includes multiple containers, container name has to be specified.
kubectl -n awx logs -f deployment/<DEPLOYMENT>
kubectl -n awx logs -f deployment/<DEPLOYMENT> -c <CONTAINER>

# Get the logs of specific Pod which is handled by Job resource.
# If the Pod includes multiple containers, container name has to be specified.
kubectl -n awx logs -f job/<JOB>
kubectl -n awx logs -f job/<JOB> -c <CONTAINER>

# Get the logs of specific Pod which is handled by StatefulSet resource
# If the Pod includes multiple containers, container name has to be specified.
kubectl -n awx logs -f statefulset/<STATEFULSET>
kubectl -n awx logs -f statefulset/<STATEFULSET> -c <CONTAINER>
```

For AWX Operator and AWX, specifically, the following commands are helpful.

- Logs of AWX Operator
  - `kubectl -n awx logs -f deployment/awx-operator-controller-manager`
- Logs of AWX related init containers
  - `kubectl -n awx logs -f deployment/awx-web -c init`
  - `kubectl -n awx logs -f deployment/awx-web -c init-projects`
  - `kubectl -n awx logs -f deployment/awx-task -c init-database`
  - `kubectl -n awx logs -f deployment/awx-task -c init-receptor`
  - `kubectl -n awx logs -f deployment/awx-task -c init-projects`
- Logs of AWX related job container
  - `kubectl -n awx logs -f job/awx-migration-<VERSION>`
- Logs of AWX related containers
  - `kubectl -n awx logs -f deployment/awx-web -c awx-web`
  - `kubectl -n awx logs -f deployment/awx-web -c awx-rsyslog`
  - `kubectl -n awx logs -f deployment/awx-web -c redis`
  - `kubectl -n awx logs -f deployment/awx-task -c awx-task`
  - `kubectl -n awx logs -f deployment/awx-task -c awx-ee`
  - `kubectl -n awx logs -f deployment/awx-task -c awx-rsyslog`
  - `kubectl -n awx logs -f deployment/awx-task -c redis`
- Logs of PostgreSQL related init container
  - `kubectl -n awx logs -f statefulset/awx-postgres-15 -c init`
- Logs of PostgreSQL related container
  - `kubectl -n awx logs -f statefulset/awx-postgres-15`

### Reveal "censored" output in the AWX Operator's log

If you've found the `FAILED` tasks while investigating AWX Operator's log, sadly sometimes it's marked as `censored` and you can't get actual log.

```bash
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
TASK [Restore database dump to the new postgresql container] ********************************
fatal: [localhost]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": true}
...
```

AWX Operator 0.23.0 or later supports making this revealed.

To achieve this, you can uncomment `no_log: false` manually under `spec` for your `awx.yaml`, `awxbackup.yaml`, or `awxrestore.yaml`, and then re-run your deployment, backup, or restoration.

```yaml
...
spec:
  ...
  # Uncomment to reveal "censored" logs
  no_log: false   👈👈👈
  ...
```

### The Pod is `ErrImagePull` with "429 Too Many Requests"

If your Pod for PostgreSQL is in `ErrImagePull` and its `Events` shows following events, this is due to [the rate limit on Docker Hub](https://docs.docker.com/docker-hub/download-rate-limit/).

```bash
$ kubectl -n awx describe pod awx-postgres-13-0
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Normal   Pulling           9s    kubelet            Pulling image "postgres:13"
  Warning  Failed            2s    kubelet            Failed to pull image "postgres:13": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/postgres:13": failed to copy: httpReadSeeker: failed open: unexpected status code https://registry-1.docker.io/v2/library/postgres/manifests/sha256:...: 429 Too Many Requests - Server message: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
  Warning  Failed            2s    kubelet            Error: ErrImagePull
  Normal   BackOff           1s    kubelet            Back-off pulling image "postgres:13"
  Warning  Failed            1s    kubelet            Error: ImagePullBackOff
```

If you follow the steps in this repository to deploy you AWX, your pull request to Docker Hub will be identified as a free, anonymous account. Therefore, you will be limited to 200 requests in 6 hours. The message "429 Too Many Requests" indicates that it has been exceeded.

To solve this, you can simply wait until the limit is freed up, or [consider giving your Docker Hub credentials to K3s by following the guide on this page](dockerhub-rate-limit.md).

### The Pod is `Pending` with "1 Insufficient cpu, 1 Insufficient memory." event

If your Pod is in `Pending` state and its `Events` shows following events, the reason is that the node does not have enough CPU and memory to start the Pod. By default AWX requires at least 2 CPUs and 4 GB RAM. In addition more resources are required to run K3s and the OS itself.

```bash
$ kubectl -n awx describe pod awx-task-5d8cd9b6b9-8ptjt
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  106s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
  Warning  FailedScheduling  105s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.   👈👈👈
```

Typical solutions are one of the following:

- **Add more CPUs or memory to your K3s node.**
  - If you have at least 3 CPUs and 5 GB RAM, AWX may work.
- **Reduce resource requests for the containers.**
  - The minimum resource requirements can be ignored by adding three lines in `base/awx.yml`.

    ```yaml
    ...
    spec:
      ...
      web_resource_requirements: {}                       👈👈👈
      task_resource_requirements: {}                      👈👈👈
      ee_resource_requirements: {}                        👈👈👈
      init_container_resource_requirements: {}            👈👈👈
      postgres_resource_requirements: {}                  👈👈👈
      redis_resource_requirements: {}                     👈👈👈
      rsyslog_resource_requirements: {}                   👈👈👈
    ```

  - You can specify more specific value for each containers. Refer [official documentation](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/containers-resource-requirements.html) for details.
  - In this way you can run AWX with fewer resources, but you may encounter performance issues.

### The Pod is `Pending` with "1 pod has unbound immediate PersistentVolumeClaims." event

If your Pod is in `Pending` state and its `Events` shows following events, the reason is that no usable Persistent Volumes are available.

```bash
$ kubectl -n awx describe pod awx-task-5d8cd9b6b9-8ptjt
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  24s   default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.   👈👈👈
```

Check the `STATUS` of your PVs and ensure your PVs doesn't have `Available` or `Bound` state.

```bash
$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                               STORAGECLASS             REASON   AGE
awx-projects-volume      2Gi        RWO            Retain           Released   awx/awx-projects-claim              awx-projects-volume               17h
awx-postgres-15-volume   8Gi        RWO            Retain           Released   awx/postgres-15-awx-postgres-15-0   awx-postgres-volume               17h
```

Probably this is the second (or more) time to deploy AWX for you. These PVs which have `Released` state are tied to your old (and probably no longer exists now) PVCs you created in the past.

There are a few things you should to know about the PVs in Kubernetes.

- Once a PV is bound from a PVC, it keeps the PVC name in its `claimRef` entry. This will be shown in the `CLAIM` column in the result of the command `kubectl get pv`.
- The `Released` state of the PV means that the PV was bound by PVC in the `claimRef` entry in the past but now the PVC does not exist. **The PV in this state cannot be bound by any PVC other than the one recorded in `claimRef`.**
- To allow the PV to bind from a PVC other than the one recorded in `claimRef`, the `claimRef` entry must be empty and the PV must has `Available` state.

To solve this, typical solutions are one of the following:

- **Patch the PV to empty `claimRef` entry for the PV.**
  - Invoke following commands:

    ```bash
    kubectl patch pv <PV Name> -p '{"spec":{"claimRef": null}}'
    ```

- **Delete the PV and recreate it.**
  - Invoke following commands:

    ```bash
    # Delete the PV
    kubectl delete pv <PV Name>

    # Recreate the PV
    kubectl apply -k base
    ```

### The Pod is `Running` but stucked with "wait-for-migrations" message

Sometimes your AWX pod is `Running` state correctly but not functional at all, and its log shows following message repeatedly.

```bash
kubectl -n awx logs -f deployment/awx-web -c awx-web
[wait-for-migrations] Waiting for database migrations...
[wait-for-migrations] Attempt 1 of 30
[wait-for-migrations] Waiting 0.5 seconds before next attempt
[wait-for-migrations] Attempt 2 of 30
[wait-for-migrations] Waiting 1 seconds before next attempt
[wait-for-migrations] Attempt 3 of 30
[wait-for-migrations] Waiting 2 seconds before next attempt
[wait-for-migrations] Attempt 4 of 30
[wait-for-migrations] Waiting 4 seconds before next attempt
...
[wait-for-migrations] Attempt 28 of 30
[wait-for-migrations] Waiting 30 seconds before next attempt
[wait-for-migrations] Attempt 29 of 30
[wait-for-migrations] Waiting 30 seconds before next attempt
...
```

This problem occurs when the AWX pod and the PostgreSQL pod cannot communicate properly. In most cases, the cause of this is the network on your K3s.

To solve this, check or try the following:

- Ensure your PostgreSQL (typically the Pod named `awx-postgres-0`, `awx-postgres-13-0`, or `awx-postgres-15-0`) is in `Running` state.
- Ensure `host` under `awx-postgres-configuration` in `base/kustomization.yaml` has correct value.
  - Specify `awx-postgres` for AWX Operator 0.25.0 or earlier, `awx-postgres-13` for `0.26.0` to `2.12.2`, `awx-postgres-15` for newer versions.
- Ensure your `firewalld`, `ufw` or any kind of firewall has been disabled on your K3s host.
- Ensure your `nm-cloud-setup` service on your K3s host is disabled if exists.
- Ensure your `awx-postgres-configuration` has correct values, especially if you're using external PostgreSQL.
- Uninstall K3s and install it again.

### The Pod for PostgreSQL is in `CrashLoopBackOff` state and shows "Permission denied" log

In this situation, your Pod for PostgreSQL is in `CrashLoopBackOff` state and its log shows following error message.

```bash
$ kubectl -n awx get pod
NAME                                               READY   STATUS             RESTARTS   AGE
awx-operator-controller-manager-57867569c4-ggl29   2/2     Running            0          8m20s
awx-postgres-13-0                                  1/1     CrashLoopBackOff   5          7m26s
awx-task-5d8cd9b6b9-8ptjt                          0/4     Running            0          6m55s
awx-web-66f89bc9cf-6zck5                           0/3     Running            0          6m9s

# On PostgreSQL 13
$ kubectl -n awx logs statefulset/awx-postgres-13
mkdir: cannot create directory '/var/lib/postgresql/data': Permission denied

# On PostgreSQL 15
$ kubectl -n awx logs statefulset/awx-postgres-15
mkdir: cannot create directory '/var/lib/pgsql/data/userdata': Permission denied
```

You should check the permissions and the owner of directories where used as PV on your K3s host.

For the PostgreSQL 13 that deployed by **AWX Operator 2.12.2 or earlier**, if you followed my guide, it would be `/data/postgres-13`. There is additional `data` directory created by K3s under `/data/postgres-13`.

```bash
$ ls -ld /data/postgres-13 /data/postgres-13/data
drwxr-xr-x. 2 root root 18 Aug 20 10:09 /data/postgres-13
drwxr-xr-x. 3 root root 20 Aug 20 10:09 /data/postgres-13/data
```

For example, `755` and `root:root` (`0:0`) should work. So you can try following commands.

```bash
sudo chown 0:0 /data/postgres-13 /data/postgres-13/data
sudo chmod 755 /data/postgres-13 /data/postgres-13/data
```

Or, you can also try `999:0` as owner/group for the directory. `999` is [the UID of the `postgres` user which used in the container](https://github.com/docker-library/postgres/blob/master/13/bullseye/Dockerfile#L13).

```bash
sudo chown 999:0 /data/postgres-13 /data/postgres-13/data
sudo chmod 755 /data/postgres-13 /data/postgres-13/data
```

For the PostgreSQL 15 that deployed by **AWX Operator 2.13.0 or later**, if you followed my guide, it would be `/data/postgres-15`. There is additional `data` directory created by K3s under `/data/postgres-15`.

```bash
$ ls -ld /data/postgres-15 /data/postgres-15/data
drwxr-xr-x. 2 root root 18 Aug 20 10:09 /data/postgres-15
drwxr-xr-x. 3   26 root 20 Aug 20 10:09 /data/postgres-15/data
```

For example, `700` and `26:0` should work. So you can try following commands. `26` is [the UID of the user which used in the container](https://github.com/sclorg/postgresql-container/blob/master/15/Dockerfile.c9s#L86).

```bash
sudo chown 26:0 /data/postgres-15 /data/postgres-15/data
sudo chmod 700 /data/postgres-15 /data/postgres-15/data
```

## Troubles during Daily Use

### Job failed with no output

If the job is invoked to a large number of hosts or is running long time, sometimes the job is marked as failed and no log will be displayed in the Output tab.

This is a problem caused by log rotation on Kubernetes. Refer [ansible/awx#10366](https://github.com/ansible/awx/issues/10366) for details.

In the case of K3s, you can reduce the possibility of this issue by changing the configuration as follows.

```bash
# Change configuration using script:
$ curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --kubelet-arg "container-log-max-files=4" --kubelet-arg "container-log-max-size=50Mi"

# If you don't want to use the script, modify /etc/systemd/system/k3s.service manually:
$ cat /etc/systemd/system/k3s.service
...
ExecStart=/usr/local/bin/k3s \
    server \
        '--write-kubeconfig-mode' \
        '644' \
        '--kubelet-arg' \                 👈👈👈
        'container-log-max-files=4' \     👈👈👈
        '--kubelet-arg' \                 👈👈👈
        'container-log-max-size=50Mi' \   👈👈👈
```

Then restart K3s. The K3s service can be safely restarted without affecting the running resources.

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### Provisioning Callback does not work

If you use Traefik which is K3s' Ingress controller as completely default, the Pod may not be able to get the client's IP address (see [k3s-io/k3s#2997](https://github.com/k3s-io/k3s/discussions/2997) for details). Therefore, the feature called Provisioning Callback in AWX does not work properly since AWX can't determine actual IP address of the remote host who request callback.

For this reason, you should fix the Traefik configuration. For a single node like doing in this repository, reconfiguring your Traefik by creating YAML file is the easy way.

```bash
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml <<EOF
---
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    hostNetwork: true
EOF
```

Then wait until your `traefik` by the following command is `1/1` `READY`.

```bash
kubectl -n kube-system get deployment traefik
```

Now your client's IP address can be passed correctly through `X-Forwarded-For` and `X-Real-Ip` headers.

The last step is modifying AWX. By default, AWX uses only `REMOTE_ADDR` and `REMOTE_HOST` headers to determine the remote host (means HTTP client). Therefore, you have to make AWX to use `X-Forwarded-For` header.

This can be achieved by modifying your `base/awx.yaml` and apply it, or simply configure `Remote Host Headers` via AWX UI.

If you want to use `base/awx.yaml` to achieve this, add following three lines to your `base/awx.yaml`.

```yaml
...
spec:
  ...
  extra_settings:                                                       👈👈👈
    - setting: REMOTE_HOST_HEADERS                                      👈👈👈
      value: "['HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR', 'REMOTE_HOST']"   👈👈👈
```

Then apply this change and wait for your AWX will be reconfigured.

```bash
kubectl apply -k base
```

You can watch its progress by following command as did when you deploy AWX at the first time.

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

Alternatively you can modify this settings via AWX UI. Move on to `Settings` > `Miscellaneous System settings` > `Edit` page, then and put following JSON strings as `Remote Host Headers`.

```json
[
  "HTTP_X_FORWARDED_FOR",
  "REMOTE_ADDR",
  "REMOTE_HOST"
]
```

Now your Provisioning Callback should work. In my environment, the name of the host in the inventory have to be defined using IP address instead of DNS hostname.

### The job failed and I got "ERROR! couldn't resolve module/action" or "Failed to import the required Python library" message

When you launch the Job Template, it may fail and you will see an error like the following:

```text
ERROR! couldn't resolve module/action 'community.postgresql.postgresql_info'. This often indicates a misspelling, missing collection, or incorrect module path.

The error appears to be in '/runner/project/site.yml': line 6, column 7, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:
  tasks:
    - community.postgresql.postgresql_info:
      ^ here
```

Alternatively, the import of Python modules may fail.

```text
...
TASK [community.postgresql.postgresql_info] ************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (psycopg2) on automation-job-12-v2gvf's Python /usr/bin/python3. Please read the module documentation and install it in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter"}
...
```

To solve this, you can build your own Execution Image, or place `requirements.yml` on the specific path in your project. Refer to [the guide about Execution Environment on this repository](../builder) for details.
