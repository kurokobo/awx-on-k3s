<!-- omit in toc -->
# [Experimental] Deploy Galaxy NG

Deploying your private Galaxy NG a.k.a. upstream version of Ansible Automation Hub.

**Note that the containerized implementation of Galaxy NG is not supported at this time. See the official installation guide for supported procedure.**

- [End User Installation · ansible/galaxy_ng Wiki](https://github.com/ansible/galaxy_ng/wiki/End-User-Installation)

All information on this page is for **development, testing and study purposes only.**

<!-- omit in toc -->
## Table of Contents

- [Deploy on Docker (Official Development Setup)](#deploy-on-docker-official-development-setup)
- [Deploy on Docker (All-in-One Container)](#deploy-on-docker-all-in-one-container)
  - [Procedure](#procedure)
- [Deploy on Kubernetes (All-in-One Container)](#deploy-on-kubernetes-all-in-one-container)
  - [Preparation](#preparation)
  - [Deploy Galaxy NG](#deploy-galaxy-ng)
  - [Initial Configuration](#initial-configuration)
- [Deploy on Kubernetes (Pulp Operator)](#deploy-on-kubernetes-pulp-operator)
  - [Install Pulp Operator](#install-pulp-operator)
  - [Prepare required files](#prepare-required-files)
  - [Deploy Galaxy NG](#deploy-galaxy-ng-1)
- [Configuration and Usage](#configuration-and-usage)
  - [Sync Collections with Public Galaxy](#sync-collections-with-public-galaxy)
  - [Publish Your Own Collections to Galaxy NG](#publish-your-own-collections-to-galaxy-ng)
  - [Install Collections Locally from Galaxy NG](#install-collections-locally-from-galaxy-ng)
  - [Push/Pull Container Image to/from Galaxy NG](#pushpull-container-image-tofrom-galaxy-ng)
- [Use with AWX](#use-with-awx)
  - [Use Collections on Galaxy NG through AWX](#use-collections-on-galaxy-ng-through-awx)
  - [Use Execution Environment on Galaxy NG through AWX](#use-execution-environment-on-galaxy-ng-through-awx)

## Deploy on Docker (Official Development Setup)

Official guide for Development Setup provides the procedure to run Galaxy NG on Docker.

[Development Setup · ansible/galaxy_ng Wiki](https://github.com/ansible/galaxy_ng/wiki/Development-Setup)

You can control the version of Galaxy NG by using Tags in cloned local Git repository. It takes some time to build the image, but it's not that complicated and it's a good way to try out Galaxy NG.

## Deploy on Docker (All-in-One Container)

[Pulp Project](https://pulpproject.org/) provides an all-in-one container image that contains all the necessary elements. One of the easiest ways to get a working Galaxy NG is to run it on Docker.

- [Pulp in One Container | software repository management](https://pulpproject.org/pulp-in-one-container/)
- [pulp/pulp-oci-images: Containerfiles and other assets for building Pulp 3 OCI images](https://github.com/pulp/pulp-oci-images)

Although not documented, a container image with Galaxy NG preinstalled and its source `Containerfile` are also available.

- [pulp/pulp-galaxy-ng - Docker Image | Docker Hub](https://hub.docker.com/r/pulp/pulp-galaxy-ng)
- [pulp-oci-images/images/pulp_galaxy_ng at latest · pulp/pulp-oci-images · GitHub](https://github.com/pulp/pulp-oci-images/tree/latest/images/pulp_galaxy_ng)

### Procedure

There are only three steps to make this work.

First, prepare a directory and a configuration file. You can replace the hostname as you like.

```bash
mkdir settings pulp_storage pgsql containers
cat <<EOF > settings/settings.py
CONTENT_ORIGIN='http://$(hostname):8080'
ANSIBLE_API_HOSTNAME='http://$(hostname):8080'
ANSIBLE_CONTENT_HOSTNAME='http://$(hostname):8080/pulp/content'
TOKEN_AUTH_DISABLED=True
EOF
```

Then invoke `docker run`.

```bash
docker run --detach \
           --publish 8080:80 \
           --name pulp \
           --volume "$(pwd)/settings":/etc/pulp \
           --volume "$(pwd)/pulp_storage":/var/lib/pulp \
           --volume "$(pwd)/pgsql":/var/lib/pgsql \
           --volume "$(pwd)/containers":/var/lib/containers \
           --device /dev/fuse \
           pulp/pulp-galaxy-ng:latest
```

Once it has started, reset the `admin` password.

```bash
$ docker exec -it pulp bash -c 'pulpcore-manager reset-admin-password'
Please enter new password for user "admin":
Please enter new password for user "admin" again:
Successfully set password for "admin" user.
```

Now your own Galaxy NG is available at `http://$(hostname):8080/`. You can log in to the GUI by user `admin` with password you reset.

## Deploy on Kubernetes (All-in-One Container)

In this step, we will run the above All-in-One container on Kubernetes.

### Preparation

Clone this repository and change directory.

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s/galaxy
```

Generate a Self-Signed Certificate. Note that IP address can't be specified.

```bash
GALAXY_HOST="galaxy.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./all-in-one/tls.crt -keyout ./all-in-one/tls.key -subj "/CN=${GALAXY_HOST}/O=${GALAXY_HOST}" -addext "subjectAltName = DNS:${GALAXY_HOST}"
```

Modify `hosts` and `host` in `all-in-one/ingress.yaml`.

```yaml
...
    - hosts:
        - galaxy.example.com     👈👈👈
      secretName: galaxy-secret-tls
  rules:
    - host: galaxy.example.com     👈👈👈
...
```

Modify FQDNs in `all-in-one/configmap.yaml`.

```yaml
...
data:
  settings.py: |-
    CONTENT_ORIGIN='https://galaxy.example.com'     👈👈👈
    ANSIBLE_API_HOSTNAME='https://galaxy.example.com'     👈👈👈
    ANSIBLE_CONTENT_HOSTNAME='https://galaxy.example.com/pulp/content'     👈👈👈
    TOKEN_AUTH_DISABLED=True
```

Prepare directories for Persistent Volumes defined in `all-in-one/pv.yaml`.

```bash
sudo mkdir -p /data/galaxy
```

### Deploy Galaxy NG

Deploy Galaxy NG.

```bash
kubectl apply -k all-in-one
```

Required resources has been deployed in `galaxy` namespace.

```bash
$ kubectl -n galaxy get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/galaxy-78df96fc64-l7tbq   1/1     Running   0          53s

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/galaxy-service   ClusterIP   10.43.201.53   <none>        80/TCP    6m14s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/galaxy   1/1     1            1           53s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/galaxy-78df96fc64   1         1         1       53s
```

### Initial Configuration

Once it has started, reset the `admin` password.

```bash
$ POD_NAME=$(kubectl -n galaxy get pod -l app=galaxy -o name)
$ kubectl -n galaxy exec -it $POD_NAME -- bash -c 'pulpcore-manager reset-admin-password'
Please enter new password for user "admin":
Please enter new password for user "admin" again:
Successfully set password for "admin" user.
```

Now Galaxy NG is available at `https://galaxy.example.com/` or the hostname you specified. You can log in to the GUI by user `admin` with password you reset.

## Deploy on Kubernetes (Pulp Operator)

There is a Kubernetes Operator for Pulp 3 named Pulp Operator.

- [pulp/pulp-operator: Kubernetes Operator for Pulp 3](https://github.com/pulp/pulp-operator)

This project is in alpha stage and under active development. In this guide, we use [Pulp Operator 1.0.0-beta.4](https://github.com/pulp/pulp-operator/tree/1.0.0-beta.4).

### Install Pulp Operator

Install specified version of Pulp Operator.

```bash
cd ~
git clone https://github.com/pulp/pulp-operator.git
cd pulp-operator
git checkout 1.0.0-beta.4
```

Export the name of the namespace where you want to deploy Pulp Operator as the environment variable `NAMESPACE` and run `make deploy`. The default namespace is `pulp-operator-system`. Note that `make deploy` requires `go` binary by default but you can remove this dependency by small `sed` patch.

```bash
sed -i 's/^deploy: manifests/deploy:/g' ./Makefile
export NAMESPACE=galaxy
make deploy
```

The Pulp Operator will be deployed to the namespace you specified.

```bash
$ kubectl -n galaxy get all
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/pulp-operator-controller-manager-9b8644f46-rg2rl   2/2     Running   0          21s

NAME                                                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/pulp-operator-controller-manager-metrics-service   ClusterIP   10.43.20.233   <none>        8443/TCP   21s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pulp-operator-controller-manager   1/1     1            1           21s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/pulp-operator-controller-manager-9b8644f46   1         1         1       21s
```

### Prepare required files

Clone this repository and change directory.

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s/galaxy
```

Generate a Self-Signed Certificate and key pair. Note that IP address can't be specified.

```bash
GALAXY_HOST="galaxy.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./pulp/tls.crt -keyout ./pulp/tls.key -subj "/CN=${GALAXY_HOST}/O=${GALAXY_HOST}" -addext "subjectAltName = DNS:${GALAXY_HOST}"
```

Modify `ingress_host` and `CSRF_TRUSTED_ORIGINS` in `pulp/galaxy.yaml`.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_class_name: traefik
  ingress_tls_secret: galaxy-secret-tls
  ingress_host: galaxy.example.com     👈👈👈
  ...
  pulp_settings:
    ...
    CSRF_TRUSTED_ORIGINS:
      - https://galaxy.example.com     👈👈👈
...
```

Modify two `password`s in `pulp/kustomization.yaml`.

```yaml
...
  - name: galaxy-postgres-configuration
    type: Opaque
    literals:
      - host=galaxy-database-svc
      - port=5432
      - database=galaxy
      - username=galaxy
      - password=Galaxy123!     👈👈👈
      - sslmode=prefer
      - type=managed

  - name: galaxy-admin-password
    type: Opaque
    literals:
      - password=Galaxy123!     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `pulp/pv.yaml`.

```bash
sudo mkdir -p /data/galaxy/database
sudo mkdir -p /data/galaxy/redis
sudo mkdir -p /data/galaxy/file
sudo chmod 755 /data/galaxy/database
sudo chown 700:0 /data/galaxy/file
```

### Deploy Galaxy NG

Deploy Galaxy NG.

```bash
kubectl apply -k pulp
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n galaxy logs -f deployments/pulp-operator-controller-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n galaxy logs -f deployments/pulp-operator-controller-manager
...
2006-01-02T15:04:05Z    INFO    repo_manager/status.go:148      galaxy finished execution ...
2006-01-02T15:04:05Z    INFO    repo_manager/controller.go:128  Operator tasks synced
```

Required objects has been deployed next to Pulp Operator in `galaxy` namespace.

```bash
$ kubectl -n galaxy get pulp,all,ingress,secrets
NAME                                       AGE
pulp.repo-manager.pulpproject.org/galaxy   3m15s

NAME                                                    READY   STATUS      RESTARTS   AGE
pod/pulp-operator-controller-manager-74f74c5846-kcnvx   2/2     Running     0          3m29s
pod/galaxy-redis-56445fcbbb-hxdq9                       1/1     Running     0          3m15s
pod/galaxy-database-0                                   1/1     Running     0          3m15s
pod/galaxy-pulpcore-migration-dcmhw-tjnxq               0/1     Completed   0          3m4s
pod/galaxy-reset-admin-password-pb5mv-s2nm6             0/1     Completed   0          3m4s
pod/galaxy-api-5bbf4c58cd-kzdgg                         1/1     Running     0          3m4s
pod/galaxy-content-59bbb99578-d87zz                     1/1     Running     0          3m4s
pod/galaxy-worker-66dd884cfb-v7r9z                      1/1     Running     0          3m4s
pod/galaxy-web-5bfcd9c5f9-xfgcv                         1/1     Running     0          2m24s

NAME                                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/pulp-operator-controller-manager-metrics-service   ClusterIP   10.43.168.53    <none>        8443/TCP    3m29s
service/galaxy-database-svc                                ClusterIP   None            <none>        5432/TCP    3m15s
service/galaxy-redis-svc                                   ClusterIP   10.43.192.108   <none>        6379/TCP    3m15s
service/galaxy-api-svc                                     ClusterIP   10.43.90.155    <none>        24817/TCP   3m4s
service/galaxy-content-svc                                 ClusterIP   10.43.240.255   <none>        24816/TCP   3m4s
service/galaxy-web-svc                                     ClusterIP   10.43.60.255    <none>        24880/TCP   2m23s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pulp-operator-controller-manager   1/1     1            1           3m29s
deployment.apps/galaxy-redis                       1/1     1            1           3m15s
deployment.apps/galaxy-content                     1/1     1            1           3m4s
deployment.apps/galaxy-api                         1/1     1            1           3m4s
deployment.apps/galaxy-worker                      1/1     1            1           3m4s
deployment.apps/galaxy-web                         1/1     1            1           2m24s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/pulp-operator-controller-manager-74f74c5846   1         1         1       3m29s
replicaset.apps/galaxy-redis-56445fcbbb                       1         1         1       3m15s
replicaset.apps/galaxy-api-5bbf4c58cd                         1         1         1       3m4s
replicaset.apps/galaxy-content-59bbb99578                     1         1         1       3m4s
replicaset.apps/galaxy-worker-66dd884cfb                      1         1         1       3m4s
replicaset.apps/galaxy-web-5bfcd9c5f9                         1         1         1       2m24s

NAME                               READY   AGE
statefulset.apps/galaxy-database   1/1     3m15s

NAME                                          COMPLETIONS   DURATION   AGE
job.batch/galaxy-pulpcore-migration-dcmhw     1/1           31s        3m4s
job.batch/galaxy-reset-admin-password-pb5mv   1/1           35s        3m4s

NAME                               CLASS     HOSTS                ADDRESS         PORTS     AGE
ingress.networking.k8s.io/galaxy   traefik   galaxy.example.com   192.168.0.219   80, 443   2m20s

NAME                                   TYPE                DATA   AGE
secret/galaxy-postgres-configuration   Opaque              7      3m15s
secret/galaxy-secret-tls               kubernetes.io/tls   2      3m15s
secret/galaxy-secret-key               Opaque              1      3m15s
secret/galaxy-server                   Opaque              1      3m15s
secret/galaxy-db-fields-encryption     Opaque              1      3m15s
secret/galaxy-container-auth           Opaque              2      3m15s
secret/galaxy-admin-password           Opaque              1      3m15s
```

Now your AWX is available at `https://galaxy.example.com/` or the hostname you specified. You can log in to the GUI by user `admin` with password you specified in `pulp/kustomization.yaml`.

## Configuration and Usage

Basic configuration and usage of Galaxy NG. Following section is based on Galaxy NG 4.9.

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
url=https://galaxy.example.com/api/galaxy/
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
sudo /usr/local/bin/crictl info

# With jq
sudo /usr/local/bin/crictl info | jq .config.registry
```

Now you can use Execution Environment on Galaxy NG through AWX as following.

1. Push your Execution Environment to your Galaxy NG (as described above)
2. Create Credential with `Container Registry` type on AWX for your Galaxy NG
3. Register new Execution Environment on AWX
4. Specify it as Execution Environment for the Job Template, Project Default, or Global Default.

Once you start the Job Template, `imagePullSecrets` will be created from Credentials and assigned to the Pod, the image will be pulled, and the playbook will run on the Execution Environment.
