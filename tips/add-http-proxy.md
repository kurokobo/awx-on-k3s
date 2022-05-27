# Add Proxy Settings for AWX containers

If you are deploying AWX in a corporate environment, you may have no direct access to the internet, but need to go through a proxy. to achieve this, you can add extra environment variables to the awx-web, awx-task and awx-ee containers. 
You also need to specify the `no_proxy` variable to avoid that internal calls to the K3S cluster are routed to the proxy.

## Obtain the ClusterUP
Therefore you need to obtain the `ClusterIP` by running `kubectl get all` in the default namespace:
```
[awx@ansible03 base]$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   26h
[awx@ansible03 base]$
```

In my case the IP to use is `10.43.0.1`. 

## Add Proxy Settings to base/awx.yaml
Now you need to specify your proxy settings in the stanza `task_extra_env`, `web_extra_env` and `ee_extra_env` in `base/awx.yaml` like this:
```
task_extra_env: |
  - name: HTTP_PROXY
    value: http://proxy.example.com:3128
  - name: HTTPS_PROXY
    value: http://proxy.example.com:3128
  - name: NO_PROXY
    value: 10.43.0.1,ansible03,localhost,.example.com,127.0.0.1
web_extra_env: |
  - name: HTTP_PROXY
    value: http://proxy.example.com:3128
  - name: HTTPS_PROXY
    value: http://proxy.example.com:3128
  - name: NO_PROXY
    value: 10.43.0.1,ansible03,localhost,.example.com,127.0.0.1
ee_extra_env: |
  - name: HTTP_PROXY
    value: http://proxy.example.com:3128
  - name: HTTPS_PROXY
    value: http://proxy.example.com:3128
  - name: NO_PROXY
    value: 10.43.0.1,ansible03,localhost,.example.com,127.0.0.1
```

You may have to adjust your settings to match your environment.

## Deploy your changes
To activate your proxy settings you need to deploy your changes using `kubectl` like this:
```
kubectl apply -k base
```

Now you need to wait some time until K3S has restarted all your pods.

