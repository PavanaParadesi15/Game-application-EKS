# Game-application-EKS
This a gaming application deployed on AWS EKS

# Welcome to the Game-application-EKS wiki!

* Create a VPC with public and private subnets and deploy application into private subnet
* I am deploying this application on AWS EKS Platform with public facing IP, with access through LB.
* I created service, ingress resource, ingress controller and will access application through LB.


# Setting up EKS Infrastructure

* Create a EC2 instance  to intall all the dependencies for EKS cluster .
* SSH to the VM 

### Install kubectl

```
sudo apt update
sudo apt install curl -y
sudo curl -LO "https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl"
sudo chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### Install eksctl

```
sudo apt update
sudo apt install curl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
sudo chmod +x /usr/local/bin/eksctl
eksctl version 
```

### Install AWS CLI

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### Configure AWS 

```
aws configure
```
Provide access key and secret access key

## Create EKS Cluster

One of the preffered way to create EKS Cluster is using eksctl
```
eksctl create cluster --name game-cluster --region us-east-1 --fargate
```

* Using fargate in this case. 
* The above eksctl command creates all the necessary field for EKS CLutser. like service role, configuration for networking, public & private subnets within the VPC, application is deployed inside private subnet.





