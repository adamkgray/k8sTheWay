# Add Nodes

## Get join command

Run the following command on `controller-0` to get the appropriate join command for your worker node(s):

```
sudo kubeadm token create --print-join-command
```

## Add worker nodes

The above command should give you something like this: `kubeadm join <apiserver>:<port> --token <letters>.<numbers> --discovery-token-ca-cert-hash sha256:<hash>`. You can copy-paste this output into your worker node, run it with `sudo`. Then your worker node will join the cluster.

## Add control plane nodes

Adding a control plane node is basically the same process as adding a worker node, but we need to copy the certs from the initial control node to the new control node first.

Add the arg `--control-plane` to add a control plane instead of a worker.
