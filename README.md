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
To delete EKS cluster
```
eksctl delete cluster --name clustername --region us-east-1         
```
* Using fargate in this case. 
* The above eksctl command creates all the necessary field for EKS CLutser. like service role, configuration for networking, public & private subnets within the VPC, application is deployed inside private subnet.

OpenID Connect provider URL  

ID - Identity provider like LDAP where we create users. 
Fargate profile : attached to default and kube-system namespaces. We can create new Fargate profile
In Authentication we can attach Identity providers. 


## Create Fargate profile for 2048 game app

```
eksctl create fargateprofile \
    --cluster game-cluster \
    --region us-east-1 \
    --name alb-sample-app \              // Fargate name - alb-sample-app
    --namespace game-2048                // this is the namespace
```

* Create deployment.yml , service.yml, ingress.yml files
* Clone the  git repo to VM

```
kubectl apply -f deployment.yml
kubectl apply -f service.yml
kubectl apply -f ingress.yml
```
To see the created resources
```
kubectl get all -n gane-2048               // "game-2048" is the name space created for the resources
kubectl get pods -n game-2048              // displays pods
kubectl get deployment -n game-2048         // displays the deployment
kubectl get svc -n game-2048               // displays services created
kubectl get ing -n game-2048               // displays ingress resource created
```

* Now the service is of "NodePort" type, anybody with access to VPC can talk to the pods using nodeIP:port
* For external users to communicate to the pods , I created ingress resource. There is no address created yet, address is for the users to access the application from outside
* We have to create ingress controller. This ing controller reads the ingress resource (ingress-2048) and creates and configures ALB 



## Deploy ALB Controller

* Any controller in K8s is a Pod. We have to grant access to Pod to AWS services like ALB. ALB Ingress controller should create Application LB.
* The pre-requisite  for ALB Controller is to configure oidc-connector. 
* Without the OIDC Connector, installing ALB Connector add-on fails.
* We need IAM OIDC Connector because, the ALB Controller running needs to access Application LB. Controller is a K8s pod.
* So for a pod to talk to AWS resources, it needs IAM Integrator, so for that purpose we need to configure IAM OIDC-Connector.

Command to configure this

```
eksctl utils associate-iam-oidc-provider --cluster cluster-name --approve
```

### Download IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

### Create IAM Policy for ALB Controller 
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Create IAM Role
* Attaching role to service account of the Pod, so that pod can interact with other AWS services
* I am creating a service account for IAM. Creating role for that

```
eksctl create iamserviceaccount \
  --cluster=game-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Deploy ALB controller
* For this I am using Helm chart, which creates actual controller which creates service account for running pod
### Install heml
```
sudo snap install helm --classic
helm version
```

### Add helm repo
```
helm repo add eks https://aws.github.io/eks-charts
```
### Update the repo
```
helm repo update eks
```
### Install ALB Controller using Helm
Helm Install Command for AWS Load Balancer Controller
This command installs the AWS Load Balancer Controller on a Kubernetes cluster using Helm
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

### Verify that the Application Load balancer controller (pods)  deployments are running.
* Check if the LB is created and there are atleast 2 replicas of it. each one in one Availability zone. It continuously watches for ingress resource
* load balancer deployments are created in "kube-system" namespace

```
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get deploy -n kube-system                   // lists the deployments under "kube-system" namespace  
```
* To check/troubleshoot the ALB controller deployment
```
kubectl describe deploy/aws-load-balancer-controller -n kube-system
```

So ALB controller has created Application Load Balancer, based on the ingress resource


To get the load balancer address (Host name), where ingress controller  created ALB watching ingress resource 
```
kubectl get ingress -n game-2048
```

Application can be accessed s=using the above LB address.


















