<!-- omit in toc -->
# [Experimental] Deploy Galaxy NG

The guide to deploy your private Galaxy NG a.k.a. upstream version of Ansible Automation Hub on K3s.

In this guide, [Galaxy Operator](https://github.com/ansible/galaxy-operator) is used to deploy Galaxy NG.

- [Galaxy-Operator](https://ansible.readthedocs.io/projects/galaxy-operator/en/latest/)
- [ansible/galaxy-operator](https://github.com/ansible/galaxy-operator)
- [ansible/galaxy-ng](https://github.com/ansible/galaxy_ng)

> [!NOTE]
> Refer to [the official installation guide](https://ansible.readthedocs.io/projects/galaxy-ng/en/latest/usage_guide/installation/) if you want to deploy Galaxy NG on Docker or Podman.

<!-- omit in toc -->
## Table of Contents

- [Environment](#environment)
- [Deployment Instruction](#deployment-instruction)
  - [Install Galaxy Operator](#install-galaxy-operator)
  - [Prepare required files to deploy Galaxy NG](#prepare-required-files-to-deploy-galaxy-ng)
  - [Deploy Galaxy NG](#deploy-galaxy-ng)
- [Configuration and Usage](#configuration-and-usage)
  - [Sync Collections with Public Galaxy](#sync-collections-with-public-galaxy)
  - [Publish Your Own Collections to Galaxy NG](#publish-your-own-collections-to-galaxy-ng)
  - [Install Collections Locally from Galaxy NG](#install-collections-locally-from-galaxy-ng)
  - [Push/Pull Container Image to/from Galaxy NG](#pushpull-container-image-tofrom-galaxy-ng)
- [Use with AWX](#use-with-awx)
  - [Use Collections on Galaxy NG through AWX](#use-collections-on-galaxy-ng-through-awx)
  - [Use Execution Environment on Galaxy NG through AWX](#use-execution-environment-on-galaxy-ng-through-awx)

## Environment

> [!WARNING]
> Galaxy NG deployed with this procedure will not be used as container registry due to [a known issue](https://github.com/ansible/galaxy-operator/issues/74). If you want to use fully working Galaxy NG, follow [the old version of this guide that uses Pulp Operator instead](https://github.com/kurokobo/awx-on-k3s/tree/2.12.1/galaxy#deploy-on-kubernetes-pulp-operator).

- Galaxy Operator 2024.5.8
- Galaxy NG
  - Service: 9d2f8ce1
  - UI: 6116e760

## Deployment Instruction

### Install Galaxy Operator

Clone this repository and change directory.

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
```

Then invoke `kubectl apply -k galaxy/operator` to deploy Galaxy Operator.

<!-- shell: operator: deploy -->
```bash
kubectl apply -k galaxy/operator
```

The Galaxy Operator will be deployed to the namespace `galaxy`.

<!-- shell: operator: get resources -->
```bash
$ kubectl -n galaxy get all
NAME                                                      READY   STATUS    RESTARTS   AGE
pod/galaxy-operator-controller-manager-69bdb6886d-jz62p   2/2     Running   0          31s

NAME                                                         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/galaxy-operator-controller-manager-metrics-service   ClusterIP   10.43.73.43   <none>        8443/TCP   31s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/galaxy-operator-controller-manager   1/1     1            1           31s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/galaxy-operator-controller-manager-69bdb6886d   1         1         1       31s
```

### Prepare required files to deploy Galaxy NG

Generate a Self-Signed Certificate and key pair. Note that IP address can't be specified.

<!-- shell: instance: generate certificates -->
```bash
GALAXY_HOST="galaxy.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./galaxy/galaxy/tls.crt -keyout ./galaxy/galaxy/tls.key -subj "/CN=${GALAXY_HOST}/O=${GALAXY_HOST}" -addext "subjectAltName = DNS:${GALAXY_HOST}"
```

Modify `hostname` in `galaxy/galaxy/galaxy.yaml`.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: galaxy-secret-tls
  hostname: galaxy.example.com   👈👈👈
  ...
```

Modify two `password`s in `galaxy/galaxy/kustomization.yaml`.

```yaml
...
  - name: galaxy-postgres-configuration
    type: Opaque
    literals:
      - host=galaxy-postgres-15
      - port=5432
      - database=galaxy
      - username=galaxy
      - password=Galaxy123!   👈👈👈
      - sslmode=prefer
      - type=managed

  - name: galaxy-admin-password
    type: Opaque
    literals:
      - password=Galaxy123!   👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `galaxy/galaxy/pv.yaml`.

<!-- shell: instance: create directories -->
```bash
sudo mkdir -p /data/galaxy/postgres-15
sudo mkdir -p /data/galaxy/file
sudo chown 1000:0 /data/galaxy/file
sudo mkdir -p /data/galaxy/redis
```

### Deploy Galaxy NG

Deploy Galaxy NG, this takes few minutes to complete.

<!-- shell: instance: deploy -->
```bash
kubectl apply -k galaxy/galaxy
```

To monitor the progress of the deployment, check the logs of `deployments/galaxy-operator-controller-manager`:

<!-- shell: instance: gather logs -->
```bash
kubectl -n galaxy logs -f deployments/galaxy-operator-controller-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n galaxy logs -f deployments/galaxy-operator-controller-manager
...
----- Ansible Task Status Event StdOut (galaxy.ansible.com/v1beta1, Kind=Galaxy, galaxy/galaxy) -----
PLAY RECAP *********************************************************************
localhost                  : ok=140  changed=25   unreachable=0    failed=0    skipped=87   rescued=0    ignored=0
```

Required objects has been deployed next to Pulp Operator in `galaxy` namespace.

<!-- shell: instance: get resources -->
```bash
$ kubectl -n galaxy get galaxy,all,ingress,secrets
NAME                               AGE
galaxy.galaxy.ansible.com/galaxy   4m44s

NAME                                                      READY   STATUS    RESTARTS   AGE
pod/galaxy-operator-controller-manager-69bdb6886d-klh28   2/2     Running   0          4m29s
pod/galaxy-postgres-15-0                                  1/1     Running   0          3m45s
pod/galaxy-redis-994cbcbff-46m95                          1/1     Running   0          3m26s
pod/galaxy-worker-5ffd987855-g56rt                        1/1     Running   0          3m30s
pod/galaxy-api-75d6bf46b8-lbt4z                           1/1     Running   0          3m19s
pod/galaxy-content-6d7dd695c5-dsjkq                       1/1     Running   0          3m34s
pod/galaxy-web-7f75d4c888-bg5pt                           1/1     Running   0          3m40s

NAME                                                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/galaxy-operator-controller-manager-metrics-service   ClusterIP   10.43.73.43    <none>        8443/TCP    4m29s
service/galaxy-postgres-15                                   ClusterIP   None           <none>        5432/TCP    3m45s
service/galaxy-web-svc                                       ClusterIP   10.43.114.49   <none>        24880/TCP   3m39s
service/galaxy-content-svc                                   ClusterIP   10.43.9.181    <none>        24816/TCP   3m37s
service/galaxy-redis-svc                                     ClusterIP   10.43.20.127   <none>        6379/TCP    3m27s
service/galaxy-api-svc                                       ClusterIP   10.43.76.66    <none>        8000/TCP    3m24s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/galaxy-operator-controller-manager   1/1     1            1           4m29s
deployment.apps/galaxy-redis                         1/1     1            1           3m26s
deployment.apps/galaxy-worker                        1/1     1            1           3m30s
deployment.apps/galaxy-api                           1/1     1            1           3m19s
deployment.apps/galaxy-content                       1/1     1            1           3m34s
deployment.apps/galaxy-web                           1/1     1            1           3m40s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/galaxy-operator-controller-manager-69bdb6886d   1         1         1       4m29s
replicaset.apps/galaxy-redis-994cbcbff                          1         1         1       3m26s
replicaset.apps/galaxy-worker-5ffd987855                        1         1         1       3m30s
replicaset.apps/galaxy-api-75d6bf46b8                           1         1         1       3m19s
replicaset.apps/galaxy-content-6d7dd695c5                       1         1         1       3m34s
replicaset.apps/galaxy-web-7f75d4c888                           1         1         1       3m40s

NAME                                  READY   AGE
statefulset.apps/galaxy-postgres-15   1/1     3m45s

NAME                                       CLASS     HOSTS                ADDRESS         PORTS     AGE
ingress.networking.k8s.io/galaxy-ingress   traefik   galaxy.example.com   192.168.0.221   80, 443   2m9s

NAME                                   TYPE                DATA   AGE
secret/galaxy-admin-password           Opaque              1      4m44s
secret/galaxy-postgres-configuration   Opaque              7      4m44s
secret/galaxy-secret-tls               kubernetes.io/tls   2      4m44s
secret/redhat-operators-pull-secret    Opaque              1      3m56s
secret/galaxy-db-fields-encryption     Opaque              1      3m48s
secret/galaxy-server                   Opaque              1      3m25s
secret/galaxy-container-auth           Opaque              2      3m22s
```

Now your AWX is available at `https://galaxy.example.com/` or the hostname you specified. You can log in to the GUI by user `admin` with password you specified in `pulp/kustomization.yaml`.

## Configuration and Usage

Basic configuration and usage of Galaxy NG.

### Sync Collections with Public Galaxy

Create a list of Collections to be synchronized as YAML file.

```yaml
---
collections:
  - name: community.general
    source: https://galaxy.ansible.com
    version: ">=8.0.0"
  - name: kubernetes.core
    source: https://galaxy.ansible.com
    version: "3.0.0"
  - name: community.vmware
    source: https://galaxy.ansible.com
    version: ">=3.10.0,<4.0.0"
  - name: awx.awx
    source: https://galaxy.ansible.com
    version: ">=23.0.0"
  - name: ansible.utils
    source: https://galaxy.ansible.com
    version: ">=2.12.0"
```

In Galaxy NG, open `Collections` > `Remotes` > `community` and click `Edit`.

Upload your YAML file in `YAML requirements` and `Save`.

Open `Collections` > `Repositories` > `community` > `Sync`, then `Sync` and wait to complete.

### Publish Your Own Collections to Galaxy NG

Create a new minimal collection with a minimal plugin, minimal module, and minimal role.

```bash
# Create skeleton collection
ansible-galaxy collection init demo.collection

# Create meta file
mkdir -p demo/collection/meta
cat <<EOF > demo/collection/meta/runtime.yml
---
requires_ansible: ">=2.9"
EOF

# Create new Plugin
mkdir -p demo/collection/plugins/vars
cat <<EOF > demo/collection/plugins/vars/sample_vars.py
DOCUMENTATION = '''
---
vars: sample_vars
short_description: Add a fixed variable named sample_var
version_added: "1.0.0"
description: Just add a fixed variable with name sample_var.
'''
from ansible.plugins.vars import BaseVarsPlugin
class VarsModule(BaseVarsPlugin):
    def get_vars(self, loader, path, entities):
        return {"sample_var": "This is sample variable"}
EOF

# Create new Module
mkdir -p demo/collection/plugins/modules
cat <<EOF > demo/collection/plugins/modules/sample_module.py
DOCUMENTATION = '''
---
module: sample_module
short_description: This is my test module
version_added: "1.0.0"
description: This is my longer description explaining my test module.
'''
from ansible.module_utils.basic import AnsibleModule
if __name__ == '__main__':
    result = dict(changed=False, message='Hello from Module')
    module = AnsibleModule(argument_spec={})
    module.exit_json(**result)
EOF

# Create new Role
cd demo/collection/roles
ansible-galaxy init sample_role
cat <<EOF > sample_role/tasks/main.yml
---
- name: Hello
  debug:
    msg: "World"
EOF

# Create CHANGELOG.rst
# You can install antsibull-changelog by "pip install antsibull-changelog"
cd ../
antsibull-changelog init .
cat <<EOF > changelogs/fragments/summary.yml
release_summary: |
  Demo Collection 1.0.0
EOF
antsibull-changelog release

# Build tarball
ansible-galaxy collection build
```

Then create `demo` namespace on Galaxy NG, and publish your collection.

Note that you can get appropriate URL for `--server` from `Collections` > `Namespaces` > `View collections` for `demo` namespace > `CLI configuration` per collections. Your token is available at `Collections` > `API token` > `Load token`.

```bash
ansible-galaxy collection publish \
  demo-collection-1.0.0.tar.gz \
  --server https://galaxy.example.com/api/galaxy/ \
  --token d926e******************************3e996 \
  -c
```

Once the command succeeded, your collection is stayed at `staging` repository. Approval by super user on `Collections` > `Approval` page is required to move your collection to `published` repository.

Optionally, this approval process can be disabled by adding `galaxy_require_content_approval: "False"` in your `settings.py`.

### Install Collections Locally from Galaxy NG

Modify your `ansible.cfg` to specify which Galaxy Instance will be used in which order. Note that you can get appropriate configuration from `Collections` > `Repositories` > repository name (`community` or `published` for example) > `Copy CLI configuration` per repositories. Your token is available at `Collections` > `API token`.

```init
[galaxy]
server_list = published_repo, community_repo

[galaxy_server.published_repo]
url=https://galaxy.example.com/api/galaxy/
token=d926e******************************3e996

[galaxy_server.community_repo]
url=https://galaxy.example.com/api/galaxy/content/community/
token=d926e******************************3e996
```

Then simply `collection install` command can be used to install collections from your Galaxy NG.

```bash
ansible-galaxy collection install community.vmware -c
ansible-galaxy collection install demo.collection -c
```

You can test your collection by invoking minimal playbook.

```bash
cat <<EOF > site.yml
- hosts: localhost
  roles:
    - demo.collection.sample_role
  tasks:
    - demo.collection.sample_module:
      register: result
    - debug:
        var: result
EOF

ansible-playbook site.yml
```

### Push/Pull Container Image to/from Galaxy NG

Add `galaxy.example.com` as an Insecure Registry.

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["galaxy.example.com"]
}
EOF
sudo systemctl restart docker
```

Then simply `login`, `tag` and `push`.

```bash
docker login galaxy.example.com
docker tag registry.example.com/ansible/ee:2.15-custom galaxy.example.com/demo/ee:2.15-custom
docker push galaxy.example.com/demo/ee:2.15-custom
```

## Use with AWX

Use your Galaxy NG with AWX.

### Use Collections on Galaxy NG through AWX

To use your Collections on your Galaxy NG through AWX, some tasks are required before creating Project.

1. Store your **Token** and **URL** for specific **Organization** in AWX
   - Add credential with type `Ansible Galaxy/Automation Hub API Token` with your Token and Galaxy Server URL.
   - You can get appropriate URL from `Collections` > `Repositories` > repository name (`community` or `published` for example) > `Copy CLI configuration` on Galaxy NG.
   - Your token is available at `Collections` > `API token` on Galaxy NG.
2. Enable your credential in **Organization**
   - In `Edit` screen for `Organization` that will use your Galaxy NG, enable your credential in `Galaxy Credentials`.
   - You can change the order of credentials to set precedence for the sync and lookup of the content.
3. Ignore SSL Certificate Verification
   - Enable `Ignore Ansible Galaxy SSL Certificate Verification` in `Settings` > `Jobs` > `Jobs settings`

Then create files to test collection.

```bash
mkdir collection-demo
cd collection-demo

mkdir collections
cat <<EOF > collections/requirements.yml
---
collections:
  - name: demo.collection
    # If you want to ignore search order and get collections from a specific Galaxy Server,
    # you can specify the source URL here
    #source: https://galaxy.example.com/api/galaxy/content/published/
EOF

cat <<EOF > site.yml
---
- hosts: localhost
  roles:
    - demo.collection.sample_role
  tasks:
    - demo.collection.sample_module:
      register: result
    - debug:
        var: result
EOF
```

Push files in `collection-demo` to your SCM, and create new Project in AWX in standard way. Once Sync has been invoked, your collection will be installed through your Galaxy NG.

### Use Execution Environment on Galaxy NG through AWX

To use your Execution Environment on your Galaxy NG through AWX, Kubernetes have to be able to pull images from your Galaxy NG.

If the endpoint of the Galaxy NG you created is HTTPS with a Self-Signed Certificate, you need to disable SSL validation for the registry.

To achieve this, create a `registries.yaml`, and then restart K3s.

```bash
sudo tee /etc/rancher/k3s/registries.yaml <<EOF
configs:
  galaxy.example.com:
    tls:
      insecure_skip_verify: true
EOF

# The K3s service can be safely restarted without affecting the running resources
sudo systemctl restart k3s
```

If this is successfully applied, you can check the applied configuration in the `config.registry` section of the following command.

```bash
sudo $(which k3s) crictl info

# With jq
sudo $(which k3s) crictl info | jq .config.registry
```

Now you can use Execution Environment on Galaxy NG through AWX as following.

1. Push your Execution Environment to your Galaxy NG (as described above)
2. Create Credential with `Container Registry` type on AWX for your Galaxy NG
3. Register new Execution Environment on AWX
4. Specify it as Execution Environment for the Job Template, Project Default, or Global Default.

Once you start the Job Template, `imagePullSecrets` will be created from Credentials and assigned to the Pod, the image will be pulled, and the playbook will run on the Execution Environment.
