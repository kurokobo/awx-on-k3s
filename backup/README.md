<!-- omit in toc -->
# Back up AWX using AWX Operator

The AWX Operator `0.10.0` or later has the ability to back up AWX in easy way.

This guide is specifically designed to use with the AWX which deployed using [the main guide on this repository](../README.md).

You can also refer [the official instructions](https://github.com/ansible/awx-operator/tree/devel/roles/backup) for more information.

<!-- omit in toc -->
## Table of Contents

- [Instruction](#instruction)
  - [Prepare for Backup](#prepare-for-backup)
  - [Back up AWX manually](#back-up-awx-manually)
  - [Clean up Backup objects](#clean-up-backup-objects)
    - [awx-operator-controller-manager is unresponsive](#awx-operator-controller-manager-is-unresponsive)
- [Appendix: Back up AWX using Ansible](#appendix-back-up-awx-using-ansible)

## Instruction

### Prepare for Backup

Prepare directories for Persistent Volumes to store backup files that defined in `backup/pv.yaml`. This guide use the `hostPath` based PV to make it easy to understand.

<!-- shell: backup: create directories -->
```bash
sudo mkdir -p /data/backup
sudo chown 26:0 /data/backup
sudo chmod 700 /data/backup
```

Then deploy Persistent Volume and Persistent Volume Claim.

<!-- shell: backup: deploy -->
```bash
kubectl apply -k backup
```

### Back up AWX manually

Modify the name of the AWXBackup object in `backup/awxbackup.yaml`.

```yaml
...
kind: AWXBackup
metadata:
  name: awxbackup-2021-06-06   👈👈👈
  namespace: awx
...
```

Then invoke backup by applying this manifest file.

<!-- shell: backup: backup -->
```bash
kubectl apply -f backup/awxbackup.yaml
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

<!-- shell: backup: gather logs -->
```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

When the backup completes successfully, the logs end with:

```bash
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWXBackup, awxbackup-2021-06-06/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=7    changed=0    unreachable=0    failed=0    skipped=9    rescued=0    ignored=0
```

This will create AWXBackup object in the namespace and also create backup files in the Persistent Volume. In this example those files are available at `/data/backup`.

<!-- shell: backup: get resources -->
```bash
$ kubectl -n awx get awxbackup
NAME                   AGE
awxbackup-2021-06-06   6m47s
```

```bash
$ sudo ls -l /data/backup/
total 0
drwxr-xr-x. 2 26 26 59 Jun  5 06:51 tower-openshift-backup-2021-06-06-105149

$ sudo ls -l /data/backup/tower-openshift-backup-2021-06-06-105149/
total 736
-rw-------. 1 26 26   1093 Jun  6 06:51 awx_object
-rw-------. 1 26 26  17085 Jun  6 06:51 secrets.yml
-rw-r--r--. 1 26 26 833184 Jun  6 06:51 tower.db
```

### Clean up Backup Objects
!!Ensure your backup data has been transferred from the backup directory before running the commands!!

After completing and transferring the backup, it is recommended to clean up the backup resource created by AWX Operator. Keeping many of these resources around may result in the `awx-operator-controller-manager` becoming unresponsive, and thus creating new backups impossible.

Browse the existing backup objects:
```sh
kubectl get awxbackup -n awx
```

Delete the resources: 
!!Ensure your backup data has been transferred from the backup directory before running the commands!!
```sh
kubectl -n awx delete awxbackup <name> # delete a single resource
kubectl -n awx delete awxbackup --all # delete all resources
```

#### awx-operator-controller-manager is unresponsive

!!Ensure your backup data has been transferred from the backup directory before running the commands!!

If you are unable to delete the resources (e.g. the delete command hangs), the following commands will help clean up the resources. 

First, scale down the unresponsive AWX Operator controller manager:

```sh
kubectl -n awx scale deployment/awx-operator-controller-manager --replicas=0
``` 

!!Ensure your backup data has been transferred from the backup directory before running the command!!

Second, patch remove each backup resource's finalizer, ensuring the resource can get deleted. 

```sh
kubectl get awxbackup -n awx -o json | jq '.items[] | .metadata.name' | xargs -I{} kubectl patch awxbackup {} -n awx --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
``` 

Browse the existing backup objects again, to see that the resources are no longer present:

```sh
kubectl get awxbackup -n awx
```

Finally, scale up the manager:

```sh
kubectl -n awx scale deployment/awx-operator-controller-manager --replicas=1
``` 

## Appendix: Back up AWX using Ansible

An example simple playbook for Ansible is also provided in this repository. This can be used with `ansible-playbook`, `ansible-runner`, and AWX. It can be also used with the scheduling feature on AWX too.

Refer [📁 **Appendix: Back up AWX using Ansible**](ansible) for details.
