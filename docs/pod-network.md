# Create a Pod Network
```
sudo kubectl --kubeconfig /etc/kubernetes/admin.config apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
```

This command installs the flannel network. Pod networks are what allow pods on different nodes to talk to eachother.

Once the flannel pod is up and running, you should see the coredns pods spin up as well.
