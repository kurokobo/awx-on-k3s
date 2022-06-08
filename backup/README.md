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
- [Appendix: Back up AWX using Ansible](#appendix-back-up-awx-using-ansible)

## Instruction

### Prepare for Backup

Prepare directories for Persistent Volumes to store backup files that defined in `backup/pv.yaml`. This guide use the `hostPath` based PV to make it easy to understand.

```bash
sudo mkdir -p /data/backup
```

Then deploy Persistent Volume and Persistent Volume Claim.

```bash
kubectl apply -k backup
```

### Back up AWX manually

Modify the name of the AWXBackup object in `backup/awxbackup.yaml`.

```yaml
...
kind: AWXBackup
metadata:
  name: awxbackup-2021-06-06     👈👈👈
  namespace: awx
...
```

Then invoke backup by applying this manifest file.

```bash
kubectl apply -f backup/awxbackup.yaml
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```

When the backup completes successfully, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWXBackup, awxbackup-2021-06-06/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0
```

This will create AWXBackup object in the namespace and also create backup files in the Persistent Volume. In this example those files are available at `/data/backup`.

```bash
$ kubectl -n awx get awxbackup
NAME                   AGE
awxbackup-2021-06-06   6m47s
```

```bash
$ ls -l /data/backup/
total 0
drwxr-xr-x. 2 root root 59 Jun  5 06:51 tower-openshift-backup-2021-06-06-10:51:49

$ ls -l /data/backup/tower-openshift-backup-2021-06-06-10\:51\:49/
total 736
-rw-r--r--. 1 root             root    749 Jun  6 06:51 awx_object
-rw-r--r--. 1 root             root    482 Jun  6 06:51 secrets.yml
-rw-------. 1 systemd-coredump root 745302 Jun  6 06:51 tower.db
```

## Appendix: Back up AWX using Ansible

An example simple playbook for Ansible is also provided in this repository. This can be used with `ansible-playbook`, `ansible-runner`, and AWX. It can be also used with the scheduling feature on AWX too.

Refer [📁 **Appendix: Back up AWX using Ansible**](ansible) for details.
