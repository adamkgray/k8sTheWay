# Create a Pod Network
```
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml
```

We install calico according to the [docs](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart#install-calico)

