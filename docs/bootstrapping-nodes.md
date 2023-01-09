# Bootstrapping Nodes

When using `kubeadm` to provision a cluster, all nodes -- control plane and worker -- require the same configuration. The commands in this section should be run on all servers.

## Prerequisites

### Update
```
sudo apt-get update -y
```

### Network settings
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

The `kubeadm` [docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites) specify that these setting be applied on all nodes

Verify the settings by running:
```
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## Containerd

In "Kubernetes The Hard Way", they run all the control plane and background services with `systemd`. On the contrary, when setting up with `kubeadm`, everything will run as a container.

K8s is a bring-your-own-container system. Popular choices are `docker`, `containerd` and `crio-o`, although technically you could write your own it it's compliant. In this tutorial we will use `containerd` as it seems to be the most popular choice, and `docker` is becoming passé.

The `containerd` process itself will run as a `systemd` service. We install it using the official [docs](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#getting-started-with-containerd).

### Download and install containerd

```
wget https://github.com/containerd/containerd/releases/download/v1.6.15/containerd-1.6.15-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.6.15-linux-amd64.tar.gz
```

### Setup containerd for systemd
```
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
mv containerd.service /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable containerd
```

### Download and install runc
```
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Download and install CNI plugins
```
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### Configure the containerd cgroup driver
```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
EOF
```

The kubernetes [docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers) state that `kubelet` and `containerd` must run in the same cgroup. By defualt they do not. Since Ubuntu -- and most other modern linux distros -- use `systemd`, we configure `kubelet` and `continerd` to use the `systemd` cgroup.

### Start containerd
```
sudo systemctl start containerd
```

### Verify installation
```
sudo systemctl status containerd
```
