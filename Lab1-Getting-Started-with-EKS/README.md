# EKS Deployment on AWS
The purpose of this simple workshop is to provide guide as a starting of your Kubernetes Journey on top of AWS. We will use eksctl, a simple CLI tool for creating clusters on EKS.
After setting up your EKS Cluster, we will deploy a microservices application. 


## 1. Turn on Cloud9 and install kubectl and aws-iam-authenticator

First step is to prepare a client machine to install kubectl and manage your EKS Cluster/WorkerNodes. We will use Cloud9 for this purpose. Spin up a Cloud9 instance by going to https://ap-southeast-1.console.aws.amazon.com/cloud9/home/product.

**IMPORTANT!** 
Cloud9 rotate credentials (secret/access key) and this is not supported by kubectl, because it detect/match the exact access key that represent IAM User that is used to initially create the cluster.
If you use Cloud9, then ensure you go to > Preferences -> AWS Settings -> Turn off "AWS managed temporary credentials"

![img1]

[img1]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/c9disableiam.png

In the IAM console, create a user with AdministratorAccess for the purpose of this lab. 

- Follow this link https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess to create an IAM role with Administrator access.
- Confirm that AWS service and EC2 are selected, then click Next to view permissions.
- Confirm that AdministratorAccess is checked, then click Next to review.
- Enter eksworkshop-admin for the Name, and select Create Role

![img2]

[img2]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/createrole.png

Next attach the IAM role to your workspace

- Follow this deep link https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=desc:launchTime to find your Cloud9 EC2 instance
- Select the instance, then choose Actions / Instance Settings / Attach/Replace IAM Role

![img3]

[img3]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/c9instancerole.png

Next configure the region **ap-southeast-1** running aws configure. You can leave the key and secret as None.

```
aws configure

```

You will observe the following output. The "error" is a warning and can be ignored.
```
$ aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [ap-southeast-1]: 
Default output format [None]: 
usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]
To see help text, you can run:

  aws help
  aws <command> help
  aws <command> <subcommand> help
aws: error: too few arguments
```

Kubernetes uses a command-line utility called kubectl for communicating with the cluster API server. Amazon EKS clusters also require the AWS IAM Authenticator for Kubernetes to allow IAM authentication for your Kubernetes cluster. Beginning with Kubernetes version 1.10, you can configure the kubectl client to work with Amazon EKS by installing the AWS IAM Authenticator for Kubernetes and modifying your kubectl configuration file to use it for authentication. 

**Install kubectl**

   ```bash
   mkdir $HOME/bin
   curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   cp ./kubectl $HOME/bin/kubectl
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   kubectl version --client
   ```
   
**Install IAM Authenticator**

   ```bash
   curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
   chmod +x ./aws-iam-authenticator
   cp ./aws-iam-authenticator $HOME/bin
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   aws-iam-authenticator help
   ```

## 3. Create your Amazon EKS Control Plane and Data Plane

Download the latest release of eksctl 

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

```

Create a basic EKS cluster

```
eksctl create cluster --name=<your cluster name> --region ap-southeast-1 --node-type t3.medium

```

A cluster will be created with default parameters

  - 2x t3.medium nodes 
  - use official AWS EKS AMI
  - ap-southeast-1 region
  - dedicated VPC (check your quotas)

Example output:

```
[ℹ]  using region ap-southeast-1
[ℹ]  setting availability zones to [ap-southeast-1a ap-southeast-1c ap-southeast-1b]
[ℹ]  subnets for ap-southeast-1a - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for ap-southeast-1c - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for ap-southeast-1b - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  nodegroup "ng-3e4f648b" will use "ami-07b922b9b94d9a6d2" [AmazonLinux2/1.12]
[ℹ]  creating EKS cluster "sgeks3" in "ap-southeast-1" region
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-1 --name=sgeks3'
[ℹ]  2 sequential tasks: { create cluster control plane "sgeks3", create nodegroup "ng-3e4f648b" }
[ℹ]  building cluster stack "eksctl-sgeks3-cluster"
[ℹ]  deploying stack "eksctl-sgeks3-cluster"
[ℹ]  building nodegroup stack "eksctl-sgeks3-nodegroup-ng-3e4f648b"
[ℹ]  --nodes-min=2 was set automatically for nodegroup ng-3e4f648b
[ℹ]  --nodes-max=2 was set automatically for nodegroup ng-3e4f648b
[ℹ]  deploying stack "eksctl-sgeks3-nodegroup-ng-3e4f648b"
[✔]  all EKS cluster resource for "sgeks3" had been created
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
[ℹ]  adding role "arn:aws:iam::284245693010:role/eksctl-sgeks3-nodegroup-ng-3e4f64-NodeInstanceRole-1BBBB85I5CX5C" to auth ConfigMap
[ℹ]  nodegroup "ng-3e4f648b" has 0 node(s)
[ℹ]  waiting for at least 2 node(s) to become ready in "ng-3e4f648b"
[ℹ]  nodegroup "ng-3e4f648b" has 2 node(s)
[ℹ]  node "ip-192-168-28-95.ap-southeast-1.compute.internal" is ready
[ℹ]  node "ip-192-168-65-212.ap-southeast-1.compute.internal" is ready
[✔]  EKS cluster "sgeks3" in "ap-southeast-1" region is ready
```

Once you have created a cluster, you will find that cluster credentials were added in ~/.kube/config.

The script requires about 15 minutes to complete. Meanwhile you can go to lab 2 to create your App Mesh Artifacts.

## 4.Test the cluster

Test your configuration. 

```
kubectl get svc
```

You should see a kubernetes svc as an output.

```
kubectl get nodes

```

You should be able to see 2 EC2 worker nodes as output

You have now completed setting up the eks cluster!


