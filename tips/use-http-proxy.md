<!-- omit in toc -->
# Configure AWX to use HTTP proxy

If you are deploying AWX in a corporate environment, you may have no direct access to the internet, but need to go through a proxy way. To achieve this, adding proxy settings to both K3s and AWX is required.

<!-- omit in toc -->
## Table of Contents

- [Add proxy settings to K3s](#add-proxy-settings-to-k3s)
- [Add proxy settings to AWX](#add-proxy-settings-to-awx)
  - [Add proxy settings to AWX by AWX UI](#add-proxy-settings-to-awx-by-awx-ui)
  - [Add Proxy Settings to AWX by AWX Operator](#add-proxy-settings-to-awx-by-awx-operator)

## Add proxy settings to K3s

The proxy settings for K3s is used to pull container images from the internet.

If you have exported the environment variables for your proxy like `HTTP_PROXY` before installation of K3s, the installation script detected them and store your environment variables to `/etc/systemd/system/k3s.service.env`.

Ensure your `/etc/systemd/system/k3s.service.env` has correct environment variables.

```bash
sudo cat /etc/systemd/system/k3s.service.env
```

If your `/etc/systemd/system/k3s.service.env` already has correct environment variables for your proxy, there is nothing to do for your K3s.

If not, export environment variables and re-run installation script,

```bash
export HTTP_PROXY=http://proxy.example.com:3128
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=127.0.0.1,localhost,.example.com
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

or, add environment variables to `/etc/systemd/system/k3s.service.env` and restart your K3s.

```bash
sudo tee -a /etc/systemd/system/k3s.service.env <<EOF
HTTP_PROXY=http://proxy.example.com:3128
HTTPS_PROXY=http://proxy.example.com:3128
NO_PROXY=127.0.0.1,localhost,.example.com
EOF
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

## Add proxy settings to AWX

The proxy settings for AWX is used to run playbooks, update inventories and projects, access Galaxy, and send notifications.

You can add proxy settings to AWX by both AWX UI and AWX Operator.

### Add proxy settings to AWX by AWX UI

Open `Settings` > `Jobs settings` page in the AWX UI and modify `Extra Environment Variables` block in JSON format.

```json
{
  "HTTPS_PROXY": "http://proxy.example.com:3128",
  "HTTP_PROXY": "http://proxy.example.com:3128",
  "NO_PROXY": "127.0.0.1,localhost,.example.com"
}
```

### Add Proxy Settings to AWX by AWX Operator

Specify your proxy settings in the section `extra_settings:` in `base/awx.yaml` like this:

```yaml
...
spec:
  ...
  extra_settings:                                   👈👈👈
    - setting: AWX_TASK_ENV['HTTP_PROXY']           👈👈👈
      value: "'http://proxy.example.com:3128'"      👈👈👈
    - setting: AWX_TASK_ENV['HTTPS_PROXY']          👈👈👈
      value: "'http://proxy.example.com:3128'"      👈👈👈
    - setting: AWX_TASK_ENV['NO_PROXY']             👈👈👈
      value: "'127.0.0.1,localhost,.example.com'"   👈👈👈
```

Note that the `value` have to be wrapped in single quotes and then double quotes as shown above.

To activate your proxy settings you need to deploy your changes using `kubectl` like this:

```bash
kubectl apply -k base
```

Now you need to wait some time until K3S has restarted all your pods.

After logging in you can navigate to `Settings` > `Jobs settings` in the AWX UI and find your proxy settings in the `Extra Environment Variables` block. But note that you will not be able to edit the setting via Web UI once the configuration has passed through AWX Operator. If you want to modify your configuration, use AWX Operator again.

> [!TIP]
> The proxy settings above don't work as expected for the specific use cases such as social authentication. If you are in these cases behind proxy, try adding proxy settings by AWX Operator in different way. This can't be configured and reviewed by AWX UI.
>
> ```yaml
> ...
> spec:
>   ...
>   task_extra_env: |                             👈👈👈
>     - name: HTTP_PROXY                          👈👈👈
>       value: http://proxy.example.com:3128      👈👈👈
>     - name: HTTPS_PROXY                         👈👈👈
>       value: http://proxy.example.com:3128      👈👈👈
>     - name: NO_PROXY                            👈👈👈
>       value: 127.0.0.1,localhost,.example.com   👈👈👈
>
>   web_extra_env: |                              👈👈👈
>     - name: HTTP_PROXY                          👈👈👈
>       value: http://proxy.example.com:3128      👈👈👈
>     - name: HTTPS_PROXY                         👈👈👈
>       value: http://proxy.example.com:3128      👈👈👈
>     - name: NO_PROXY                            👈👈👈
>       value: 127.0.0.1,localhost,.example.com   👈👈👈
> ```
