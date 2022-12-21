---
name: ❔ Question - How to use for my use cases
about: Request for help with usage and customization for specific use cases
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
<!-- Describe your goal and you have tried or investigated here. -->
<!-- Note that @kurokobo may not respond to topics out-of-scope of this repository. -->

Blah blah blah ...

## Step to Reproduce
<!-- Describe the steps that you have tried but no luck. -->

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
