# Kubernetes basics
website to automate redaction requests

### Setup
install [minikube](https://minikube.sigs.k8s.io/docs/start/)
```bash
minikube start
```

### Basic kubectl cmds
```bash
kubectl get po
kubectl get svc
kubectl exec -it -- /bin/bash
kubectl get -po --show-labels
```

### Pods
- basically, a wrapper for a container
- not meant to be accessed from the frontend
- not visible from outside the cluster
- can be accessed by certain commands
```bash
kubectl exec -it <pod-name> -- /bin/bash
``` 
- ephemeral lifecycle
- usually 1 container per pod

### Services
- long running object in k8s
- has IP address, stable fixed ports
- used as a bridge by the pod to connect to the outside world
- use `app` label as a kvp to connect to a pod
- if using Desktop Docker, you will have to "tunnel", and keep it running in background
-follow URL: `http://127.0.0.1:53423`
```bash
minikube svc <service-name> --url
```

### ReplicaSets
- used to wrap pod in order to automatically restart a failed pod
- writing a replicaset yaml also includes the pod yaml definition
- running a `kubectl describe` on your rs will see a log of everytime it spins up the pod
- there will be a short downtime between crashed pod and rs recreating the pod
- architecture question: does it make sense for your app to have multiple rs for each microservice? 
```bash
kubectl describe rs webapp-replicaset
```
