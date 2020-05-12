# Minikube 

## Installation

Follow the installation instructions on [kubernetes.io](https://kubernetes.io/docs/tasks/tools/install-minikube/).   

## Start a cluster

(I haven't tested this setup locally, as I had some problems with enabling virtualization on my pc. But it should work fine if your computer is able to run virtual machines.)

To create a cluster run the following command: 

`minikube start`

To access your applications after deploying it you will need to enable ingress with the following command: 

`minikube addons enable ingress`

## More information 

This tutorial is a summary of these guides [Installing Kubernetes with Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [Set up Ingress on Minikube with the NGINX Ingress Controller](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/). You can find some additions infos there.
