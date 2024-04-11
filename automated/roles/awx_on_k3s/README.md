Role Name
=========

A role to deploy AWX, Galaxy_NG, and EDA on k3s. Based on https://github.com/kurokobo/awx-on-k3s/

Requirements
------------
Creating a custom virtual environment (currently doesn't seem to work without it), and installing these packages.
```
python3 -m venv ansible
source ansible/bin/active
pip3 install -r requirements.txt
```

Python modules:
```console
cryptography
kubernetes
kubernetes-validate
```

Role Variables
--------------

|Variable Name|Default Value|Required|Type|Description|
|:---:|:---:|:---:|:---:|:---:|
|`awx_on_k3s_awx_host`|awx.example.com|true|str||
|`awx_on_k3s_awx_admin_pass`|'Password1234!'|true|str||
|`awx_on_k3s_galaxy_host`|galaxy.example.com|true|str||
|`awx_on_k3s_galaxy_admin_pass`|'Password1234!'|true|str||
|`awx_on_k3s_eda_host`|eda.example.com|true|str||
|`awx_on_k3s_eda_admin_pass`|'Password1234!'|true|str||
|`awx_on_k3s_deployment_dir`|/awx_on_k3s|true|str||
|`awx_on_k3s_deploy_k3s`|true|true|bool||
|`awx_on_k3s_disable_services`|[firewalld, nm-cloud-setup.service, nm-cloud-setup.timer]|true|list||
|`awx_on_k3s_hosts_lines`|(see defaults/main.yml)|false|list||

Dependencies
------------

```yaml
---
collections:
  - name: kubernetes.core
  - name: community.crypto
...
```

Example Playbook
----------------

```yaml
---
- name: Deploy AWX on k3s
  hosts: localhost
  connection: local
  gather_facts: true
  become: true
  vars:
    awx_on_k3s_local_hosts: true
    ansible_python_interpreter: UPDATE/TO/YOUR/VENV/PATH
  environment:
    K8S_AUTH_KUBECONFIG: /etc/rancher/k3s/k3s.yaml
  tasks:
    - name: Import awx_on_k3s role
      ansible.builtin.import_role:
        name: awx_on_k3s

...
```

License
-------

MIT

Author Information
------------------

* David Danielsson @djdanielsson
* kurokobo
