# Initialise the Cluster
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Run this command on the ec2 instance named `controller-0`. You can actually run it on any controller instance, but let's keep things in order.

The way kubernetes works is that all pods get assigned an internal IP address from a cidr range which is non-overlapping with VPC cidr range. In this tutorial, we created a VPC with the internal cidr range `10.0.0.0/16`. So, we designate the pod cidr range to be `192.168.0.0/16`, which is non-overlapping (play around with [cidr.xyz](https://cidr.xyz/). Why `192.168.0.0/16`? Because that's what calico - the network overlay we are going to install - uses by [default](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart). Other pod networks have their preffered pod cidr ranges that you have to read about before you initialise the cluster. Don't be hasty - read the docs!

For the record, it _is_ possible to run just `sudo kubeadm init` without the additional `--pod-network-cidr` argument. It will still report a successful cluster creation. However, when you go to add a pod network overlay, the overlay pods will crash loop because there is no designated pod cidr range. Basically, `--pod-network-cidr` is an important flag to pass when initialising the cluster.

You can look at your cluster with kubectl by using the `admin.conf` file like this:
```
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf [command]
```

After you initialise the cluster, this first controller node should be running the following as pods:
- coredns
- etcd
- kube-apiserver
- kube-controller
- kube-proxy
- kube-scheduler

Additionally, the pods for `coredns` should be trying to start but failing. They will stabilise afer we install a pod network overlay.
