<!-- omit in toc -->
# Version Mapping for AWX Operator and AWX

- [Default version mapping for AWX Operator and AWX](#default-version-mapping-for-awx-operator-and-awx)
- [Appendix: Gather bundled AWX version from AWX Operator](#appendix-gather-bundled-awx-version-from-awx-operator)

## Default version mapping for AWX Operator and AWX

The table below maps the AWX Operator versions and bundled AWX versions.

| AWX Operator | AWX     |
| ------------ | ------- |
| 2.19.1       | 24.6.1  |
| 2.19.0       | 24.6.0  |
| 2.18.0       | 24.5.0  |
| 2.17.0       | 24.4.0  |
| 2.16.1       | 24.3.1  |
| 2.16.0       | 24.3.0  |
| 2.15.0       | 24.2.0  |
| 2.14.0       | 24.1.0  |
| 2.13.1       | 24.0.0  |
| 2.13.0       | 24.0.0  |
| 2.12.2       | 23.9.0  |
| 2.12.1       | 23.8.1  |
| 2.12.0       | 23.8.0  |
| 2.11.0       | 23.7.0  |
| 2.10.0       | 23.6.0  |
| 2.9.0        | 23.5.1  |
| 2.8.0        | 23.5.0  |
| 2.7.2        | 23.4.0  |
| 2.7.1        | 23.3.1  |
| 2.7.0        | 23.3.0  |
| 2.6.0        | 23.2.0  |
| 2.5.3        | 23.1.0  |
| 2.5.2        | 23.0.0  |
| 2.5.1        | 22.7.0  |
| 2.5.0        | 22.6.0  |
| 2.4.0        | 22.5.0  |
| 2.3.0        | 22.4.0  |
| 2.2.1        | 22.3.0  |
| 2.2.0        | 22.3.0  |
| 2.1.0        | 22.2.0  |
| 2.0.1        | 22.1.0  |
| 2.0.0        | 22.0.0  |
| 1.4.0        | 21.14.0 |
| 1.3.0        | 21.13.0 |
| 1.2.0        | 21.12.0 |
| 1.1.4        | 21.11.0 |
| 1.1.3        | 21.10.2 |
| 1.1.2        | 21.10.1 |
| 1.1.1        | 21.10.0 |
| 1.1.0        | 21.9.0  |
| 1.0.0        | 21.8.0  |
| 0.30.0       | 21.7.0  |
| 0.29.0       | 21.6.0  |
| 0.28.0       | 21.5.0  |
| 0.27.0       | 21.5.0  |
| 0.26.0       | 21.4.0  |
| 0.25.0       | 21.3.0  |
| 0.24.0       | 21.3.0  |
| 0.23.0       | 21.2.0  |
| 0.22.0       | 21.1.0  |
| 0.21.0       | 21.0.0  |
| 0.20.2       | 21.0.0  |
| 0.20.1       | 21.0.0  |
| 0.20.0       | 20.1.0  |
| 0.19.0       | 20.0.1  |
| 0.18.0       | 20.0.1  |
| 0.17.0       | 20.0.0  |
| 0.16.1       | 19.5.1  |
| 0.16.0       | 19.5.1  |
| 0.15.0       | 19.5.0  |
| 0.14.0       | 19.4.0  |
| 0.13.0       | 19.3.0  |
| 0.12.0       | 19.2.2  |
| 0.11.0       | 19.2.1  |
| 0.10.0       | 19.2.0  |
| 0.9.0        | 19.1.0  |
| 0.8.0        | 19.0.0  |
| 0.7.0        | 18.0.0  |
| 0.6.0        | 15.0.0  |

In the current version of AWX Operator, [there is `image_version` parameter for AWX resource to change which image will be used](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/deploying-a-specific-version-of-awx.html), but it appears that using a version of AWX other than the one bundled with the AWX Operator [is currently not supported](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/deploying-a-specific-version-of-awx.html).

## Appendix: Gather bundled AWX version from AWX Operator

For AWX Operator 0.23.0 or later (AWX 21.2.0 or later), you can find the bundled version in the release notes on GitHub. See [release notes for AWX Operator](https://github.com/ansible/awx-operator/releases) or [release notes for AWX](https://github.com/ansible/awx/releases).

If you want to get the bundled version for older versions or by means other than accessing the release notes, try following commands.

- For AWX Operator **0.15.0 or later**

  ```bash
  # Using Docker
  docker run -it --rm --entrypoint /usr/bin/bash quay.io/ansible/awx-operator:${OPERATOR_VERSION} -c env | grep DEFAULT_AWX_VERSION

  # Using Kubernetes
  kubectl run awx-operator --restart=Never -it --rm --command /usr/bin/bash --image=quay.io/ansible/awx-operator:${OPERATOR_VERSION} -- -c "env" | grep DEFAULT_AWX_VERSION
  ```

- For AWX Operator **0.10.0 to 0.14.0**

  ```bash
  curl -sf https://raw.githubusercontent.com/ansible/awx-operator/${OPERATOR_VERSION}/roles/installer/defaults/main.yml | egrep "^image_version:"
  ```

- For AWX Operator **0.9.0**

  ```bash
  curl -sf https://raw.githubusercontent.com/ansible/awx-operator/${OPERATOR_VERSION}/roles/installer/defaults/main.yml | egrep "^tower_image_version:"
  ```

- For AWX Operator **0.7.0 and 0.8.0**

  ```bash
  curl -sf https://raw.githubusercontent.com/ansible/awx-operator/${OPERATOR_VERSION}/roles/installer/defaults/main.yml | egrep "^tower_image:"
  ```

- For AWX Operator **0.6.0**

  ```bash
  curl -sf https://raw.githubusercontent.com/ansible/awx-operator/${OPERATOR_VERSION}/roles/awx/defaults/main.yml | egrep "^tower_image:"
  ```
