# Deployement and Maintenance of cluster with Kubernetes

# Specs

Can be deployed on any OS that supports Docker.
We will be deploying on Debian Stable. 

# Dependencies

- [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) 
- [kind](https://kind.sigs.k8s.io/)

# Structure

- [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/): It's the configuration for virtual domains, traffic management and certificats. *requires ingress-nginx*

/k8s/ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1    # specify the version of the api that we are using
kind: Ingress                       # specify that we are configuring an ingress config for kubernetes
metadata:
  name: ingress                     # metadata, used internally by the system
spec:                              
  defaultBackend:                   
    service:                        # we specify the name and port of the running routing service
      name: foo-service
      port:
        number: 5678
  rules:                            # we specify the proxy rules used to dispatch requests
  - http:                           # the type of the request protocol
      paths:
      - pathType: Prefix
        path: "/foo"                # if we type /foo, dispatch to foo-service port 5678
        backend:
          service:
            name: foo-service
            port:
              number: 5678
      - pathType: Prefix
        path: "/bar"                # if we type /bar, dispatch to bar-service port 5678
        backend:
          service:
            name: bar-service
            port:
              number: 5678
```
- [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) : templates of a pod.

/k8s/deployments/bar.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bar-deployment
  labels:
    app: bar
spec:
  replicas: 1                               # the default number of replicas that we want for this pod
  selector:                                 # used to link the template with the pod
    matchLabels:
      app: bar
  template:                                 # the actual template that will be used to generate a new pod
    metadata:
      labels:
        app: bar
    spec:
      containers:
      - name: bar                           # the name of the container
        image: hashicorp/http-echo:0.2.3    # the image of the container, it's a simple dummy website that echos some text
        args:
        - "-text=bar"                       # what is the text that the website should return (in our case "bar")
```
/k8s/deployments/foo.yaml

(Similar to bar.yaml, but only for a foo service, and the text from the webserver is "foo", not "bar")
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-deployment
  labels:
    app: foo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      - name: foo
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=foo"
```

- [pod](https://kubernetes.io/docs/concepts/workloads/pods/) : Group of one or more containers, with shared storage and network resources.
- [hpa](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) : horizontal pod autoscaling, a service that describes the metrics used to monitor activities of the pods and replicates them accordingly based on preconfigured threshold values. *requires a metric server*

/k8s/hpa/foo.yaml
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler       # we are saying that this config is for an Autoscaler service
metadata:
  name: foo-hpa                     # the name of the service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: foo-deployment            # what deployement we are monitoring
  minReplicas: 1                    # minimum number of replicas, so obviously one at least
  maxReplicas: 10                   # maximum number of replicas 
  metrics:                          # the metrics used for monitoring
  - type: Resource
    resource:
      name: cpu                     # we are watching the CPU
      target:
        type: Utilization
        averageUtilization: 5       # percentage of utilization of the CPU
```

- kind configuration
/kind/cluster.yaml
```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
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
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

-  exposed ports :  80 & 443 (externally exposed ports for http and https)

# Commands

To create a new cluster with kind, we use the next command:
```
kind create cluster --config kind/cluster.yaml --name cluster
```
This will create a new kind cluster with the name "cluster".

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
This command tells kubectl to wait for the ingress-nginx process to start all the pods correctly, and sets a timeout of 90s in case it fails to do so.
```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

This command will fetch the config necessary to spawn a pod that will be only to collect metrics from the slave machine, so the auto-scaling can work correctly.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
``` 

This command will open the config file for the metrics-server and we will add a few options that are specific to kind.
```
kubectl edit deployment metrics-server -n kube-system
```
Because we use kind, we need to specify a few options to disable TLS, since we are not in a production environment, the option for "InternalIP" is necessary since kind is *not* actually a true deployed system (it emulates kubernetes locally).
```
--kubelet-insecure-tls
--kubelet-preferred-address-types="InternalIP"
```

The next command applies the kubernetes configuration files, located in k8s/ recursively. It will not stop currently running services that were started with a prior configuration. 
```
kubectl apply -f k8s/ -R
```


# Generate load

We can generate an artificial load on the system with this command: 
```
while sleep 0.01; do wget -q -O- http://localhost/foo > /dev/null; done
```

# Delete

If we want to "shutdown" the system, we delete the cluster, which in turn will delete all running pods.
```
kind delete cluster --name cluster
```

# Authors
* **[etoome](https://github.com/etoome)**
* **[alfduque](https://github.com/alfduque)**
* **[sebaarte](https://github.com/sebaarte)**
* **[vpiryns](https://github.com/vpiryns)**
* **[tlissenk](https://github.com/tlissenk)**
