# Create a Pod Network
```
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml
```

We install calico according to the [docs](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart#install-calico)

As of 12-1-2023 there is a bug in the install. It can be fixed as seen in this [thread](`https://github.com/projectcalico/calico/issues/7003#issuecomment-1368926816`).

As a note, messing with the pod network overlay is very frustrating. The learning curve is extremely steep and the docs quickly devolve into raw spec with little to no commentary. Playing around with calico, flannel, weave etc. is the fastest way to give you that genuine "wtf k8s" feeling. Prepare to be own your own with very few people on the internet able to help.

