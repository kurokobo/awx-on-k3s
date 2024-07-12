# Expose your `/etc/hosts` to Pods on K3s

If we don't have a DNS server and are using `/etc/hosts`, we will need to do some additional tasks to get the Pods on K3s to resolve names according to `/etc/hosts`.

This is necessary for AWX to resolve the hostname for your private Git repository or pull images from the container registry.

One easy way to do this is to use `dnsmasq`.

Note that any IP addresses in the following steps have to be replaced with your K3s host's one.

1. Add entries to `/etc/hosts` on your K3s host.

   ```bash
   sudo tee -a /etc/hosts <<EOF
   192.168.0.221 awx.example.com
   192.168.0.221 registry.example.com
   192.168.0.221 git.example.com
   192.168.0.221 galaxy.example.com
   EOF
   ```

2. Install `dnsmasq`.

   ```bash
   sudo dnf install dnsmasq
   ```

3. Add minimal configuration file for `dnsmasq`.

   ```bash
   sudo tee /etc/dnsmasq.d/10-k3s.conf <<EOF
   listen-address=192.168.0.221
   EOF
   ```

4. Enable and start `dnsmasq`.

   ```bash
   sudo systemctl enable dnsmasq --now
   ```

5. Create new `resolv.conf` for K3s.

   ```bash
   sudo tee /etc/rancher/k3s/resolv.conf <<EOF
   nameserver 192.168.0.221
   EOF
   ```

6. Add `--resolv-conf /etc/rancher/k3s/resolv.conf` as an argument for `k3s server` command.

   ```bash
   # Change configuration using script:
   $ curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.6+k3s2 sh -s - --write-kubeconfig-mode 644 --resolv-conf /etc/rancher/k3s/resolv.conf

   # If you don't want to use the script, modify /etc/systemd/system/k3s.service manually:
   $ cat /etc/systemd/system/k3s.service
   ...
   ExecStart=/usr/local/bin/k3s \
       server \
           '--write-kubeconfig-mode' \
           '644' \
           '--resolv-conf' \                  👈👈👈
           '/etc/rancher/k3s/resolv.conf' \   👈👈👈
   ```

7. Restart K3s and CoreDNS. The K3s service can be safely restarted without affecting the running resources.

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart k3s
   kubectl -n kube-system rollout restart deployment coredns
   ```

8. Ensure that your hostname can be resolved as defined in `/etc/hosts`.

   ```bash
   $ kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 -- nslookup git.example.com
   Server:    10.43.0.10
   Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      git.example.com
   Address 1: 192.168.0.221
   pod "busybox" deleted
   ```

9. If you update your `/etc/hosts`, restarting `dnsmasq` is required.

   ```bash
   sudo systemctl restart dnsmasq
   ```
