# Initialise the Cluster
```
sudo kubeadm init --pod-network-cidr 10.10.0.0/24
```

The way kubernetes works is that all pods get assigned an internal IP address from a cidr range which is non-overlapping with VPC cidr range. In this tutorial, we created a VPC with the internal cidr range `10.0.0.0/16`. So, we designate the pod cidr range to be `10.10.0.0/16`, which is non-overlapping (see [cidr.xyz](https://cidr.xyz/). Kubernetes won't makes guesses about your network, so you have to explicitly define this pod cidr range.

For the record, it _is_ possible to run just `sudo kubeadm init` without the additional `--pod-network-cidr` argument. It will still report a successful cluster creation. However, when you go to add a pod network overlay, the overlay pods will crash loop because there is no designated pod cidr range. In the end, `--pod-network-cidr` is an important flag to pass when initialising the cluster.
