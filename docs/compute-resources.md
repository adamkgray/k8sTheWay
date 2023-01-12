# Provisioning Compute Resources

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

### Security Groups
```
CP_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name k8stheway-cp \
  --description "k8s cp security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

WORKER_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name k8stheway-worker \
  --description "k8s worker security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

aws ec2 create-tags --resources ${CP_SECURITY_GROUP_ID} --tags Key=Name,Value=k8stheway-cp
aws ec2 create-tags --resources ${WORKER_SECURITY_GROUP_ID} --tags Key=Name,Value=k8stheway-worker

# control plane rules
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 6443 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 5473 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 179 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 5473 --source-group ${WORKER_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 179 --source-group ${WORKER_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 6443 --source-group ${WORKER_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 2379-2380 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 10250 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 10259 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${CP_SECURITY_GROUP_ID} \
  --protocol tcp --port 10257 --source-group ${CP_SECURITY_GROUP_ID}


# worker rules
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 10250 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 10250 --source-group ${WORKER_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 5473 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 179 --source-group ${CP_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 5473 --source-group ${WORKER_SECURITY_GROUP_ID}
aws ec2 authorize-security-group-ingress \
  --group-id ${WORKER_SECURITY_GROUP_ID} \
  --protocol tcp --port 179 --source-group ${WORKER_SECURITY_GROUP_ID}

# authorise ssh for control plane and workers
aws ec2 authorize-security-group-ingress --group-id ${CP_SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${WORKER_SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
```

We create the security group according to the required ports and protocols as specified in the [kubernetes documentation](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

- we allow traffic to control planes nodes on `6443` from anywhere. That's because at first we won't put any control plane nodes behind a load balancer. However, if you are designing for high-availability, then you need to put the control plane nodes behind a load balancer. In that case, we can restrict access to port `6443` only from the local VPC, and instead allow open access to `443` on the load balancer. More on that later.
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
    --security-group-ids ${CP_SECURITY_GROUP_ID} \
    --instance-type t3.small \
    --private-ip-address 10.0.1.1${i} \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 create-tags --resources ${INSTANCE_ID} --tags "Key=Name,Value=controller-${i}"
  echo "controller-${i} created "
done
```

Create 2 controller nodes - 1 main node and 1 that we will join to it. We explicitly assign these instances private IPs with values `10.0.1.10` and `10.0.1.11`. Explicit IPs are not necessary for setup, but they makes it trivial put the controllers behind a network load balancer later on. The alternative would be to place these instances in an autoscaling group, but that is going too far for this tutorial.

We use `t3.small` instances because the kubeadm [docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) recommend "2 GB or more of RAM per machine" and "2 CPUs or more"

### Workers
```
for i in 0 1; do
  INSTANCE_ID=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8stheway \
    --security-group-ids ${WORKER_SECURITY_GROUP_ID} \
    --instance-type t3.small \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 create-tags --resources ${INSTANCE_ID} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```

Create 2 worker nodes. They do not need explicit private IPs.
