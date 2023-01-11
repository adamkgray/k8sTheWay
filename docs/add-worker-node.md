# Add Worker Nodes

## Get join command

Run the following command on `controller-0` to get the appropriate join command for your worker node(s):

```
sudo kubeadm token create --print-join-command
```

This should give you something like this: `kubeadm join <apiserver>:6443 --token <letters>.<numbers> --discovery-token-ca-cert-hash sha256:<hash>. You can copy-paste this output into your worker node, run it with `sudo`. Then your worker node will join the cluster.
