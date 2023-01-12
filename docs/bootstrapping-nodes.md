# Bootstrapping Nodes

When using `kubeadm` to provision a cluster, all nodes require the same initial configuration.

- `containerd` + `runc`
- `crictl`
- `kubectl`
- `kubeadm`
- `kubelet`

## Prerequisites

### Dependencies
```
sudo apt-get update -y

sudo apt-get -y install \
  socat \
  conntrack \
  ipset
```

### Turn swap off
```
sudo swapoff -a
```

### Network settings
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

The `kubeadm` [docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites) specify that these setting be applied on all nodes

Honestly I'm not entirely sure what these settings do. I think they enable VxLAN traffic, but how and why is a mystery.

## Containerd

In "Kubernetes The Hard Way", they run all the control plane and background services with `systemd`. On the contrary, when setting up with `kubeadm`, everything will run as a container. The `containerd` process itself will run as a `systemd` service. We install it using the official [docs](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#getting-started-with-containerd).

### Download and install containerd

```
wget https://github.com/containerd/containerd/releases/download/v1.6.15/containerd-1.6.15-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.15-linux-amd64.tar.gz
```

- See [the containerd repo for more versions](https://github.com/containerd/containerd/releases)

### Setup containerd for systemd
```
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable containerd
```

- notice that the `containerd.service` file is going into `/usr/local/lib/systemd/system`. If you've worked with `systemd` before, you may know that this is not the only place this file can go. [See this thread](https://askubuntu.com/questions/876733/where-are-the-systemd-units-services-located-in-ubuntu#answers) for more options.

### Download and install runc
```
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

- `runc` is like a subcomponent of `containerd` which actually does the running of the container.

### Download and install CNI plugins
```
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### Configure the containerd cgroup driver
```
sudo mkdir -p /etc/containerd/

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

## Calico configuration
```
sudo mkdir -p /etc/NetworkManager/conf.d

cat <<EOF | sudo tee /etc/NetworkManager/conf.d/calico.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF
```

Later on we are going to install calico as the pod network overlay. Other pod network overlays may require different settings. See why this is config necessary in the [calico docs](https://projectcalico.docs.tigera.io/maintenance/troubleshoot/troubleshooting#configure-networkmanager)

## crictl

### Install
```
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz
tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
chmod +x crictl
sudo mv crictl /usr/local/bin/

sudo touch /etc/crictl.yaml

cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
EOF
```

Docs [here](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/#before-you-begin)

## Kubeadm and Kubectl

### Install
```
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates curl
sudo mkdir -p /etc/apt/keyrings/
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubeadm kubectl kubelet
sudo apt-mark hold kubeadm kubectl kubelet
```

Docs [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
