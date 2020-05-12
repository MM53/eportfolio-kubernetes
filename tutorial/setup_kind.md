# Kind - Kubernetes in Docker 

## Installation

Follow the installation instructions from their [official page](https://kind.sigs.k8s.io/docs/user/quick-start/).   

## Start a cluster

Your cluster should be able to run an ingress controller in order to route traffic from outside to pods inside the cluster. Therefore we need to do some extra configuration to get this working.

To create a cluster run the following command: 

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

Now we can install the nginx-ingress-controller with the following command:

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml`

## More information 

This tutorial is a summary of the [official guide](https://kind.sigs.k8s.io/docs/user/ingress/). You can find some additions infos there.
