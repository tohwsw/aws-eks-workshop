

# Getting Started with EKS

## Overview

This lab introduces the basics of working with microservices and [EKS](https://aws.amazon.com/eks/). This includes: setting up the initial EKS cluster, deploying the sample application, service discovery and deployment of the containers with traffic routed through via AWS App Mesh. The sample application that will be running on EKS requires a couple Docker images built and placed in an ECR repository that your EKS cluster has access to.

**Note**: You should have containers and Docker knowledge before attempting this lab. You should know the basics of creating a Docker file and a Docker image, and checking in the image into a Docker registry. Otherwise, please complete part 1 and 2 of the [Get Started with Docker tutorial](https://docs.docker.com/get-started/).

The sample application consists of ColorGateway and ColorTeller applications. 

**ColorGateway**

Color-gateway is a simple http service written in go that is exposed to external clients and responds to http://service-name:port/color that responds with color retrieved from color-teller and histogram of colors observed at the server that responded so far. For e.g.

```
$ curl -s http://colorgateway.default.svc.cluster.local:9080/color
{"color":"blue", "stats": {"blue":"1"}}
```

color-gateway app runs as a service in EKS. 

**ColorTeller**

Color-teller is a simple http service written in go that is configured to return a color. This configuration is provided as environment variable and is run within a task. Multiple versions of this service are deployed each configured to return a specific color.

At the microservice level the application looks like this:

![img1]

[img1]:https://github.com/tohwsw/aws-ecs-workshop/blob/master/Lab1-Getting-Started-with-ECS/img/microservicesapp.png

They shall be deployed using AWS EKS, which makes it easy for you to run Kubernetes on AWS without needing to install and operate your own Kubernetes clusters. We shall deploy two instances of Gateway container and 2 instances of ColorTeller. We will use AWS App Mesh for ColorGateway to discover ColorTeller. AWS App Mesh is a service mesh based on the Envoy proxy that makes it easy to monitor and control microservices. App Mesh standardizes how your microservices communicate, giving you end-to-end visibility and helping to ensure high-availability for your applications. 


**Note**: 
You'll need to have a working AWS account to use this lab.

