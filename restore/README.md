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

To perfom restoration, you need to have AWX Operator running on Kubernetes. If you are planning to restore to a new environment, first prepare Kubernetes and AWX Operator by referring to [the instructions on the main guide](../README.md).

It is strongly recommended that the version of AWX Operator is the same as the version when the backup was taken. This is because the structure of the backup files differs between versions and may not be compatible. If you have upgraded AWX Operator after taking the backup, it is recommended to downgrade AWX Operator first before perfoming the restore. To deploy `0.13.0` or earlier version of AWX Operator, refer [📝Tips: Deploy older version of AWX Operator](../tips/deploy-older-operator.md)

### Prepare for Restore

If your AWX instance is running, it is recommended that it be deleted along with any data in the PV for the PostgreSQL first, in order to restore to be succeeded.

```bash
kubectl -n awx delete awx awx
sudo rm -rf /data/postgres
```

Then prepare directories for your PVs. `/data/projects` is required if you are restoring the entire AWX to a new environment.

```bash
sudo mkdir -p /data/postgres
sudo mkdir -p /data/projects
sudo chmod 755 /data/postgres
sudo chown 1000:0 /data/projects
```

Then deploy Persistent Volume and Persistent Volume Claim. It is recommended that making the size of PVs and PVCs same as the PVs which your AWX used when the backup was taken.

```bash
kubectl apply -k restore
```

### Restore Manually

Modify the name of the AWXRestore object in `restore/awxrestore.yaml`.

```yaml
...
kind: AWXRestore
metadata:
  name: awxrestore-2021-06-06     👈👈👈
  namespace: awx
...
```

If you want to restore from AWXBackup object, specify its name in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from AWXBackup object
  backup_pvc_namespace: awx
  backup_name: awxbackup-2021-06-06     👈👈👈
...
```

If the AWXBackup object no longer exists, place the backup files and specify the name of the PVC and directory in `restore/awxrestore.yaml`.

```yaml
...
  # Parameters to restore from existing files on PVC (without AWXBackup object)
  backup_pvc_namespace: awx
  backup_pvc: awx-backup-claim     👈👈👈
  backup_dir: /backups/tower-openshift-backup-2021-06-06-10:51:49     👈👈👈
...
```

Then invoke restore by applying this manifest file.

```bash
kubectl apply -f restore/awxrestore.yaml
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```

When the restore complete successfully, the logs end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=67   changed=0    unreachable=0    failed=0    skipped=42   rescued=0    ignored=0
```

This will create AWXRestore object in the namespace, and now your AWX is restored.

```bash
$ kubectl -n awx get awxrestore
NAME                    AGE
awxrestore-2021-06-06   137m
```

Note that if you are using AWX Operator `0.12.0` or earlier, the Secret for TLS should be manually restored (or create newly using original certificate and key file). This step is not required for `0.13.0` or later.

```bash
kubectl apply -f awx-secret-tls.yaml
```
