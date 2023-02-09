# K8s The Way

Resources for learning kubernetes using only the official documentation, geared towards passing the CKA exam.

## Create a Cluster By Hand on AWS

Based on [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by Kelsey Hightower in combination with it's [AWS Version](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws) by Prabhat Sharma.

It is useful to struggle through these "hard way" tutorials once or twice. But for the purposes of practical usage and passing the CKA exam, it is more important to know how to kubernetes the "official way". Basically, this means using `kubeadm` to set it up. I also chose AWS because it is the largest cloud provider and therefore can reach the most users.

My hope with this tutorial is to leave the reader satisfied, informed, and practically prepared for the CKA exam. When possible, I have added as much explanation as I can to the commands. Each command corresponds to a necessary configuration required to run kubernetes, so it is important that you understand every single one of them. Indeed, another shortcoming of the "hard way" tutorials is that they don't explain things in detail, like why we must disable source-destination checks on EC2 nodes.

---

# 1. Provisioning Compute Resources

There is no one way to set up the compute resources for a kubernetes cluster. This tutorial uses AWS to provision 4 servers in a reasonably secure network.

## Networking

### VPC
```
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId')
aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=k8stheway
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
```

This VPC cidr range goes from `10.0.0.0` to `10.0.255.255`.

For more information on subnet ranges, play around with [cidr.xyz](cidr.xyz)

### Subnet
```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.1.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=k8stheway
```

This subnet cidr range goes from `10.0.1.0` to `10.0.1.255`.

Since we are setting up all EC2 instances by hand, it is convenient to place them in a single public facing subnet and give them public IP addresses. In real life you should keep all instances in a private subnet and only access them via a bastion server.

### Internet Gateway
```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=k8stheway
aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
```

If you are new to setting up a VPC in AWS, an internet gateway is essentially a static resource with no config that allows a VPC access to the internet.

### Route Tables
```
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=k8stheway
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
```

If you are new to setting up a VPC in AWS, route tables explicitly redirect network traffic from a cidr range to a destination. Think of it like plumbing to redirect the flow of water. When you create a VPC, by default it routes requests from within the VPC cidr range to `local`. If you create an internet gateway, you must explicitly route all non-local requests, i.e. `0.0.0.0/0` to the internet gateway. Without an entry for the internet gateway in the route table, AWS doesn't know where to send your non-local request.

```
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
```

It is not enough to merely create the route table. You must associate it with one or more subnets. Here we associate the route table we created with the public subnet we created earlier. To continue the metaphore, we apply this plumbing to that bathroom.

### Security Groups
```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name k8stheway \
  --description "k8stheway security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=k8stheway

# Allow all inter-node communication
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all --source-group ${SECURITY_GROUP_ID}

# kube-apiserver access from local network
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp --port 6443 --cidr 10.0.0.0/16

# NodePort services access from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0

# SSH access from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

We create the security group according to the required ports and protocols as specified in the [kubernetes documentation](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

- we allow SSH from anywhere into the nodes. This is not best practice, but neither is running AWS commands raw from the terminal to create your k8s cluster. In practice it should be very difficult for any human to directly log into your servers.
- we allow traffic to worker `node port` services from anywhere with the intent that we can deploy some nginx servers there and test them later on. In practice you can restrict access to these ports in a more intelligent way that alligns with your solution.
- in the *Kubernetes The Hard Way* tutorial they explicitly allow traffic from the pod cidr range into the security group. Indeed, K8s requires that pods communicate over a network range that is *completely outside your VPC cidr range*. The hard way is to accomplish this without a CNI plugin is by routing it all yourself. This makes it a pain in the ass to add or remove nodes from the cluster. But in the CKA exam they will use a CNI plugin, and in real life you should too. So, we do not need to handle the pod network explicitly in the security group.

## Compute Instances

### Image
```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --output json \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```

In this tutorial we will use Ubuntu 20.04. You are making your life harder if you choose to use something else.

### SSH Key Pair
```
aws ec2 create-key-pair --key-name k8stheway --output text --query 'KeyMaterial' > k8stheway
chmod 400 k8stheway
mv k8stheway ~/.ssh/
```

Create a key-pair called `k8stheway` and put it in your `~/.ssh` directory. We will use it to log into all compute instances.

### Controllers
```
for i in 0 1; do
  INSTANCE_ID=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8stheway \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.small \
    --private-ip-address 10.0.1.1${i} \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --no-source-dest-check
  aws ec2 create-tags --resources ${INSTANCE_ID} --tags "Key=Name,Value=controller-${i}"
  echo "controller-${i} created "
done
```

Create 2 controller nodes - 1 main node and 1 that we will join to it. We explicitly assign these instances private IPs with values `10.0.1.10` and `10.0.1.11`. Explicit IPs are not necessary for setup, but they makes it trivial put the controllers behind a network load balancer later on. The alternative would be to place these instances in an autoscaling group, but that is going too far for this tutorial.

We use `t3.small` instances because the kubeadm [docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) recommend "2 GB or more of RAM per machine" and "2 CPUs or more"

Notice that we disable the source-destination checks for the nodes. *This is very important!* When using a network overlay, inter-pod communication will happen in a network that is outside the jurisdiction of EC2. Unless we disable the source-destination checks, EC2 will try to stop this traffic by default. So we need to explicitly tell AWS that we are doing our own thing.

### Workers
```
for i in 0 1; do
  INSTANCE_ID=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8stheway \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.small \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --no-source-dest-check
  aws ec2 create-tags --resources ${INSTANCE_ID} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```

Create 2 worker nodes. They do not need explicit private IPs.

### Load Balancer
```
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
  --name k8stheway \
  --subnets ${SUBNET_ID} \
  --scheme internet-facing \
  --type network \
  --output text --query 'LoadBalancers[].LoadBalancerArn')

TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name k8stheway \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')

aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=10.0.1.1{0,1}

aws elbv2 create-listener \
  --load-balancer-arn ${LOAD_BALANCER_ARN} \
  --protocol TCP \
  --port 443 \
  --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
  --output text --query 'Listeners[].ListenerArn'

KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

We create an internet-facing network load balancer. It forwards all TCP traffic to the controllers. Although it is theoretically possible to spin up a k8s cluster using `kubeadm` without a load balancer for the controllers, your team will want it eventually, so better to future proof. A simple network load balancer in `us-east-1` won't break the bank (with little traffic it could be like $20 a month?)

---

# 2. Bootstrapping Nodes

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

--

# 3. Initialise the Cluster

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=<KUBERNETES_PUBLIC_ADDRESS>:443
```

The `KUBERNETES_PUBLIC_ADDRESS` is the domain name of the network load balancer. Replace it with the correct value. You have to specify port `443` because by default it will look for `6443` instead.

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

---

# 4. Create the Pod Network

```
sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml

sudo kubectl --kubeconfig /etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml
```

We install calico according to the [docs](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart#install-calico). This step is simple but it's doing a lot under the hood. Essentially, we install the calico operator and some CRDs. Calico runs some main systems components, and then nodes as a daemonset on all other nodes.

As a note, messing with the pod network overlay is very frustrating. The learning curve is extremely steep and the docs quickly devolve into raw spec with little to no commentary. Playing around with calico, flannel, weave etc. is the fastest way to give you that genuine "wtf k8s" feeling. Prepare to be own your own with very few people on the internet able to help.

After we install calico, run a `watch kubectl get pods -n kube-system` and wait until the `coredns` pods come up. If everything is running, good job.

---

# 5. Add Nodes

## Get join command

Run the following command on `controller-0` to get the appropriate join command for your worker node(s):

```
sudo kubeadm token create --print-join-command
```

## Add worker nodes

The above command should give you something like this: `kubeadm join <apiserver>:<port> --token <letters>.<numbers> --discovery-token-ca-cert-hash sha256:<hash>`. You can copy-paste this output into your worker node, run it with `sudo`. Then your worker node will join the cluster.

## Add control plane nodes

Adding a control plane node is basically the same process as adding a worker node, but we need to copy the certs from the initial control node to the new control node first. We can achieve this with one `scp` command like so:
```
```

Then add the arg `--control-plane` to add a control plane instead of a worker.

---

# Exam Tips, or "K8s The Fast Way 🏎️"

If you are comfortable with kubernetes, maybe even consider yourself good at it, then your biggest enemy on the CKA exam is time. Practically, you have fewer then 10 minutes per question. For questions that only ask you to create some pods, roles, secrets etc. - you need to be able to sail through, even when there are a lot of sub-tasks. A deployment with a serviceaccount, role, custom command, volume, and antiaffinity policy should be trivial. This guide will help you memorize the most common commands and template locations in the deocumentation.

## Imperative

The following resources can be created imperatively. The notes below highlight the most important flags.

For key-value type things (like labels, annotation etc.) pass them as a list of params. For example, `kubectl run my-webserver --image=nginx --labels="env=prod,stack=frontend"`

Limited functionality is exposed via imperative commands, which is a blessing and a curse. The bad news is that you'll never know ahead of time when a flag you think should exist is actually unsupported. The good news is that there are fewer commands to keep in mind.

The name of the game is doing as much as you can imperatively. If that doesn't suffice, write the output of the imperative command to yaml and then edit it by hand before applying it.

Tip: remember that most of these command can take the `-n <namespace>` arg as well to specify which namespace the resource should be created in. Here it is omitted to avouid repetition.

#### Pod

`kubectl run <name>`
- `--image`
- `--labels=[]`
- `--annotations=[]`
- `--env=[]`
- `--port`
- `--privileged`
- `-- [COMMAND] [args]` (at the end of the imperative command)

Opening ports on a container has sort of a funny syntax declaritively, so it is best to use the `--port` option here to do the dirty work for you.

#### Deployment

`kubectl create deployment <name>`
- `--image`
- `--replicas`
- `--port`
- `-- [COMMAND] [args]` (at the end of the imperative command)

Like pods, opening ports on containers has sort of a funny syntax declaritively, so it is best to use the `--port` option here to do the dirty work for you.

If you need to create multiple containers in a deployment, just create one container with as much config as you can first. Then write the yaml to a file and hand edit it there.

The other cool thing you can do with deployments is `kubectl scale deployment <name>`, and then pass the arg `--replicas`.

#### Job
`kubectl create job <name>`
- `--image`
- `--from`

#### Cronjob
`kubectl create cronjob <name>`
- `--image`
- `--schedule`

#### Service
You can create a service explicitly with `kubectl create service <type> <args>`. But in the exam you don't want to commit this much mental energy to the task. Luckily, we have `kubectl expose <resource>` to sort all the particulars for us.

`kubectl expose <resource> <name>`
- `--name`
- `--type`
- `--port`
- `--target-port`

#### Servicaccount
`kubectl create serviceaccount <name>`

Tip: besides perhaps a namespace, this one never needs additional args.

#### (Cluster)Role(Binding)

`kubectl create (cluster)role <name>`
- `--verb=[]`
- `--resource=[]`

`kubectl create (cluster)rolebinding <name>`
- `--role` or `--clusterrole`
- `--group` or `--serviceaccount`

In my humble opinion, it is in fact easier to create roles and rolebindings imperatively. Declaritively, you have to fumble around to find the names of the API groups. The imperative command connects all the dots for you. Nothing to it.

The only gotchya is that the value of `--serviceaccount` must be in the form of `<namespace>:<name>`. Don't fret, if you omit the namespace part `kubectl` will yell at you and tell you to add it.

#### Secret
`kubectl create secret generic <name>`
- `--from-literal="foo=bar="`
- `--from-file=/path/to/file`

Note the you always want to create a "generic" secret. There are other types but I don't think they come up on the exam.

Further note that the args for this imperative command are **not lists**! You have to specify multiple `--from-literal` args to create a secret with multiple keys in it. If you do something like `--from-literal="foo=bar,baz=buz`, then you will just end up with a single key `foo` containing the base64 value of `bar,baz=buz` which is probably not what you meant.

#### ConfigMap
`kubectl create cm <name>
- `--from-literal="foo=bar="`
- `--from-file=/path/to/file`
- `--from-env-file=/path/to/file`

The `--from-literal` arg is the same deal as secrets; it must be a single value - not a list.

#### PriorityClass
`kubectl create priorityclass <name>`
- `--value`

## CTRL-F

The syntaxes for the following resources (and configurations) can only be found through examples in the K8s documentation. The table below indicates the `resource`, `page` and `ctrl-f term` for each resource you may want to make.

Resource | Page | Term
---|---|---
PersistentVolume | Persistent Volumes | pv0003
PersistentVolumeClaim | Persistent Volumes | myclaim
Pod with PVC | Persistent Volumes | mypod
Pod with Ephemeral Volume | Volumes | cache-volume
Ingress | Ingress | minimal ingress
Daemonset | Daemonset | fluentd-elasticsearch
Secret as env | Secrets | secret-env-pod
Secret as volume | Secrets | mypod
ConfigMap as env and volume | ConfigMaps | configmap-demo-pod
Pod with toleration | Taints and Tolerations | nginx
nodeSelector | Assign Pods to Nodes | disktype: ssd
(Anti)affinity | Assign*ing* Pods to Nodes | web-store
Liveness and Readiness with command | Configure Liveness, Readiness and Startup Probes | liveness-exec
Liveness and Readiness with http | Configure Liveness, Readiness and Startup Probes | liveness-http
Pod metadata | Expose Pod Information to Containers Through Environment Variables | MY_NODE_NAME
Network Policy | Network Policies | test-network-policy