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
- long running object in k8s - used to expose functionality of pods via ports
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
kubectl describe rs <rs-name>
```

### Deployments
- automatic rolling updates with zero downtime - always 1 healthy pod in existence
- able to rollback previous deployment versions
```bash
# to get rollout status
kubectl rollout status deployment <deploy-name>

# revision history
kubectl rollout history deploy <deploy-name>

# rollback
kubectl rollout undo deploy <deploy-name> --to-revision=2
```

### K8s Networking 
- two containers must be networked together; eg. if 2 containers in a pod, then it is connected via a localhost
- k8s has its own private DNS service - db containing set of KVPs (keys = labels/names; values = IP address of k8s services)
- `kube-dns` - service that's always running in the background for k8s; has DNS container
- namespace - similar to a python package, in terms of partitioning resources
- k8s NS auto-setup include `kube-sytem` && `kube-public` - used internally by k8s
- file called `resolv.conf` - configures how DNS name resolutions; needs to find DNS nameserver <IP address>
- Service Discovery - find any IP address of service via invoking its name
```bash
kubectl get all -n kube-system
```