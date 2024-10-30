# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap two Kubernetes worker nodes. The following components will be installed: [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

Replace `SUBNET` in `configs/10-bridge.conf`, `configs/kubelet-config.yaml` with subnet for each node defined in `machines.txt` and copy to `~/` of `node-0`, and `node-1`  

For example: `machines.txt`:
```text
192.168.1.3 server.kubernetes.local server  
192.168.1.4 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.1.5 node-1.kubernetes.local node-1 10.200.1.0/24
```

***In `node-0`***
```yaml
# ~/10-bridge.conf
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.200.0.0/24"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
```
```yaml
# ~/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"
podCIDR: "10.200.0.0/24"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet.key"
```

Do the same with `node-1`  

Download CNI plugin binary from [link](https://github.com/containernetworking/plugins/releases) 

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz
``` 

Download `kubelet`, `kube-proxy`, `kubectl`, `kube-proxy` binary from [kubernetes lastest version](https://kubernetes.io/releases/download/)  

```bash
wget https://dl.k8s.io/v1.31.2/bin/linux/amd64/kube-proxy
wget https://dl.k8s.io/v1.31.2/bin/linux/amd64/kubectl
wget https://dl.k8s.io/v1.31.2/bin/linux/amd64/kubelet
```

```bash
for host in node-0 node-1; do
  scp \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/kubelet.service \
    units/kube-proxy.service \
    root@$host:~/
done
```

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```bash
apt-get update
apt-get -y install socat conntrack ipset
apt-get install -y containerd
```

> The socat binary enables support for the `kubectl port-forward` command.

### Disable Swap

By default, the kubelet will fail to start if [swap](https://help.ubuntu.com/community/SwapFaq) is enabled. It is [recommended](https://github.com/kubernetes/kubernetes/issues/7294) that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.

Verify if swap is enabled:

```bash
swapon --show
```

If output is empty then swap is not enabled. If swap is enabled run the following command to disable swap immediately:

```bash
swapoff -a
```

> To ensure swap remains off after reboot consult your Linux distro documentation.

### Config containerd

```bash
sudo nano /etc/modules-load.d/containerd.conf
```

Add the following two lines to the file:  

```sh
overlay
br_netfilter
```
Save the file and exit.  

Next, use the [modprobe command](https://phoenixnap.com/kb/modprobe-command) to add the modules:  

```bash
sudo modprobe overlay
```

```bash
sudo modprobe br_netfilter
```

Open the **k8s.conf** file to configure Kubernetes networking:  

```bash
sudo nano /etc/sysctl.d/k8s.conf
```
Add the following lines to the file:  

```sh
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```
Reload the configuration by typing:  

```bash
sudo sysctl --system
```
Stop and disable  **AppArmor**:  

```bash
sudo systemctl stop apparmor && sudo systemctl disable apparmor
```
Restart  **containerd**:  

```bash
sudo systemctl restart containerd.service
```

Create the installation directories:

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```bash
{
  tar -xvf cni-plugins-linux-amd64-v1.6.0.tgz -C /opt/cni/bin/
  chmod +x kubectl kube-proxy kubelet 
  mv kubectl kube-proxy kubelet /usr/local/bin/
}
```

### Configure CNI Networking

Create the `bridge` network configuration file:

```bash
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

### Configure containerd

Install the `containerd` configuration files:

```bash
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  systemctl restart containerd
}
```

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```bash
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```

### Configure the Kubernetes Proxy

```bash
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```

### Start the Worker Services

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

## Verification

The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the `jumpbox` machine.

List the registered Kubernetes nodes:

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   1m     v1.28.3
node-1   Ready    <none>   10s    v1.28.3
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
