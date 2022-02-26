# Examples of Ansible Builder

This repository includes ready-to-use files as an example to use Ansible Builder.

- [ansible/ansible-builder](https://github.com/ansible/ansible-builder)
- [Introduction — ansible-builder documentation](https://ansible-builder.readthedocs.io/en/latest/)

## Environment in This Example

- CentOS Stream 8 (Minimal)
- Python 3.9
- Docker 20.10.12

## Install

```bash
python3 -m pip install ansible-builder
```

## Prepare Required Files

`execution-environment.yml` is required file to build Execution Environment.

The base image can be chosen from the tags from [http://quay.io/ansible/ansible-runner](https://quay.io/repository/ansible/ansible-runner?tab=tags).

## Build Execution Environment

`ansible-builder build` command builds Execution Environment as a container image according to the definition in `execution-environment.yml`.

```bash
cd builder
ansible-builder build --tag registry.example.com/ansible/ee:2.12-custom --container-runtime docker --verbosity 3
```

```bash
$ ansible-builder build --tag registry.example.com/ansible/ee:2.12-custom --container-runtime docker --verbosity 3
Ansible Builder is building your execution environment image, "registry.example.com/ansible/ee:2.12-custom".
File context/_build/requirements.yml will be created.
File context/_build/requirements.txt will be created.
File context/_build/bindep.txt will be created.
File context/_build/ansible.cfg will be created.
Rewriting Containerfile to capture collection requirements
Running command:
  docker build -f context/Dockerfile -t registry.example.com/ansible/ee:2.12-custom context
Sending build context to Docker daemon   7.68kB
Step 1/25 : ARG EE_BASE_IMAGE=quay.io/ansible/ansible-runner:stable-2.12-latest
Step 2/25 : ARG EE_BUILDER_IMAGE=quay.io/ansible/ansible-builder:latest
Step 3/25 : FROM $EE_BASE_IMAGE as galaxy
...
Removing intermediate container cb1d45eac7ba
 ---> f6c3375db22e
Successfully built f6c3375db22e
Successfully tagged registry.example.com/ansible/ee:2.12-custom

Complete! The build context can be found at: /home/********/awx-on-k3s/builder/context
```

```bash
$ docker image ls
REPOSITORY                        TAG                  IMAGE ID       CREATED         SIZE
registry.example.com/ansible/ee   2.12-custom          f6c3375db22e   4 minutes ago   748MB
```

Now you can push this image to the container registry to use as Execution Environment on AWX. If you want to deploy your own private container registry, refer [additional guide on this repository](../registry).

```bash
$ docker push registry.example.com/ansible/ee:2.12-custom
The push refers to repository [registry.example.com/ansible/ee]
...
2.12-custom: digest: sha256:043a2bd19f4fcc5bd189f0ef0e8fb4e3b436c90e984f23f7dcf0e6b3da4443e0 size: 4515
```

The `Dockerfile` generated for the build will be saved under the `context` directory.

```bash
$ cat context/Dockerfile
ARG EE_BASE_IMAGE=quay.io/ansible/ansible-runner:stable-2.12-latest
ARG EE_BUILDER_IMAGE=quay.io/ansible/ansible-builder:latest

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
USER root
...
```

## Create Context Files

`ansible-builder create` command only creates context files like `Dockerfile`, it does not invoke building the image.

```bash
$ ansible-builder create --verbosity 3
Ansible Builder is generating your execution environment build context.
File context/_build/requirements.yml will be created.
File context/_build/requirements.txt will be created.
File context/_build/bindep.txt will be created.
File context/_build/ansible.cfg will be created.
Rewriting Containerfile to capture collection requirements
Complete! The build context can be found at: /home/********/awx-on-k3s/builder/context
```
