<!-- omit in toc -->
# Deploy Private Container Registry

Deploying your private container registry on your K3s to use with AWX.

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare required files](#prepare-required-files)
  - [Deploy Private Container Registry](#deploy-private-container-registry-1)
- [Quick Testing](#quick-testing)
  - [Testing with Docker](#testing-with-docker)
  - [Digging into the Registry](#digging-into-the-registry)
- [Use as Private Container Registry for AWX or K3s](#use-as-private-container-registry-for-awx-or-k3s)
  - [Procedure](#procedure-1)
  - [Testing](#testing)

## Procedure

### Prepare required files

Generate a Self-Signed Certificate. Note that IP address can't be specified.

<!-- shell: instance: generate certificates -->
```bash
REGISTRY_HOST="registry.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./registry/tls.crt -keyout ./registry/tls.key -subj "/CN=${REGISTRY_HOST}/O=${REGISTRY_HOST}" -addext "subjectAltName = DNS:${REGISTRY_HOST}"
```

Modify `hosts` and `host` in `registry/ingress.yaml`.

```yaml
...
    - hosts:
        - registry.example.com     👈👈👈
      secretName: registry-secret-tls
  rules:
    - host: registry.example.com   👈👈👈
...
```

Generate `htpasswd` string by your own username and password to use as the user for the container registry.

```bash
$ kubectl run htpasswd -it --restart=Never --image httpd:2.4 --rm -- htpasswd -nbB reguser Registry123!
reguser:$2y$05$VLMvcWCPF0VUuHi0BXBz7eoXGZ6KRl1gataiqTXz4DdSVIXGloKiq

pod "htpasswd" deleted
```

Replace `htpasswd` in `registry/configmap.yaml` with your own `htpasswd` string that generated above.

```yaml
...
  htpasswd: |-
    reguser:$2y$05$VLMvcWCPF0VUuHi0BXBz7eoXGZ6KRl1gataiqTXz4DdSVIXGloKiq   👈👈👈
```

Prepare directories for Persistent Volumes defined in `registry/pv.yaml`.

<!-- shell: instance: create directories -->
```bash
sudo mkdir -p /data/registry
```

### Deploy Private Container Registry

Deploy private container registry.

<!-- shell: instance: deploy -->
```bash
kubectl apply -k registry
```

Required resources has been deployed in `registry` namespace.

<!-- shell: instance: get resources -->
```bash
$ kubectl -n registry get all,ingress
NAME                            READY   STATUS    RESTARTS   AGE
pod/registry-7457f6c64b-sxqfp   1/1     Running   0          9s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/registry-service   ClusterIP   10.43.15.228   <none>        5000/TCP   9s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/registry   1/1     1            1           9s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/registry-7457f6c64b   1         1         1       9s

NAME                                         CLASS    HOSTS                  ADDRESS         PORTS     AGE
ingress.networking.k8s.io/registry-ingress   <none>   registry.example.com   192.168.0.221   80, 443
```

Now your container registry can be used through `registry.example.com` or the hostname you specified.

## Quick Testing

### Testing with Docker

Add your registry as an insecure registry and restart Docker daemon.

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["registry.example.com"]
}
EOF
sudo systemctl restart docker
```

Log in to your container registry.

```bash
$ docker login registry.example.com
Username: reguser
Password: 
WARNING! Your password will be stored unencrypted in /home/********/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

Now you can push/pull the image to/from your container registry.

```bash
# Pull from docker.io
docker pull docker.io/docker/whalesay:latest

# Tag as your own image on your private container registry
docker tag docker.io/docker/whalesay:latest registry.example.com/reguser/whalesay:latest

# Push your own image to your private container registry
docker push registry.example.com/reguser/whalesay:latest
```

```bash
# Remove local images
docker image rm docker.io/docker/whalesay:latest
docker image rm registry.example.com/reguser/whalesay:latest

# Pull the image from your private container registry
docker pull registry.example.com/reguser/whalesay:latest
```

```bash
$ docker run -it --rm registry.example.com/reguser/whalesay:latest cowsay hoge
 ______ 
< hoge >
 ------ 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/   
```

### Digging into the Registry

There is an useful CLI tool called [**reg**](https://github.com/genuinetools/reg) to dig into the container registry.

```bash
# Install reg
sudo curl -fSL https://github.com/genuinetools/reg/releases/download/v0.16.1/reg-linux-amd64 -o /usr/local/bin/reg
sudo chmod +x /usr/local/bin/reg

# List repositories and tags in the container registry
reg ls -k registry.example.com
reg tags -k registry.example.com/reguser/whalesay

# Delete tags on the registry
reg rm -k registry.example.com/reguser/whalesay:latest
```

## Use as Private Container Registry for AWX or K3s

This registry can be used not only as a registry to store Execution Environment for AWX, but also as a private registry for K3s.

### Procedure

To achieve this, create a `registries.yaml` and restart K3s.

Note that required `imagePullSecrets` will be automatically created by AWX once you register valid Credential for your registry on AWX. Therefore, the `auth` section is only necessary if Kubernetes pulls the image directly without AWX, as in the following [Testing](#testing) procedure.

The `tls` section is required to disable SSL Verification as the endpoint is HTTPS with a Self-Signed Certificate.

<!-- shell: config: insecure registry -->
```bash
sudo tee /etc/rancher/k3s/registries.yaml <<EOF
configs:
  registry.example.com:
    auth:
      username: reguser
      password: Registry123!
    tls:
      insecure_skip_verify: true
EOF

# The K3s service can be safely restarted without affecting the running resources
sudo systemctl restart k3s
```

If this is successfully applied, you can check the applied configuration in the `config.registry` section of the following command.

<!-- shell: config: dump config -->
```bash
sudo $(which k3s) crictl info

# With jq
sudo $(which k3s) crictl info | jq .config.registry
```

If you want Kubernetes to be able to pull images directly from this private registry, alternatively you can also manually create `imagePullSecrets` for the Pod instead of writing your credentials in `auth` in `registries.yaml`. [Another guide about rate limiting on Docker Hub](../tips/dockerhub-rate-limit.md) explains how to use `ImagePullSecrets`.

### Testing

You can launch your Pod using an image from a private repository that requires authentication.

```bash
$ kubectl run whalesay -it --restart=Never --image registry.example.com/reguser/whalesay:latest --rm -- cowsay hoge
 ______ 
< hoge >
 ------ 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/   
pod "whalesay" deleted
```
