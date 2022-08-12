<!-- omit in toc -->
# Appendix: Back up AWX using Ansible

An example simple playbook for Ansible is also provided in this repository. This can be used with `ansible-playbook`, `ansible-runner`, and AWX. It can be also used with the scheduling feature on AWX too.

<!-- omit in toc -->
## Table of Contents

- [Requirements](#requirements)
- [Variables](#variables)
- [Preparation](#preparation)
  - [Prepare Service Account and API Token](#prepare-service-account-and-api-token)
  - [Prepare Backup Storage](#prepare-backup-storage)
- [Use with Ansible](#use-with-ansible)
- [Use with Ansible Runner](#use-with-ansible-runner)
- [Use with AWX](#use-with-awx)

## Requirements

- Ansible collections
  - [`kubernetes.core`](https://galaxy.ansible.com/kubernetes/core)
- Pip modules
  - [Refer the `kubernetes.core.k8s` module documentation](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html#requirements)

## Variables

[This example playbook](project/backup.yml) is designed to allow you to customize your backup with variables.

<!-- markdownlint-disable MD033 -->

| Variables | Description | Default |
| - | - | - |
| `awxbackup_namespace` | The name of the NameSpace where the `AWXBackup` resource will be created. | `awx` |
| `awxbackup_name` | The name of the `AWXBackup` resource. Dynamically generated using execution time by default. | `awxbackup-{{ lookup('pipe', 'date +%Y-%m-%d-%H-%M-%S') }}` |
| `awxbackup_spec` | The `spec` of the `AWXBackup` resource. Refer [official documentation](https://github.com/ansible/awx-operator/tree/0.26.0/roles/backup) for acceptable fields. | `deployment_name: awx`<br>`backup_pvc: awx-backup-claim`<br>`clean_backup_on_delete: true` |
| `awxbackup_timeout` | Time to wait for backup to complete, in seconds. If exceeded, the playbook will fail. | `600` |
| `awxbackup_keep_days` | Number of days to keep `AWXBackup` resources. `AWXBackup` resources older than this value will be deleted by this playbook. Set `0` to keep forever. | `30` |

<!-- markdownlint-enable MD033 -->

Note that this playbook enables `clean_backup_on_delete` by default that only works with AWX Operator `0.24.0` and later. This option makes that your actual backup data in your PVC is deleted at the same time the AWXBackup resource is deleted. You can disable this feature by explicitly specifying `clean_backup_on_delete: false`. Refer [the official documentation](https://github.com/ansible/awx-operator/tree/devel/roles/backup) for detail.

## Preparation

### Prepare Service Account and API Token

Create a Service Account, Role, and RoleBinding to manage the `AWXBackup` resource.

```bash
# Specify NameSpace where your AWXBackup resources will be created.
$ NAMESPACE=awx
$ kubectl -n ${NAMESPACE} apply -f rbac/sa.yaml
serviceaccount/awx-backup created
role.rbac.authorization.k8s.io/awx-backup created
rolebinding.rbac.authorization.k8s.io/awx-backup created
```

Obtain the API Token which required to authenticate the Kubernetes API. This token will be used later.

```bash
# For Kubernetes 1.24 or later:
# This will generate new token that valid for 87600 hours (10 years).
# Of course you can modify the value since "87600h" is just an example.
$ kubectl -n awx create token awx-backup --duration=87600h
eyJhbGciOiJSUzI...hcGsPI5MzmaMHQvw

# For Kubernetes 1.23 or earlier:
# Obtain and decode token from Secret that automatically generated for the Service Account.
$ SECRET=$(kubectl -n ${NAMESPACE} get sa awx-backup -o jsonpath='{.secrets[0].name}')
$ kubectl -n ${NAMESPACE} get secret ${SECRET} -o jsonpath='{.data.token}' | base64 -d
eyJhbGciOiJSUzI...hcGsPI5MzmaMHQvw
```

### Prepare Backup Storage

Since you have complete control over `spec` of `AWXBackup` via `awxbackup_spec` variables, whether or not this step is required depends on your environment. Check [the official documentation](https://github.com/ansible/awx-operator/tree/devel/roles/backup) and prepare as needed.

If your AWX was deployed by referring [the main guide on this repository](../../README.md), preparing backup storage by following [he basic backup guide](../README.md#prepare-for-backup) is good starting point.

## Use with Ansible

Export required environment variables.

```bash
export K8S_AUTH_VERIFY_SSL=no
export K8S_AUTH_HOST="https://<Your K3s Host>:6443/"
export K8S_AUTH_API_KEY="<Your API Token>"
```

```bash
# Modify variables using "-e" as needed
ansible-playbook project/backup.yml \
  -e awxbackup_spec="{'deployment_name':'awx','backup_pvc':'awx-backup-claim','clean_backup_on_delete':'true'}" \
  -e keep_days=90
```

## Use with Ansible Runner

Refer [the guide for Ansible Runner](../../runner) for the basic usage.

Modify following files as needed. Note that the EE `quay.io/ansible/awx-ee:latest` contains required modules and collections by default.

- [📝`env/settings`](env/settings): Configure your Execution Environment
- [📝`env/envvars`](env/envvars): Specify your K3s host and API Token
- [📝`env/extravars`](env/extravars): Modify variables

Then execute Ansible Runner.

```bash
ansible-runner run . -p backup.yml
```

## Use with AWX

This playbook can also be run through Job Templates on AWX. Schedules can be also set up in the Job Template to obtain periodic backups.

It is also possible to making the backup of the AWX itself where the Job Template for the backup is running on. In this case, the PostgreSQL will be dumped while the job is running, so complete logs of the job itself is not part of the backup. Therefore, after restoration, **the last backup job will be shown as failed** since the AWX can't determine the result of the job, but this can be safely ignored.

1. Add new Credential for your K3s host.
   - Select `OpenShift or Kubernetes API Bearer Token` as Credential Type.
   - Specify `https://<Your K3s Host>:6443/` as `OpenShift or Kubernetes API Endpoint`.
   - Specify your API Token as `API authentication bearer token`.
   - Toggle `Verify SSL` if needed.
2. Add new Project including the playbook.
   - You can specify this repository (`https://github.com/kurokobo/awx-on-k3s.git`) directly, but use with caution. The playbook in this repository is subject to change without notice. You can use [Tag](https://github.com/kurokobo/awx-on-k3s/tags) or [Commit](https://github.com/kurokobo/awx-on-k3s/commits/main) to fix the version to be used.
3. Add new Job Template which use the playbook.
   - Select appropriate `Execution Environment`. The default `AWX EE (latest)` (`quay.io/ansible/awx-ee:latest`) contains required collections and modules by default, so it's good for the first choice.
   - Select your `backup.yml` as `Playbook`.
   - Select your Credentials created in the above step.
   - Specify `Variables` as needed.
4. (Optional) Add new Schedules for periodic backups.
