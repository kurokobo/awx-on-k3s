<!-- omit in toc -->
# Deploy Private Git Repository using Gitea

Deploying your private Git repository using [Gitea](https://gitea.io/en-us/) to use AWX with the playbooks on SCM.

Note that this sample manifest does not include any databases, so the SQLite3 has to be selected as `Database Type` for Gitea.

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Prepare required files](#prepare-required-files)
  - [Deploy Private Git Repository](#deploy-private-git-repository)
- [Configure AWX to use Git Repository with Self-Signed Certificate](#configure-awx-to-use-git-repository-with-self-signed-certificate)

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
    - host: git.example.com   👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `git/pv.yaml`.

```bash
sudo mkdir -p /data/git
```

### Deploy Private Git Repository

Deploy private Git repository.

```bash
kubectl apply -k git
```

Required resources has been deployed in `git` namespace.

```bash
$ kubectl -n git get all,ingress
NAME                      READY   STATUS    RESTARTS   AGE
pod/git-56cc958f9-2q44j   1/1     Running   0          9s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
service/git-service   ClusterIP   10.43.134.80   <none>        3000/TCP,22/TCP   9s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/git   1/1     1            1           9s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/git-56cc958f9   1         1         1       9s

NAME                                    CLASS    HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/git-ingress   <none>   git.example.com   192.168.0.221   80, 443   9s
```

Now your Git repository is accessible through `https://git.example.com/` or the hostname you specified. Visit the URL and follow the installation wizard.

Note that this sample manifest does not include any databases, so the SQLite3 has to be selected as `Database Type` for Gitea.

| Configuration  | Recommended Value                                        |
| -------------- | -------------------------------------------------------- |
| Database Type  | `SQLite3`                                                |
| Server Domain  | `git.example.com` or the hostname you specified          |
| Gitea Base URL | `https://git.example.com/` or the hostname you specified |

## Configure AWX to use Git Repository with Self-Signed Certificate

1. Add Credentials for SCM
2. Allow Self-Signed Certificate such as this Gitea
   - Open `Settings` > `Jobs settings` in AWX
   - Press `Edit` and scroll down to `Extra Environment Variables`, then add `"GIT_SSL_NO_VERIFY": "True"` in `{}`

     ```json
     {
       "GIT_SSL_NO_VERIFY": "True"
     }
     ```

   - Press `Save`
