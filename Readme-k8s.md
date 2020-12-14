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
- type of `ClusterIP` - can only make calls within cluster, nothing else
-follow URL: `http://127.0.0.1:53423`
```bash
minikube service <service-name> --url
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

### Microservices
-  Microservices best practice principles: 
    - highly cohesive - each microservice should be handling 1 req or should have single set of responsibilities
    - loose coupling - less dependencies between each application
- **fleetman-queue** service
    - user: admin ; pw: admin
- **fleetman-position-tracker** service 
    - append to tunnel url `~/vehicles/City%20Truck`


### Persistence
- When a pod is destroyed, all data contained is lost
- able to store data in a Persistent volume 
    - eg. file stored in host system, or hard drive in cloud env (eg. AWS EBS or "HD on the cloud")
- Deployment **position-tracker**
    - position data stored in memory (Springboot)
    - issue #1 - it will run out of RAM eventually
        - solution - use an external db like mongo to store data
    - issue #2 - data is stored in mongodb container, if pod dies so will container and all data is lost
        - solution - use a [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes) 
            to mount to store data in local host system (hostPath)
- **Mount** a dir - store data from a container to a dir in host system (local); data for that dir is stored outside the container
- [**hostPath**](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) 
    - represents a pre-existing file/dir on the host machine that is exposed to the container
    - useful for local dev working before moving to cloud platform
    - more [types of PV](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)
- **Binding** 
    - at runtime, PVC will be satisfied with PV by matching *storageClassName* 
    - mongodb pod will bind with this

```bash
kubectl get pv
# NAME=local-storage; STATUS=Bound; CLAIM=mongo-pvc

kubectl get pvc
# NAME=mongo-pvc; STATUS=Bound; VOLUME=local-storage
```


### Running in AWS
- Node
    - physical local server in the system; eg. VM/Docker for minikube
    - uses several nodes in the server in cloud; eg. AWS EC2 instance
    - Worker Node's job is to run the k8s pods 
    - W. Nodes can be designed to be failure-resilient
- Master Node 
    - manages scheduling of lower nodes
    - if using multiple worker nodes, a ReplicaSet can be deployed by master node in another surviving wNode
    - `kubectl` cmds will be run on M Node
    - EKS - AWS will run the mNode; AWS will spin up a new mNode whenever it crashes
        - integrates with Fargate - allows to run containers with "no servers"

### AWS EKS setup
- setup ec2 instance; ssh into ec2 instance
- ip address found in AWS EC2 UI, under *IPv4 Public IP*
- AWS setup steps
    1. setup EC2 instance
    2. ssh into instance to install `aws` && `eksctl` && `kubectl`
        - ensure kubectl and eks versions match
    3. setup IAM group and user roles/perms; create custom policy if needed
    4. create cluster using eksctl cmd
        - *will incur ongoing costs!*
        - creating takes ~20 mins
        - check eks clusters & ec2 nodes instances in AWS UI
    5. ensure to delete cluster when done

```bash
ssh -i bootstrap-dev-ko.pem ec2-user@3.86.221.248
aws eks list-clusters
eksctl create cluster --name my-first-cluster --nodes-min=3
kubectl get all
```

### AWS EKS operations
- everything in this section is done **within EKS instance**
- deploy `workloads.yaml` to cluster
- mount new volume to store mongo data
    - use `StorageClass` for EBS storage - only used by aws when needed
- AWS UI - notice...
    - new EKS cluster
    - EC2 Volumes w/ 3 new nodes
    - EC2 LoadBalancer - find DNS name and paste in browser url
- Route 53 - AWS DNS service; create record eg. k8s-tutorial-ko.2u.com
- Node failure - if you want web ui without downtime
    - increase **replicaSet** from `1` -> `2`
    - ensure pod/app is **stateless** or does not contain data
    - `queue` and `mongo` pods are **stateful**
- AWS MQ
    - config MQ instance, then re-engineer code to MQ
    - aka. shift responsibility of stateful pods to 3rd party
- Delete AWS resources
    - delete cluster from shell; takes ~5 mins
    - EC2 UI
        - check LBs && Instances are deleted
        - delete `k8s-dynamic-pvc` volume;
        - delete EBS `Bootstrap` volume; will incur small costs
        - `stop` bootstrap instance - keeps all k8s resources
        - to restart, run cmd `eksctl create cluster ...`
    -EKS UI
        - check Clusters are deleted
- fleetman-webapp UI
    - use loadbalancer url: eg. `http://a081ea3501c0f4a58918c0989781e3dd-2008835085.us-east-1.elb.amazonaws.com`
```bash
# inside ec2 instance; via ssh
eksctl get cluster
nano storage-aws.yaml
# cmd-o to write, then cmd-x
kubectl apply -f storage-aws.yaml
kubectl get pvc
kubectl get pv

# how to know which node your pod is currently running on?
kubectl get pods -o wide

# to delete cluster
eksctl delete cluster k8s-tutorial-ko
```

### Logging
- ELK or ElasticStack
    - distributed logging between multiple containers in single node
    - use multiple software components
        1. *E*lastic search - distributed search and analytics engine, store in db and query using REST API
        2. *L*og container installed in node to gather all logs (eg. Logstash or Fluentd)
        3. *K*ibana - visualize elastic search data
- Architecture
    - **Fluentd** installed in each node
    - **Kibana** and **Elastic search** can be installed on any node or instance
    - ES use >1 replicaSet
    - Kibana - webapp
    - deployed in `kube-system` ns
- `fluentD-config.yaml`
    - determines how logs files should be processed
- `elastic-stack.yaml`
    - DaemonSet 
        - similar to replicaSet, but doesnt need to specify #; it will run in all nodes
        - ref to FluentD docker image
    - StatefulSet
        - pods created will have set name (metadata-name)
        - `VolumeClaimTemplates` - stores in temp dir
- to simulate app failure - mod to `replicas: 0` for Deployment **queue**
```bash
kubectl get all -n kube-system
```

### Monitoring
- AWS comes with basic free monitoring of EC2 instances/nodes
- Prometheus - external monitoring
    - gather metrics from a source (cluster)
    - integrate with Grafana dashboards
- Helm - package manager for k8s
- more K8s resources
    - CustomResourceDefinition - extend k8s to add further objects defined in yaml
- k8s ops
    - must apply **crds.yaml** first before **eks-monitoring.yaml**
    - Prom: edit **type** of svc `monitoring-kube-prometheus-prometheus` from ClusterIP -> LoadBalancer
        - Prometheus UI: go to url of LB+9090 TCP/IP (eg. `http://1234-2008835085.us-east-1.elb.amazonaws.com:9090`)
        - *WARNING* - will incur costs; change back to ClusterIP && delete `nodePort`
    - Grafana: edit **type** of svc `monitoring-grafana` from ClusterIP -> LoadBalancer 
- some important metrics
    - `node_load15` - Prometheus constantly sampling nodes/workers for CPU usage or load
    - `node_load1` - load over last 1 min
    - `node_load5` - load over last 5 min
    - USE Method
        - *U*tilization - how much is resource being used on avg?
        - *S*aturation - how much is resource being overloaded or unable to handle?
        - *E*rrors - how many errors?
```bash
kubectl apply -f crds.yaml
kubectl apply -f eks-monitoring.yaml

# 10 new monitoring pods added
kubectl get pods -n monitoring

# 8 new services added
kubectl get svc -n monitoring

# edit type: ClusterIP -> LoadBalancer
kubectl edit svc monitoring-kube-prometheus-prometheus -n monitoring
```

### Ingress Controllers
- Application Load Balancers
    - used to configure routing rules
    - if incoming domain name has a particular pattern, then route to a particular DNS service
- Ingress Controllers 
    - Service that makes routing decisions based on domain name request by end user, 
        it will point to a configured Service
    - avoids multiple ALB for each Service you have