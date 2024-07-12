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
| `awxbackup_spec` | The `spec` of the `AWXBackup` resource. Refer [official documentation](https://github.com/ansible/awx-operator/tree/2.19.1/roles/backup) for acceptable fields. | `deployment_name: awx`<br>`backup_pvc: awx-backup-claim`<br>`clean_backup_on_delete: true` |
| `awxbackup_timeout` | Time to wait for backup to complete, in seconds. If exceeded, the playbook will fail. | `600` |
| `awxbackup_keep_days` | Number of days to keep `AWXBackup` resources. `AWXBackup` resources older than this value will be deleted by this playbook. Set `0` to keep forever. | `30` |

<!-- markdownlint-enable MD033 -->

Note that this playbook enables `clean_backup_on_delete` by default that only works with AWX Operator `0.24.0` and later. This option makes it so that your actual backup data in your PVC is deleted at the same time the AWXBackup resource is deleted. You can disable this feature by explicitly specifying `clean_backup_on_delete: false`. Refer to [the official documentation](https://github.com/ansible/awx-operator/tree/devel/roles/backup) for detail.

## Preparation

### Prepare Service Account and API Token

Create a Service Account, Role, and RoleBinding to manage the `AWXBackup` resource.

<!-- shell: backup: serviceaccount -->
```bash
# Specify NameSpace where your AWXBackup resources will be created.
$ NAMESPACE=awx
$ kubectl -n ${NAMESPACE} apply -f rbac/sa.yaml
serviceaccount/awx-backup created
role.rbac.authorization.k8s.io/awx-backup created
rolebinding.rbac.authorization.k8s.io/awx-backup created
```

### Prepare Backup Storage

Since you have complete control over `spec` of `AWXBackup` via `awxbackup_spec` variables, whether or not this step is required depends on your environment. Check [the official documentation](https://github.com/ansible/awx-operator/tree/devel/roles/backup) and prepare as needed.

If your AWX was deployed by referencing [the main guide on this repository](../../README.md), preparing backup storage by following [the basic backup guide](../README.md#prepare-for-backup) is a good starting point.

## Use with Ansible

Export the required environment variables.

```bash
export K8S_AUTH_HOST="https://<Your K3s Host>:6443/"
export K8S_AUTH_VERIFY_SSL=no
```

Obtain and export the API Token which is required to authenticate to the Kubernetes API.

```bash
# For Kubernetes 1.24 or later:
# This will generate new token that valid for 1 hours.
# Of course you can modify the value since "1h" is just an example.
$ export K8S_AUTH_API_KEY=$(kubectl -n awx create token awx-backup --duration=1h)

# For Kubernetes 1.23 or earlier:
# Obtain and decode token from Secret that automatically generated for the Service Account.
$ SECRET=$(kubectl -n ${NAMESPACE} get sa awx-backup -o jsonpath='{.secrets[0].name}')
$ export K8S_AUTH_API_KEY=$(kubectl -n ${NAMESPACE} get secret ${SECRET} -o jsonpath='{.data.token}' | base64 -d)
```

```bash
# Modify variables using "-e" as needed
ansible-playbook project/backup.yml \
  -e awxbackup_spec="{'deployment_name':'awx','backup_pvc':'awx-backup-claim','clean_backup_on_delete':'true'}" \
  -e keep_days=90
```

## Use with Ansible Runner

Refer to [the guide for Ansible Runner](../../runner) for the basic usage.

Modify the following files as needed. Note that the EE `quay.io/ansible/awx-ee:latest` contains required modules and collections by default.

- [📝`env/settings`](env/settings): Configure your Execution Environment
- [📝`env/envvars`](env/envvars): Specify your K3s host and API Token
- [📝`env/extravars`](env/extravars): Modify variables

Then execute Ansible Runner.

```bash
ansible-runner run . -p backup.yml
```

## Use with AWX

This playbook can also be run through Job Templates on AWX. Of course Schedules can be set up in the Job Template to obtain periodic backup.

This is the way to make the backup of the AWX itself where the Job Template for the backup is configured.

In this case, the PostgreSQL db will be dumped while the job is running, so complete logs of the job itself is not a part of the backup. Therefore, after restoration, **the last backup job will be shown as failed** the AWX can't determine the result of the job, but this can be safely ignored.

1. Add a new Container Group to make the API token usable inside the EE.
   - Enable `Customize pod specification` and put the following YAML string. `serviceAccountName` and `automountServiceAccountToken` are important to make the API token usable inside the EE.

     <!-- yaml: backup: container group -->
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       namespace: awx
     spec:
       serviceAccountName: awx-backup
       automountServiceAccountToken: true
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
     ```

2. Add a new Project including the playbook.
   - You can specify this repository (`https://github.com/kurokobo/awx-on-k3s.git`) directly, but use with caution. The playbook in this repository is subject to change without notice. You can use [Tag](https://github.com/kurokobo/awx-on-k3s/tags) or [Commit](https://github.com/kurokobo/awx-on-k3s/commits/main) to fix the version to be used.
3. Add a new Job Template which uses the playbook.
   - Select the appropriate `Inventory`. The bundled `Demo Inventory` is enough to use. If you specify your own inventory, ensure `localhost` is defined in the inventory and the following variables are enabled for `localhost`.
     - `ansible_connection: local`
     - `ansible_python_interpreter: '{{ ansible_playbook_python }}'`
   - Select the appropriate `Execution Environment`. The default `AWX EE (latest)` (`quay.io/ansible/awx-ee:latest`) contains required collections and modules by default, so it's good for the first choice.
   - Select your `backup.yml` as `Playbook`.
   - Specify `Variables` as needed.
   - Select your Container Group created in the above step as `Instance Group`.
4. (Optional) Add new Schedules for periodic backup.

If you want to make the backup of another AWX on a different namespace of different cluster, create a new Credential with `OpenShift or Kubernetes API Bearer Token` type instead of Container Group and then specify the Credential in the Job Template. To obtain the API token, refer to the [_"Use with Ansible"_](#use-with-ansible) section.
