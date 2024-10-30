# Install Flannel CNI plugin (practice in control plane node)

Remove all other cni config from `/etc/cni/net.d/`
```bash
mv /etc/cni/net.d /etc/cni/bak.net.d
mkdir /etc/cni/net.d
```

Run command to install flannel as pod in all worker nodes
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

When all `kube-flannel-***` pod is running, ensure:
- All workernodes have the `cni0` network interface bridge that is assigned ip address `.1` of ip range subnet of that node and broadcast ip is `.255`. For example, `10.200.0.0/24` is allocated to node as a subnet IP pods, so `cni0` ip address is `10.200.0.1/24` and boardcast is `10.200.0.255`.
- All workernodes have the `flannel.1` network interface that is assigned ip address `.0` (network address) of pod subnet. For example, `10.200.0.0/24` is allocated to node as a subnet IP pods, so `Flannel.1` ip address is `10.200.0.0/32`.  

### Troubleshoot
If ip address of `cni0` virtual bridge is out of subnet ip range, remove `cni0` and restart `kubelet` to recreate.
```bash
sudo rm -rf /var/lib/cni/networks/cbr0
sudo ip link delete cni0
systemctl restart kubelet
```


Next: [Smoke Test](12-smoke-test.md)
