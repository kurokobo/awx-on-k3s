# Expose your `/etc/hosts` to Pods on K3s

If we don't have a DNS server and are using `/etc/hosts`, we will need to do some additional tasks to get the Pods on K3s to resolve names according to `/etc/hosts`.

This is necessary for AWX to resolve the hostname for your private Git repository or pull images from the container registry.

One easy way to do this is to use `dnsmasq`.

1. Add entries to `/etc/hosts` on your K3s host. Note that the IP addresses have to be replaced with your K3s host's one.

   ```bash
   sudo tee -a /etc/hosts <<EOF
   192.168.0.100 awx.example.com
   192.168.0.100 registry.example.com
   192.168.0.100 git.example.com
   192.168.0.100 galaxy.example.com
   EOF
   ```

2. Install and start `dnsmasq` with default configuration.

   ```bash
   sudo dnf install dnsmasq
   sudo systemctl enable dnsmasq --now
   ```

3. Create new `resolv.conf` to use K3s. Note that the IP addresses have to be replaced with your K3s host's one.

   ```bash
   sudo tee /etc/rancher/k3s/resolv.conf <<EOF
   nameserver 192.168.0.100
   EOF
   ```

4. Add `--resolv-conf /etc/rancher/k3s/resolv.conf` as an argument for `k3s server` command.

   ```bash
   # Change configuration using script:
   $ curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --resolv-conf /etc/rancher/k3s/resolv.conf

   # If you don't want to use the script, modify /etc/systemd/system/k3s.service manually:
   $ cat /etc/systemd/system/k3s.service
   ...
   ExecStart=/usr/local/bin/k3s \
       server \
           '--write-kubeconfig-mode' \
           '644' \
           '--resolv-conf' \     👈👈👈
           '/etc/rancher/k3s/resolv.conf' \     👈👈👈
   ```

5. Restart K3s and CoreDNS. The K3s service can be safely restarted without affecting the running resources.

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart k3s
   kubectl -n kube-system delete pod -l k8s-app=kube-dns
   ```

6. Ensure that your hostname can be resolved as defined in `/etc/hosts`.

   ```bash
   $ kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 -- nslookup git.example.com
   Server:    10.43.0.10
   Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      git.example.com
   Address 1: 192.168.0.100
   pod "busybox" deleted
   ```

7. If you update your `/etc/hosts`, restarting `dnsmasq` is required.

   ```bash
   sudo systemctl restart dnsmasq
   ```
