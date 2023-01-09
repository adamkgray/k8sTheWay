# Bootstrapping Nodes

When using `kubeadm` to provision a cluster, all nodes -- control plane and worker -- require the same configuration. The commands in this section should be run on all servers.

## Multiplexing

If you are new to `tmux`, this section explains what it is and how to use it.

The name `tmux` is a shortening of the phrase "terminal multiplexer". It lets you run multiple terminals screens at once. Now, if you run a modern terminal emulator like `iterm2` or `alacritty`, these already come with support for things like tabs and split panes. What makes `tmux` different is that it allows you to do split screen on on _any_ terminal, regardless of whether the parent app supports it. The use case for this is that you can have the same workflow on your local machine _and on remote servers_. Here are some of the core commands:

- start tmux: `tmux`
- exit tmux: `exit` or `<ctrl-d>` (`tmux` exits when the shell exits)
- split panes vertically: `<ctrl-b> %`
- split panes horizontally: `<ctrl-b> "`
- toggle pane full screen: `<ctrl-b> z`
- new tab/window: `<ctrl-b> c"`
- open up tab/window browser: `<ctrl-b> w"` (then use vim-keys or d-pad to select a tab and press `enter`)
- send same keystrokes to all panes: `<ctrl-b> :setw synchronize-panes on <enter>`
- stop same keystrokes to all panes: `<ctrl-b> :setw synchronize-panes off <enter>`

Log into all the servers created in the previous section in different `tmux` panes, and then synchronize keystrokes. If that's too much for you, just do it all one-by-one.

## Prerequisites

Let's begin preparing the servers.

### Update
```
sudo apt-get update -y`
```

### Network Settings
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


