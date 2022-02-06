<!-- omit in toc -->
# Redirect HTTP to HTTPS

Traefik, the default Ingress controller for K3s, listens for access over both HTTP and HTTPS by default, but can be configured to redirect HTTP to HTTPS.

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare Traefik](#prepare-traefik)
  - [Patch your AWX to enable HTTPS redirection](#patch-your-awx-to-enable-https-redirection)
    - [Patch your AWX using Kustomize](#patch-your-awx-using-kustomize)
    - [Patch your AWX manually](#patch-your-awx-manually)
- [Enable redirects for other services in this repository](#enable-redirects-for-other-services-in-this-repository)

## Procedure

Note that the method described in this page is applicable only when Traefik is used as Ingress Controller.

### Prepare Traefik

To enable redirection, you need to deploy a middleware with [redirectScheme](https://doc.traefik.io/traefik/v2.0/middlewares/redirectscheme/).

Since this can be referenced from other namespaces, you will create it in the `default` namespace for ease of sharing.

```bash
cat <<EOF > middleware.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
spec:
  redirectScheme:
    scheme: https
    permanent: true
EOF

kubectl -n default apply -f middleware.yaml
kubectl -n default get middleware
```

### Patch your AWX to enable HTTPS redirection

To enable redirection, the Ingress resource must have the following annotation.

```bash
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
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
  ingress_annotations: |     👈👈👈
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd     👈👈👈
```

then invoke `apply` again. Once the command has been invoked, then AWX Operator will start to modify related resources. Note that the AWX Pod will be recreated, so AWX will be temporarily disabled.

```bash
$ kubectl apply -k base
namespace/awx unchanged
secret/awx-admin-password unchanged
secret/awx-postgres-configuration unchanged
secret/awx-secret-tls configured
persistentvolume/awx-postgres-volume unchanged
persistentvolume/awx-projects-volume unchanged
persistentvolumeclaim/awx-projects-claim unchanged
awx.awx.ansible.com/awx configured     👈👈👈
```

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager --tail=100
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0
----------
```

You can confirm that the annotations will be added to the Ingress resource.

```bash
$ kubectl -n awx get ingress awx-ingress -o=jsonpath='{.metadata.annotations}' | jq
{
  ...
  "traefik.ingress.kubernetes.io/router.middlewares": "default-redirect@kubernetescrd"
}
```

Now the redirection should be working. Go to `http://awx.example.com/` or the hostname you specified and make sure you are redirected to `https://`.

#### Patch your AWX manually

You can patch the AWX resource with the following command. Once the command has been invoked, then AWX Operator will start to modify related resources. Note that the AWX Pod will be recreated, so AWX will be temporarily disabled.

```bash
kubectl -n awx patch awx awx --type=merge \
 -p '{"spec": {"ingress_annotations": "traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd"}}'
```

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager --tail=100
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0
----------
```

You can confirm that the annotations will be added to the Ingress resource.

```bash
$ kubectl -n awx get ingress awx-ingress -o=jsonpath='{.metadata.annotations}' | jq
{
  ...
  "traefik.ingress.kubernetes.io/router.middlewares": "default-redirect@kubernetescrd"
}
```

Now the redirection should be working. Go to `http://awx.example.com/` and make sure you are redirected to `https://awx.example.com/`.

## Enable redirects for other services in this repository

You can also enable HTTPS redirection for [Git repository](../git/), [container registry](../registry) and [Galaxy NG](../galaxy), which are included in this repository, by configuring Ingress as well.

Add the following lines to the `ingress.yaml` for each resource,

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <resouce name>
  annotations:     👈👈👈
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd     👈👈👈
...
```

and `apply` them by Kustomize as you did the first time you deployed it.

```bash
kubectl apply -k <path>
```

Or you can also patch Ingress resources directly.

```bash
kubectl -n <namespace> patch ingress <resouce name> --type=merge \
 -p '{"metadata": {"annotations": {"traefik.ingress.kubernetes.io/router.middlewares": "default-redirect@kubernetescrd"}}}'
```
