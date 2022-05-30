# Add Proxy Settings for AWX containers

If you are deploying AWX in a corporate environment, you may have no direct access to the internet, but need to go through a proxy way to achieve this is to add a section `extra_settings:` to awx.yaml. These settings will be available in the `Settings` -> `Jobs Settings` -> `Extra Environment Variables` block in the AWX UI.

## Add Proxy Settings to base/awx.yaml
You need to specify your proxy settings in the section `extra_settings:` in `base/awx.yaml` like this:

```
extra_settings: |
  - setting: AWX_TASK_ENV['HTTP_PROXY']
    value: "'http://proxy.example.com:3128'"
  - setting: AWX_TASK_ENV['HTTPS_PROXY']
    value: "'http://proxy.example.com:3128'"
  - setting: AWX_TASK_ENV['NO_PROXY']
    value: "'10.43.0.1,ansible03,localhost,.example.com,127.0.0.1'"
```

You may have to adjust your settings to match your environment.

## Deploy your changes
To activate your proxy settings you need to deploy your changes using `kubectl` like this:
```
kubectl apply -k base
```

Now you need to wait some time until K3S has restarted all your pods.

After logging in you can navigate to `Settings` -> `Jobs Settings` and find your proxy settings in the `Extra Environment Variables` block:

