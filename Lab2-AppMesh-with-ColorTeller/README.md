This lab builds upon lab 1. This has been updated to use appmesh.k8s.aws/v1beta2


## 5. Build the Docker Images and push them to ECR

First create the ECR repositories for the 2 applications.

```
aws ecr create-repository --repository-name colorteller

aws ecr create-repository --repository-name colorgateway
```

In the terminal of Cloud9, clone the code

```
git clone https://github.com/tohwsw/aws-app-mesh-examples.git
```

Retrieve the login command to use to authenticate your Docker client to your registry.

```
$(aws ecr get-login --no-include-email --region ap-southeast-1)
```

Go to the folder examples/apps/colorapp/src/colorteller. Execute a docker build with the respective repository uri for colorteller and push it to the repository.

```
cd ~/environment/aws-app-mesh-examples/examples/apps/colorapp/src/colorteller

docker build -t XXXXXXXXXXXX.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller .

docker push XXXXXXXXXXXX.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller:latest
```

Go to the folder examples/apps/colorapp/src/gateway. Execute a docker build with the respective repository uri for colorgateway and push it to the repository.

```
cd ~/environment/aws-app-mesh-examples/examples/apps/colorapp/src/gateway

docker build -t XXXXXXXXXXXX.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway .

docker push XXXXXXXXXXXX.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway:latest
```

## 6. Install App Mesh controller and sidecar injector

When you use AWS App Mesh with Kubernetes, you manage App Mesh resources, such as virtual services and virtual nodes, that align to Kubernetes resources, such as services and deployments. You also add the App Mesh sidecar container images to Kubernetes pod specifications. This tutorial guides you through the installation of the following open source components that automatically complete these tasks for you when you work with Kubernetes resources: 

**App Mesh control controller**

Install the integration components

```
curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh
./pre_upgrade_check.sh

helm repo add eks https://aws.github.io/eks-charts

kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"

kubectl create ns appmesh-system

export CLUSTER_NAME=cluster-name
export AWS_REGION=ap-southeast-1

eksctl utils associate-iam-oidc-provider \
    --region=$AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve

eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn  arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
    --override-existing-serviceaccounts \
    --approve

```

Deploy the APP Mesh controller

```
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller

```




## 6. Create AppMesh and the required configuration

The following diagram represents the abstract view in terms of App Mesh resources:

![img1]

[img1]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab2-AppMesh-with-ColorTeller/img/appmesh.png

Create the App Mesh

```
curl https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/mesh.yml | kubectl apply -f -

```

Create the Namespace

```
curl https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/namespace.yml | kubectl apply -f -

```

Copy and paste these three virtual node definitions into your terminal to create the black, blue, and red collorteller virtual nodes:

```
curl https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/virtualnode.yml | kubectl apply -f -

```

Create the virtual service

```
curl https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/virtualservice.yml | kubectl apply -f -

```

Create the virtual router

```
curl https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/virtualrouter.yml | kubectl apply -f -

```

Up to this point, we’ve created all required components of the app mesh (virtual services, virtual nodes, virtual routers, and routes) to support our application.

## 7. Deploy the application

To deploy the app, download colorapp.yml and and deploy it.

```
curl -O https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/colorapp.yml

```

**Important** You will need to update color.yaml with the ECR respository uri of your colorgateway and colorteller docker images.
You can do so by substituting the account id 284245693010 with your own.

```
sed -i 's/284245693010/<your account id>/g' colorapp.yml

```

Next, deploy the applications.


```
kubectl apply -f colorapp.yml

```

You can verify that the pods are running properly.

```
kubectl get pods

```


## 8. Test the application

A network load balancer is also deployed. Identify the address of the elb via the command.

```
export NLB=$(kubectl get svc -o wide |grep colorgateway | awk '{print $4}')

```


Next, paste the following to repeatedly request the colorgateway service. Remember to replace the address of the elb with your own. Do note that the elb might take a few minutes to become active.

```
while [ 1 ]; do  curl -s --connect-timeout 2 $NLB:/color;echo;sleep 1; done

```

You should see a different color is returned on each request. This is because colorgateway always forwards to colorteller, which via the colorteller-route, always routes to one of the color backends.

Let’s modify the colorteller-route so it instead routes to the blue, red, and black colorteller virtual nodes, each at a 30% weighted ratio.

If you look at the curler terminal, you should now see an equal distribution of traffic to the blue, red, and black virtual nodes. This shows that App Mesh is now controlling the distribution of traffic!

## 9. Lab Cleanup

To clean up the lab, please delete using the following command

```
eksctl delete cluster --name=$CLUSTER --region=ap-southeast-1

```

## 10. Optional Labs

There are more labs hosted at https://eksworkshop.com/. Do check them out!








