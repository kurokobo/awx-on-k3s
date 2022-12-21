---
name: ❔ Question - Something doesn't work as guided
about: Request for help if you have trouble during following the guide
title: ''
labels: question
assignees: ''

---

## Environment
<!-- K3s version can be checked by `k3s --version` command. -->

- OS: CentOS X.Y, RHEL X.Y, Ubuntu X.Y, Debian X.Y, ...
- Kubernetes/K3s: X.Y.Z
- AWX Operator: X.Y.Z

## Description
<!-- Describe the problem here. -->
<!-- Refer to troubleshooting guide first before creating issue. -->
<!-- https://github.com/kurokobo/awx-on-k3s/blob/main/tips/troubleshooting.md -->

Blah blah blah ...

## Step to Reproduce
<!-- Describe the steps to reproduce the problem or the operation you performed. -->

1. Deploy ...
2. Invoke `kubectl ...`
3. ...
4. ...

## Logs
<!-- Copy and paste the logs, command output, and error messages you've faced. -->
<!-- Refer to troubleshooting guide to obtain logs from Operator and AWX. -->
<!-- https://github.com/kurokobo/awx-on-k3s/blob/main/tips/troubleshooting.md -->

```bash
$ kubectl ...
...
```

## Files
<!-- If the actual files that used to deploy can be provided, copy & paste or attach them. -->

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
...
```
