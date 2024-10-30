# Bootstrapping the etcd Cluster (practice in `Server`)

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

Download etcd binary to the `server` from: https://github.com/etcd-io/etcd/releases

Copy file `units/etcd.service` to `/root` in `Server`

## Bootstrapping an etcd Cluster

### Install the etcd Binaries

Extract and install the `etcd` server and the `etcdctl` command line utility:

```bash
{
  tar -xvf etcd-v3.4.34-linux-arm64.tar.gz
  mv etcd-v3.4.34-linux-arm64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt /etc/etcd/
}
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

Create the `etcd.service` systemd unit file:

```bash
mv etcd.service /etc/systemd/system/
```

### Start the etcd Server

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

## Verification

List the etcd cluster members:

```bash
etcdctl member list
```

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

# Set up cluster for ETCD
- Additional backup server ETCD: 
  - hostname: `backup`
  - ip: `192.168.1.6`
- Set up the same as master server
- Update etcd service config for master
```yaml
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \
  --name master \
  --initial-advertise-peer-urls http://192.168.1.3:2380 \
  --listen-peer-urls http://192.168.1.3:2380 \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://0.0.0.0:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master=http://0.0.0.0:2380,backup=http://192.168.1.6:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
- Update etcd service config for backup
```yaml
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \
  --name master \
  --initial-advertise-peer-urls http://192.168.1.6:2380 \
  --listen-peer-urls http://192.168.1.6:2380 \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://0.0.0.0:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master=http://0.0.0.0:2380,backup=http://192.168.1.3:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Troubleshoot
- When starting `etcd.service`, if you get error [cluster id mistmatch](https://stackoverflow.com/questions/40585943/etcd-cluster-id-mistmatch), remove `/var/lib/etcd` and recreate it.

```bash
rm -rf /var/lib/etcd
mkdir /var/lib/etcd
systemctl restart etcd
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
