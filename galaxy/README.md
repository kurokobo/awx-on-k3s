<!-- omit in toc -->
# [Experimental] Deploy Galaxy NG

Deploying your private Galaxy NG a.k.a. upstream version of Ansible Automatuin Hub.

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
  - [Patch K3s](#patch-k3s)
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
- [pulp-oci-images/pulp_galaxy_ng at latest · pulp/pulp-oci-images](https://github.com/pulp/pulp-oci-images/tree/latest/pulp_galaxy_ng)

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

Then inovoke `docker run`.

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

Once it has started, load the initial configuration file.

```bash
DATA_FIXTURE_URL="https://raw.githubusercontent.com/ansible/galaxy_ng/master/dev/automation-hub/initial_data.json"
curl $DATA_FIXTURE_URL | docker exec -i pulp bash -c "cat > /tmp/initial_data.json"
docker exec pulp bash -c "/usr/local/bin/pulpcore-manager loaddata /tmp/initial_data.json"
```

Now your own Galaxy NG is available at `http://$(hostname):8080/`. You can log in to the GUI by user `admin` with password `admin`.

## Deploy on Kubernetes (All-in-One Container)

In this step, we will run the above All-in-One container on Kubernetes.

### Preparation

Generate a Self-Signed Certificate. Note that IP address can't be specified.

```bash
GALAXY_HOST="galaxy.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./galaxy/all-in-one/tls.crt -keyout ./galaxy/all-in-one/tls.key -subj "/CN=${GALAXY_HOST}/O=${GALAXY_HOST}" -addext "subjectAltName = DNS:${GALAXY_HOST}"
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
kubectl apply -k galaxy/all-in-one
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

Once it has started, load the initial configuration file.

```bash
POD_NAME=$(kubectl -n galaxy get pod -l app=galaxy -o name)
DATA_FIXTURE_URL="https://raw.githubusercontent.com/ansible/galaxy_ng/master/dev/automation-hub/initial_data.json"
curl $DATA_FIXTURE_URL | kubectl -n galaxy exec -i $POD_NAME -- bash -c "cat > /tmp/initial_data.json"
kubectl -n galaxy exec -i $POD_NAME -- bash -c "/usr/local/bin/pulpcore-manager loaddata /tmp/initial_data.json"
```

Now Galaxy NG is available at `https://galaxy.example.com/` or the hostname you specified. You can log in to the GUI by user `admin` with password `admin`.

## Deploy on Kubernetes (Pulp Operator)

There is a Kubernetes Operator for Pulp 3 named Pulp Operator.

- [pulp/pulp-operator: Kubernetes Operator for Pulp 3](https://github.com/pulp/pulp-operator)

This project is still under active development and there is no support, however, at least the code to create a new instance seems to be implemented. In this procedure, we use [Pulp Operator 0.8.0](https://github.com/pulp/pulp-operator/tree/0.8.0)

### Patch K3s

If you use Traefik which is K3s' Ingress controller as completely default, the Pod may not be able to get the client's IP address (see [k3s-io/k3s#2997](https://github.com/k3s-io/k3s/discussions/2997) for details). In the current implementation of Pulp, this causes problems with the web UI being unreachable.

For this reason, you should fix the Traefik configuration. For a single node like doing in this repository, the following command is easy to use.

```bash
kubectl -n kube-system patch deployment traefik --patch '{"spec":{"template":{"spec":{"hostNetwork":true}}}}'
```

Then wait until your `traefik` by the following command is `1/1` `READY`.

```bash
kubectl -n kube-system get deployment traefik
```

Now your client's IP address can be passed correctly through `X-Forwarded-For` and `X-Real-Ip` headers.

### Install Pulp Operator

Install specified version of Pulp Operator.

```bash
cd ~
git clone https://github.com/pulp/pulp-operator.git
cd pulp-operator
git checkout 0.8.0
```

Export the name of the namespace where you want to deploy Pulp Operator as the environment variable `NAMESPACE` and run `make deploy`. The default namespace is `pulp-operator-system`.

```bash
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

Modify `hostname` in `pulp/galaxy.yaml`.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: galaxy-secret-tls
  hostname: galaxy.example.com     👈👈👈
...
```

Modify two `password`s in `pulp/kustomization.yaml`.

```yaml
...
  - name: galaxy-postgres-configuration
    type: Opaque
    literals:
      - host=galaxy-postgres
      - port=5432
      - database=galaxy
      - username=galaxy
      - password=Galaxy123!     👈👈👈
      - type=managed

  - name: galaxy-admin-password
    type: Opaque
    literals:
      - password=Galaxy123!     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `pulp/pv.yaml`.

```bash
sudo mkdir -p /data/galaxy/postgres
sudo mkdir -p /data/galaxy/redis
sudo mkdir -p /data/galaxy/pulp
```

### Deploy Galaxy NG

Deploy Galaxy NG.

```bash
kubectl apply -k pulp
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n galaxy logs -f deployments/pulp-operator-controller-manager -c pulp-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n galaxy logs -f deployments/pulp-operator-controller-manager -c pulp-manager
...
----- Ansible Task Status Event StdOut (pulp.pulpproject.org/v1beta1, Kind=Pulp, galaxy/galaxy) -----
PLAY RECAP *********************************************************************
localhost                  : ok=75   changed=0    unreachable=0    failed=0    skipped=62   rescued=0    ignored=0
```

Required objects has been deployed next to Pulp Operator in `galaxy` namespace.

```bash
$ kubectl -n galaxy get pulp,all,ingress,secrets
NAME                               AGE
pulp.pulp.pulpproject.org/galaxy   3m58s

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/pulp-operator-controller-manager-9b8644f46-plw72   2/2     Running   0          4m37s
pod/galaxy-redis-65c899f464-c6dhb                      1/1     Running   0          3m50s
pod/galaxy-postgres-0                                  1/1     Running   0          3m53s
pod/galaxy-content-76f68bb68f-6tn2q                    1/1     Running   0          3m37s
pod/galaxy-worker-76c986fd54-qjg28                     1/1     Running   0          3m36s
pod/galaxy-api-76985868c8-vjwks                        1/1     Running   0          3m39s
pod/galaxy-web-67cc96d8d6-644dh                        1/1     Running   0          3m48s
pod/galaxy-resource-manager-64dbf49746-jcsfs           1/1     Running   0          118s

NAME                                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/pulp-operator-controller-manager-metrics-service   ClusterIP   10.43.173.114   <none>        8443/TCP    4m37s
service/galaxy-postgres-svc                                ClusterIP   None            <none>        5432/TCP    3m53s
service/galaxy-redis-svc                                   ClusterIP   10.43.29.57     <none>        6379/TCP    3m51s
service/galaxy-web-svc                                     ClusterIP   10.43.245.204   <none>        24880/TCP   3m47s
service/galaxy-api-svc                                     ClusterIP   10.43.146.254   <none>        24817/TCP   3m40s
service/galaxy-content-svc                                 ClusterIP   10.43.214.64    <none>        24816/TCP   3m38s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pulp-operator-controller-manager   1/1     1            1           4m37s
deployment.apps/galaxy-redis                       1/1     1            1           3m50s
deployment.apps/galaxy-content                     1/1     1            1           3m37s
deployment.apps/galaxy-worker                      1/1     1            1           3m36s
deployment.apps/galaxy-api                         1/1     1            1           3m39s
deployment.apps/galaxy-web                         1/1     1            1           3m48s
deployment.apps/galaxy-resource-manager            1/1     1            1           118s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/pulp-operator-controller-manager-9b8644f46   1         1         1       4m37s
replicaset.apps/galaxy-redis-65c899f464                      1         1         1       3m50s
replicaset.apps/galaxy-content-76f68bb68f                    1         1         1       3m37s
replicaset.apps/galaxy-worker-76c986fd54                     1         1         1       3m36s
replicaset.apps/galaxy-api-76985868c8                        1         1         1       3m39s
replicaset.apps/galaxy-web-67cc96d8d6                        1         1         1       3m48s
replicaset.apps/galaxy-resource-manager-64dbf49746           1         1         1       118s

NAME                               READY   AGE
statefulset.apps/galaxy-postgres   1/1     3m53s

NAME                                       CLASS    HOSTS                ADDRESS         PORTS     AGE
ingress.networking.k8s.io/galaxy-ingress   <none>   galaxy.example.com   192.168.0.100   80, 443   3m50s

NAME                                   TYPE                                  DATA   AGE
secret/default-token-vsnlj             kubernetes.io/service-account-token   3      4m37s
secret/pulp-operator-sa-token-smzj2    kubernetes.io/service-account-token   3      4m37s
secret/galaxy-admin-password           Opaque                                1      3m58s
secret/galaxy-postgres-configuration   Opaque                                6      3m58s
secret/galaxy-secret-tls               kubernetes.io/tls                     2      3m58s
secret/galaxy-server                   Opaque                                1      3m44s
secret/galaxy-db-fields-encryption     Opaque                                1      3m41s
```

Now your AWX is available at `https://galaxy.example.com/` or the hostname you specified. You can log in to the GUI by user `admin` with password you specified in `pulp/kustomization.yaml`.

## Configuration and Usage

Basic configuration and usage of Galaxy NG.

### Sync Collections with Public Galaxy

Create a list of Collections to be syncronized as YAML file.

```yaml
---
collections:
  - name: community.general
    source: https://galaxy.ansible.com
    version: ">=3.2.0"
  - name: community.kubernetes
    source: https://galaxy.ansible.com
    version: "2.0.0"
  - name: community.vmware
    source: https://galaxy.ansible.com
    version: ">=1.10.0,<1.12.0"
  - name: awx.awx
    source: https://galaxy.ansible.com
    version: ">=19.0.0"
  - name: ansible.utils
    source: https://galaxy.ansible.com
    version: ">=2.1.0"
```

In Galaxy NG, open `Collections` > `Repository Management` > `Remote` and click `Configure` on `community`.

Select your YAML file in `YAML requirements` and `Save`, then `Sync` and wait to complete.

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

# Build tarball
cd ../
ansible-galaxy collection build
```

Then create `demo` namespace on Galaxy NG, and publish your collection.

Note that you can get appropriate URL for `--server` from `Collections` > `Namespaces` > `View collections` > `CLI Configuration` per collections. Your token is available at `Collections` > `API Token`.

```bash
ansible-galaxy collection publish \
  demo-collection-1.0.0.tar.gz \
  --server https://galaxy.example.com/api/galaxy/content/inbound-demo/ \
  --token d926e******************************3e996 \
  -c
```

Once the command succeeded, your collection is stayed at `staging` distribution. Approval by super user on `Collections` > `Approval` page is required to move your collection to `published` distribution.

Optionally, this approval process can be disabled by adding `galaxy_require_content_approval: "False"` in your `settings.py`.

### Install Collections Locally from Galaxy NG

Modify your `ansible.cfg` to speficy which Galaxy Instance will be used in which order. Note that you can get appropriate configuration from `Collections` > `Repository Management` > `Local` > `CLI configuration` per distributions. Your token is available at `Collections` > `API Token`.

```init
[galaxy]
server_list = published_repo, community_repo

[galaxy_server.published_repo]
url=https://galaxy.example.com/api/galaxy/content/published/
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
docker tag registry.example.com/ansible/ee:2.12-custom galaxy.example.com/demo/ee:2.12-custom
docker push galaxy.example.com/demo/ee:2.12-custom
```

## Use with AWX

Use your Galaxy NG with AWX.

### Use Collections on Galaxy NG through AWX

To use your Collections on your Galaxy NG through AWX, some tasks are required before creating Project.

1. Store your **Token** and **URL** for specific **Organization** in AWX
   - Add credential with type `Ansible Galaxy/Automation Hub API Token` with your Token and Galaxy Server URL.
   - You can get appropriate URL from `Collections` > `Repository Management` > `Local` > `CLI configuration` per distributions on Galaxy NG.
   - Your token is available at `Collections` > `API Token` on Galaxy NG.
1. Enable your credential in **Organization**
   - In `Edit` screen for `Organization` that will use your Galaxy NG, enable your credential in `Galaxy Credentials`.
   - You can change the order of credentials to set precedence for the sync and lookup of the content.
1. Ignore SSL Certificate Verification
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

Once you start the Job Template, `imagePullSecrets` will be created from Credentials and assinged to the Pod, the image will be pulled, and the playbook will run on the Execution Environment.
