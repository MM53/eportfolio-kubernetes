# Basic kubernetes tutorial

## Prepare

* Get access to an kubernetes-cluster
  * use an existing one
  * or install [Minikube](https://github.com/MM53/eportfolio-kubernetes/blob/master/tutorial/setup_minikube.md)
  * or install [kind](https://github.com/MM53/eportfolio-kubernetes/blob/master/tutorial/setup_kind.md)
  * or use the online playground at [katacoda](https://www.katacoda.com/courses/kubernetes/playground)
* Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to interact with your cluster

## Getting started 

### 1. Verify everything is working

To check that you can access the cluster simply call: 

`kubectl get nodes`

This should return a list of all nodes belonging to the cluster. Ideally all of them will have the status `Ready`. Otherwise there may be a problem with your setup.

Next you should verify that all critical pods are running by calling:

`kubectl get pods -A`

This lists all existing pods inside your cluster. Make sure at least all pods inside the `kube-system` namespace are in state `Running` as these are required for kubernetes to run correctly.

### 2. Setting up the hello-world example 

A hello-world is always a good point to start. Therefore the first thing we will do is to deploy a hello-world container as pod. The following snippet shows a minimal deployment yaml for this task.
It sets a **name** for the pod and specifies one container inside with the **name** `frontend`, the **image** `docker.pkg.github.com/mm53/hello-world-kubernetes/frontend:latest` and exposes the **port** `8080`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-world-kubernetes-frontend
spec:
  containers:
  - name: frontend
    image: docker.pkg.github.com/mm53/hello-world-kubernetes/frontend:latest
    ports:
    - containerPort: 8080
```

You can deploy this yaml be either creating this file locally or using the example from GitHub. This command deploys the pod with the git version:

`kubectl apply -f https://raw.githubusercontent.com/MM53/eportfolio-kubernetes/master/examples/hello-world-pod.yaml`

If everything works as intended you can run `kubectl port-forward hello-world-kubernetes-frontend 8080` and open your browser at `localhost:8080`
then you should see something similar to this: 

![Example image](https://raw.githubusercontent.com/MM53/eportfolio-kubernetes/master/tutorial/hello-world.png)

If you had successfully deployed this pod, you now can delete it with `kubectl delete pod hello-world-kubernetes-frontend`

The next step will be creating a deployment for the hello-world pods. A deployment takes a template for one pod and some extra information and will then take care about rolling out your pods and make sure there are always enough pods available.

For thr hello-world frontend the yaml file could look the following way:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: hello-world-frontend          # Name of the deployment
spec:
 replicas: 3                         # Count of pods that always be running
 revisionHistoryLimit: 3             # how much old versions of the configuration should be kept, importent for updates
 selector:
   matchLabels:
     app: hello-world-kubernetes     # only pods with the matching labels will be managed
     stage: frontend
 strategy:
   type: RollingUpdate               # how should updates be handeled: all at once or one after another
   rollingUpdate:
     maxSurge: 1
     maxUnavailable: 0
 template:                           # template to create pods 
   metadata:
     name: hello-world-kubernetes-frontend
     labels:
       app: hello-world-kubernetes   # labels to match the deployment
       stage: frontend
   spec:
     containers:
     - name: frontend
       image: docker.pkg.github.com/mm53/hello-world-kubernetes/frontend:latest
       ports:
       - containerPort: 8080
```

To apply the deployment run `kubectl apply -f https://raw.githubusercontent.com/MM53/eportfolio-kubernetes/master/examples/hello-world-deployment.yaml`

You should now see 3 new pods when running `kubectl get pods`

```
NAME                                    READY   STATUS    RESTARTS   AGE
hello-world-frontend-85bd48c8c6-588v4   1/1     Running   0          3s
hello-world-frontend-85bd48c8c6-5gsjp   1/1     Running   0          3s
hello-world-frontend-85bd48c8c6-t9wqj   1/1     Running   0          3s
```

To access your pods you will need to add a service which allows to connect to the pods. The service needs a selector to find the pods based on the labels and the ports of your application.
  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort                  # NodePort will be reachable outside the cluster, ClusterIP is only accessible from inside 
  selector:                       # pods with these labels are addressed
    app: hello-world-kubernetes   
    stage: frontend
  ports:
  - protocol: TCP
    port: 80                      # port which will be exposed by the service
    targetPort: 8080              # port which is exposed by the container
  # nodePort: 31280               # on this port you can access your app with the ip address of the node, will be generated (only available for type NodePort) 
```

Install the service with: `kubectl apply -f https://github.com/MM53/eportfolio-kubernetes/raw/master/examples/hello-world-service-node-port.yaml`

Now you can already access your application from every device within the same network by using the IP of the node and the port you will see under `NodePort`, when running `kubectl describe service hello-world`.

But there is another better way to achieve this if your application only needs HTTP or HTTPS traffic. In this case you could use an ingress. This is some type of "proxy" where you could configure different paths or hostnames to be routed to different services.

As we only have one endpoint at this time, it will be enough to configure a default backend as followed:  

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
spec:
  backend:
    serviceName: hello-world   # name of the service
    servicePort: 80            # port specified for the service
``` 

To set it up, first delete the old service and replace it by a ClusterIp service. Then apply the new manifests:

`kubectl delete service hello-world`

`kubectl apply -f https://github.com/MM53/eportfolio-kubernetes/raw/master/examples/hello-world-service-cluster-ip.yaml`

`kubectl apply -f https://github.com/MM53/eportfolio-kubernetes/raw/master/examples/hello-world-ingress-simple.yaml`

Now you have completely deployed your first application into a kubernetes cluster.

As a next step we can add some configuration to it using ConfigMaps. This is a way to store information or even files inside your cluster in a key-value format.

For example this ConfigMap will change the greeting message when mounted into the right place:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-greeting
data:
  greeting.html: |
    <div class=""greeting>
      Now you will see another message ... <br/>
    </div>
```

Apply it with `kubectl apply -f https://github.com/MM53/eportfolio-kubernetes/raw/master/examples/hello-world-greeting-config-map.yaml`

Then edit the deployment so that it looks the following way:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-frontend
spec:
  replicas: 3
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: hello-world-kubernetes
      stage: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      name: hello-world-kubernetes-frontend
      labels:
        app: hello-world-kubernetes
        stage: frontend
    spec:
      containers:
      - name: frontend
        image: docker.pkg.github.com/mm53/hello-world-kubernetes/frontend:latest
        ports:
        - containerPort: 8080
        volumeMounts:                                     # mount the volume in your container to the specified path
        - name: templates
          mountPath: /var/www/templates/additional
      volumes:                                            # define a new volume based on the ConfigMap
      - name: templates
        configMap:
          name: hello-world-greeting
```

You can either do this by deleting and re-adding the correct deployment.yaml or by using `kubectl edit deployment hello-world-frontend`.

### 3. Now you!

After setting up the hello-world frontend it's your turn to add the backend (although it can't do very much 😅 )

Note: If you need help with writing a manifest (yaml file), you could use `kubectl explain <config-field-path>`. For example `kubectl explain pod.spec.containers` will show you all fields with an description, you can use to configure a container inside a pod.

For the backend you will need a second deployment which uses the image `docker.pkg.github.com/mm53/hello-world-kubernetes/backend:latest` and has at least 2 replicas. It also should have different labels, e.g. `stage: backend`

You also need another ClusterIp service, there you could copy the first one and adjust the parameters to your needs.

When you have done this it's time to tell our frontend to use the backend by passing in an env-variable `BACKEND_URL` containing the in-cluster url to the backend service. (If you don't know how to do this: it was mentioned as example in the slides)

In order to display the data requested from the backend you need to override the [template](https://github.com/MM53/eportfolio-kubernetes/blob/master/tutorial/node-overview.html.tmpl) `node-overview.html.tmpl` via a ConfigMap. (Tip: generate the map with kubectl)

The last task is to expose the `/hello` endpoint of the backend to the world. To achieve this replace our old ingress object with a new one containing two endpoints where `/` is routed to the frontend and `/hello` to the backend. (This [example](https://github.com/MM53/eportfolio-kubernetes/blob/master/examples/hello-world-ingress.yaml) gives you an idea how to do it)


(I know that this example setup is a bit stupid in some parts but the goal was to show basic capabilities of kubernetes. For this purpose it will be good enough) 

The solutions can be found in the solutions folder 😉

### 4. What's next ?

Hopefully you could follow my explanations in this tutorial and are now ready to experiment with kubernetes on your own.  

As a next step containerizing your own application e.g. by using docker and start deploying it with kubernetes will probably the best. Surely you will experience some problems. But in my opinion debugging those problems is a very good way to become more familiar with the whole topic.

If it was too confusing for you, you might have a look at the official [tutorials](https://kubernetes.io/docs/tutorials/kubernetes-basics/). They will cover some different aspects. 
