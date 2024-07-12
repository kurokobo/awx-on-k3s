<!-- omit in toc -->
# Restore AWX using AWX Operator

The AWX Operator `0.10.0` or later has the ability to restore AWX in easy way.

This guide is specifically designed to use with the AWX which deployed using [the main guide on this repository](../README.md).

You can also refer [the official instructions](https://github.com/ansible/awx-operator/tree/devel/roles/backup) for more information.

<!-- omit in toc -->
## Table of Contents

- [Instruction](#instruction)
  - [Prepare for Restore](#prepare-for-restore)
  - [Restore Manually](#restore-manually)

## Instruction

To perform restoration, you need to have AWX Operator running on Kubernetes. If you are planning to restore to a new environment, first prepare Kubernetes and AWX Operator by referring to [the instructions on the main guide](../README.md).

It is strongly recommended that the version of AWX Operator is the same as the version when the backup was taken. This is because the structure of the backup files differs between versions and may not be compatible. If you have upgraded AWX Operator after taking the backup, it is recommended to downgrade AWX Operator first before performing the restore. To deploy `0.13.0` or earlier version of AWX Operator, refer [📝Tips: Deploy older version of AWX Operator](../tips/deploy-older-operator.md)

Some manual additions, such as [the HSTS configuration](../tips/enable-hsts.md) or [similar tips](../tips/README.md) will not be restored automatically, and will have to be reapplied after restoring. AWX may not be fully functional depending on the missing manual additions after restoring.

### Prepare for Restore

If your AWX instance is running, it is recommended that it be deleted along with PVC and PV for the PostgreSQL first, in order to restore to be succeeded.

<!-- shell: restore: uninstall -->
```bash
# Delete AWX resource, PVC, and PV
kubectl -n awx delete pvc postgres-15-awx-postgres-15-0 --wait=false
kubectl -n awx delete awx awx
kubectl delete pv awx-postgres-15-volume
```

<!-- shell: restore: delete directories -->
```bash
# Delete any data in the PV
sudo rm -rf /data/postgres-15
```

Then prepare directories for your PVs. `/data/projects` is required if you are restoring the entire AWX to a new environment.

<!-- shell: restore: create directories -->
```bash
sudo mkdir -p /data/postgres-15/data
sudo chown 26:0 /data/postgres-15/data
sudo chmod 700 /data/postgres-15/data
sudo mkdir -p /data/projects
sudo chown 1000:0 /data/projects
```

Then deploy PV and PVC. It is recommended that making the size of PVs and PVCs same as the PVs which your AWX used when the backup was taken.

<!-- shell: restore: deploy -->
```bash
kubectl apply -k restore
```

### Restore Manually

Modify the name of the AWXRestore object in `restore/awxrestore.yaml`.

```yaml
...
kind: AWXRestore
metadata:
  name: awxrestore-2021-06-06   👈👈👈
  namespace: awx
...
```

If you want to restore from AWXBackup object, specify its name in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from AWXBackup object
  backup_name: awxbackup-2021-06-06   👈👈👈
...
```

If the AWXBackup object no longer exists, place the backup files under `/data/backup/<backup directory>` (e.g. `/data/backup/tower-openshift-backup-2021-06-06-105149`) and specify the name of the PVC and directory in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from existing files on PVC (without AWXBackup object)
  backup_pvc: awx-backup-claim                                    👈👈👈
  backup_dir: /backups/tower-openshift-backup-2021-06-06-105149   👈👈👈
...
```

Then invoke restore by applying this manifest file.

<!-- shell: restore: restore -->
```bash
kubectl apply -f restore/awxrestore.yaml
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

<!-- shell: restore: gather logs -->
```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

When the restore complete successfully, the logs end with:

```bash
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=92   changed=0    unreachable=0    failed=0    skipped=81   rescued=0    ignored=1
```

This will create AWXRestore object in the namespace, and now your AWX is restored.

<!-- shell: restore: get resources -->
```bash
$ kubectl -n awx get awxrestore
NAME                    AGE
awxrestore-2021-06-06   137m
```
