<!-- omit in toc -->
# Uninstall deployed resources

<!-- omit in toc -->
## Table of Contents

- [Uninstall resources on Kubernetes](#uninstall-resources-on-kubernetes)
- [Remove data in PVs](#remove-data-in-pvs)
- [Uninstall K3s](#uninstall-k3s)

### Uninstall resources on Kubernetes

The resources that you've deployed by `kubectl create -f (-k)` or `kubectl apply -f (-k)` commands can be also removed `kubectl delete -f (-k)` command by passing the same manifest files.

These are the example commands to delete existing AWX on K3s. Note that PVC for PostgreSQL should be removed manually since this PVC was created by not `kubectl apply -k` but AWX Operator.

<!-- shell: instance: uninstall -->
```bash
$ kubectl -n awx delete pvc postgres-15-awx-postgres-15-0 --wait=false
$ kubectl delete -k base
secret "awx-admin-password" deleted
secret "awx-postgres-configuration" deleted
secret "awx-secret-tls" deleted
persistentvolume "awx-postgres-volume" deleted
persistentvolume "awx-projects-volume" deleted
persistentvolumeclaim "awx-projects-claim" deleted
awx.awx.ansible.com "awx" deleted
```

You can also delete all resources in the specific namespace by deleting the namespace. Any PVs cannot be deleted in this way since the PVs are namespace-independent resources, so they need to be deleted manually.

```bash
$ kubectl delete ns awx
namespace "awx" deleted

$ kubectl delete pv <volume name>
persistentvolume "<volume name>" deleted
```

### Remove data in PVs

All PVs that deployed by any guides on this repository are designed to persist data under `/data` using `hostPath`. But removing PVs by `kubectl delete pv` command does not remove actual data in the host filesystem. So if you want to remove the data in the PVs, you need to remove it manually.

These are the example commands to remove the data in the PVs for AWX.

<!-- shell: pv: remove -->
```bash
sudo rm -rf /data/projects
sudo rm -rf /data/postgres-15
```

### Uninstall K3s

K3s comes with a handy uninstall script. Once executed, it will perform an uninstall that includes removing all resources deployed on Kubernetes.

```bash
/usr/local/bin/k3s-uninstall.sh
```
