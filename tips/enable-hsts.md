<!-- omit in toc -->
# Enable HTTP Strict Transport Security (HSTS)

Traefik, the default Ingress controller for K3s, listens for access over both HTTP and HTTPS by default, but can be configured to force users to use HTTPS.

To achieve this, this guide provides the steps to enable [HTTP Strict Transport Security (HSTS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security).

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare Traefik](#prepare-traefik)
    - [Note for restoring AWX that uses HSTS](#note-for-restoring-awx-that-uses-hsts)
  - [Patch your AWX to enable HSTS](#patch-your-awx-to-enable-hsts)
    - [Patch your AWX using Kustomize](#patch-your-awx-using-kustomize)
    - [Patch your AWX manually](#patch-your-awx-manually)
- [Enable HSTS for other services in this repository](#enable-hsts-for-other-services-in-this-repository)

## Procedure

Note that the method described in this page is applicable only when Traefik is used as Ingress Controller.

### Prepare Traefik

To enable HSTS, you need to deploy a middleware with [customized headers](https://doc.traefik.io/traefik/middlewares/http/headers/).

Since this can be referenced from other namespaces, in this guide it will be created in the `kube-system` namespace for ease of sharing.

```bash
cat <<EOF > middleware.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  namespace: kube-system
  name: hsts
spec:
  headers:
    sslRedirect: true
    forceSTSHeader: true
    stsSeconds: 63072000
    stsIncludeSubdomains: true
    stsPreload: true
EOF

kubectl -n kube-system apply -f middleware.yaml
kubectl -n kube-system get middleware.traefik.io
```

#### Note for restoring AWX that uses HSTS

When deploying the middleware, it will not be part of the [restore instructions in the restore guide](../restore/README.md).

Traefik will assume the middleware is present when the restore is complete, but you will have to reapply the middleware in the `kube-system` namespace if it is not already present, e.g. after restoring to a fresh node.

### Patch your AWX to enable HSTS

To enable HSTS for your AWX, the Ingress resource must have the following annotation.

```bash
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: kube-system-hsts@kubernetescrd
```

AWX Operator allows you to add any annotations to your Ingress by `ingress_annotations` parameter for AWX. Here are two ways to add `ingress_annotations` parameter.

- Patch your AWX using Kustomize
- Patch your AWX manually

#### Patch your AWX using Kustomize

In this repository, Kustomize was used to deploy AWX. If you still have the files you used for your first deployment, it is easy to use them again to modify AWX.

Add these two lines to your `awx.yaml`,

```yaml
spec:
  ...
  ingress_annotations: |                                                               👈👈👈
    traefik.ingress.kubernetes.io/router.middlewares: kube-system-hsts@kubernetescrd   👈👈👈
```

then invoke `apply` again. Once the command has been invoked, then AWX Operator will start to modify related resources. Note that the AWX Pod will be recreated, so AWX will be temporarily disabled.

```bash
$ kubectl apply -k base
namespace/awx unchanged
secret/awx-admin-password unchanged
secret/awx-postgres-configuration unchanged
secret/awx-secret-tls configured
persistentvolume/awx-postgres-15-volume unchanged
persistentvolume/awx-projects-volume unchanged
persistentvolumeclaim/awx-projects-claim unchanged
awx.awx.ansible.com/awx configured   👈👈👈
```

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager --tail=100
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=**   changed=0    unreachable=0    failed=0    skipped=**   rescued=0    ignored=0
```

You can confirm that the annotations will be added to the Ingress resource.

```bash
$ kubectl -n awx get ingress awx-ingress -o=jsonpath='{.metadata.annotations}' | jq
{
  ...
  "traefik.ingress.kubernetes.io/router.middlewares": "kube-system-hsts@kubernetescrd"
}
```

Now the HSTS should be working. Go to `http://awx.example.com/` (HTTP) or the hostname you specified and make sure you are redirected to `https://awx.example.com/` (HTTPS).

#### Patch your AWX manually

You can patch the AWX resource with the following command. Once the command has been invoked, then AWX Operator will start to modify related resources. Note that the AWX Pod will be recreated, so AWX will be temporarily disabled.

```bash
kubectl -n awx patch awx awx --type=merge \
 -p '{"spec": {"ingress_annotations": "traefik.ingress.kubernetes.io/router.middlewares: kube-system-hsts@kubernetescrd"}}'
```

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager --tail=100
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=**   changed=0    unreachable=0    failed=0    skipped=**   rescued=0    ignored=0
```

You can confirm that the annotations will be added to the Ingress resource.

```bash
$ kubectl -n awx get ingress awx-ingress -o=jsonpath='{.metadata.annotations}' | jq
{
  ...
  "traefik.ingress.kubernetes.io/router.middlewares": "kube-system-hsts@kubernetescrd"
}
```

Now the HSTS should be working. Go to `http://awx.example.com/` (HTTP) or the hostname you specified and make sure you are redirected to `https://awx.example.com/` (HTTPS).

## Enable HSTS for other services in this repository

You can also enable HSTS for [Git repository](../git/), [container registry](../registry) and [Galaxy NG](../galaxy), which are included in this repository, by configuring Ingress as well.

Add the following lines to the `ingress.yaml` for each resource,

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <resource name>
  annotations:                                                                         👈👈👈
    traefik.ingress.kubernetes.io/router.middlewares: kube-system-hsts@kubernetescrd   👈👈👈
...
```

and `apply` them by Kustomize as you did the first time you deployed it.

```bash
kubectl apply -k <path>
```

Or you can also patch Ingress resources directly.

```bash
kubectl -n <namespace> patch ingress <resource name> --type=merge \
 -p '{"metadata": {"annotations": {"traefik.ingress.kubernetes.io/router.middlewares": "kube-system-hsts@kubernetescrd"}}}'
```
