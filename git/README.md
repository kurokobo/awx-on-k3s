<!-- omit in toc -->
# Deploy Private Git Repository using Gitea

Deploying your private Git repository using [Gitea](https://gitea.io/en-us/) to use AWX with the playbooks on SCM.

Note that this sample manifest does not include any databases, so the SQLite3 has to be selected as `Database Type` for Gitea.

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare required files](#prepare-required-files)
  - [Deploy Private Git Repository](#deploy-private-git-repository)

## Procedure

### Prepare required files

Generate a Self-Signed Certificate. Note that IP address can't be specified.

```bash
GIT_HOST="git.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./git/tls.crt -keyout ./git/tls.key -subj "/CN=${GIT_HOST}/O=${GIT_HOST}" -addext "subjectAltName = DNS:${GIT_HOST}"
```

Modify `hosts` and `host` in `git/ingress.yaml`.

```yaml
...
    - hosts:
        - git.example.com     👈👈👈
      secretName: git-secret-tls
  rules:
    - host: git.example.com     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `git/pv.yaml`.

```bash
sudo mkdir -p /data/git
```

### Deploy Private Git Repository

Deploy Private Git Repository.

```bash
kubectl apply -k git
```

Required resources has been deployed in `git` namespace.

```bash
$ kubectl get all -n git
NAME                       READY   STATUS    RESTARTS   AGE
pod/git-576868dc5b-z7z55   1/1     Running   0          31s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
service/git-service   ClusterIP   10.43.192.81   <none>        3000/TCP,22/TCP   31s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/git   1/1     1            1           31s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/git-576868dc5b   1         1         1       31s
```

Now your Git repository is accesible through `https://git.example.com/` or the hostname you specified. Visit the URL and follow the installation wizard.

Note that this sample manifest does not include any databases, so the SQLite3 has to be selected as `Database Type` for Gitea.

| Configration   | Recommemded Value                                        |
| -------------- | -------------------------------------------------------- |
| Database Type  | `SQLite3`                                                |
| Gitea Base URL | `https://git.example.com/` or the hostname you specified |
