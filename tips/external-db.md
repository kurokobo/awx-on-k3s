<!-- omit in toc -->
# Deploy AWX using external PostgreSQL database

The guide to deploy AWX using your existing external PostgreSQL database. The overview of the procedure is almost the same as [the main guide](../), but a few additional files need to be modified.

<!-- omit in toc -->
## Table of Contents

- [Prepare PostgreSQL](#prepare-postgresql)
- [Prepare required files](#prepare-required-files)
  - [Modify `base/awx.yaml`](#modify-baseawxyaml)
  - [Modify `base/kustomization.yaml`](#modify-basekustomizationyaml)
  - [Modify `base/pv.yaml`](#modify-basepvyaml)
  - [Prepare directories](#prepare-directories)
- [The next steps](#the-next-steps)

## Prepare PostgreSQL

Prepare your PostgreSQL.

Here, for the simplest example, I prepared it on another host (named `postgres.example.internal`) using Docker Compose.

```yaml
version: "3"

services:
  postgres:
    image: postgres:13
    ports:
      - 5432:5432
    restart: always
    environment:
      - POSTGRES_DB=awx
      - POSTGRES_USER=awx
      - POSTGRES_PASSWORD=SecurePasswordForMyExternalPostgreSQLForAWX123!
    volumes:
      - "postgres-data:/var/lib/postgresql/data"

volumes:
  postgres-data:
```

## Prepare required files

In addition to the steps in [the main guide (`README.md`)](../), here are a few additional files that need to be modified before you deploy AWX.

### Modify `base/awx.yaml`

Comment out following four lines which are unnecessary settings in `base/awx.yaml`.

```yaml
...
spec:
  ...
  postgres_configuration_secret: awx-postgres-configuration

  # postgres_storage_class: awx-postgres-volume   👈👈👈
  # postgres_storage_requirements:                👈👈👈
  #   requests:                                   👈👈👈
  #     storage: 8Gi                              👈👈👈

  projects_persistence: true
  projects_existing_claim: awx-projects-claim
  ...
```

### Modify `base/kustomization.yaml`

Replace and modify following lines under `awx-postgres-configuration` in `base/kustomization.yaml` to suit your environment.

```yaml
secretGenerator:
  ...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=postgres.example.internal                             👈👈👈
      - port=5432                                                  👈👈👈
      - database=awx                                               👈👈👈
      - username=awx                                               👈👈👈
      - password=SecurePasswordForMyExternalPostgreSQLForAWX123!   👈👈👈
      - sslmode=prefer                                             👈👈👈
      - type=unmanaged                                             👈👈👈
```

Note that the `type=unmanaged` is the important configuration to use external database.

### Modify `base/pv.yaml`

Comment out following unnecessary lines which related to `awx-postgres-13-volume` in `base/pv.yaml`.

```yaml
# ---                                       👈👈👈
# apiVersion: v1                            👈👈👈
# kind: PersistentVolume                    👈👈👈
# metadata:                                 👈👈👈
#   name: awx-postgres-13-volume            👈👈👈
# spec:                                     👈👈👈
#   accessModes:                            👈👈👈
#     - ReadWriteOnce                       👈👈👈
#   persistentVolumeReclaimPolicy: Retain   👈👈👈
#   capacity:                               👈👈👈
#     storage: 8Gi                          👈👈👈
#   storageClassName: awx-postgres-volume   👈👈👈
#   hostPath:                               👈👈👈
#     path: /data/postgres-13               👈👈👈

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: awx-projects-volume
...
```

### Prepare directories

You do not need to create the `/data/postgres-13` directory that the main guide instructs you to create.

## The next steps

The other steps are the same as in [the main guide (`README.md`)](../). Have fun!
