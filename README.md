# K8s The Way

Resources for learning kubernetes using only the official documentation, geared towards passing the CKA exam.

## Create a Cluster By Hand on AWS

Based on [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by Kelsey Hightower in combination with it's [AWS Version](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws) by Prabhat Sharma.

It is useful to struggle through these "hard way" tutorials once or twice. But for the purposes of practical usage and passing the CKA exam, it is more important to know how to kubernetes the "official way". Basically, this means using `kubeadm` to set it up. I also chose AWS because it is the largest cloud provider and therefore can reach the most users.

My hope with this tutorial is to leave the reader satisfied, informed, and practically prepared for the CKA exam. When possible, I have added as much explanation as I can to the commands. Each command corresponds to a necessary configuration required to run kubernetes, so it is important that you understand every single one of them. Indeed, another shortcoming of the "hard way" tutorials is that they don't explain things in detail, like why we must disable source-destination checks on EC2 nodes.

Enjoy!

- [1 - Provision Compute Resources](docs/01-compute-resources.md.md)
- [2 - Bootstrap Nodes](docs/02-bootstrapping-nodes.md)
- [3 - Initialize Cluster](docs/03-init-cluster.md)
- [4 - Install Pod Network](docs/04-pod-network.md)
- [5 - Add Nodes](docs/05-add-nodes.md)

## CKA Exam Tips

[K8s The Fast Way](docs/k8s-the-fast-way.md)