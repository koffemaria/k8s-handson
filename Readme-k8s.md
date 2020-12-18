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
- Bonus: how to access a live EKS cluster without using a LoadBalancer?
    - NodePort FAQs:
        1. How do we know which particular Node the pod is running on? 
            - k8s built-in Proxy service redirects traffic to active node to handle request
            - if a node crashes, a new one spins up with a completely different IP address
        2. What to do about security?
            - EKS cluster uses **Security Groups**, which limits incoming traffic to a node by specific ports
            - can be modified in AWS EC2 UI
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

### Requests & Limits
- Requests can be memory request or cpu request
- Limits
    - goal of Limits are to protect overall health of a cluster by **limiting** its resources
    - acts as a safety net
    - Memory
        - if actual Memory usage of the container at runtime exceeds the specified limit, the container will be killed.
        - the Pod will remain and the container will be restarted
    - CPU
        - if actual CPU usage of the container at runtime exceeds the specified limit, the CPU is throttled
        - the container will continue to run 
- Requests
    - Memory
        - inserted within `containers` definition
        - `memory` does not affect pod runtime; merely allocating how much RAM is used from the Node
    - CPU
        - inserted within `containers` definition
        - 1 CPU = 1 AWS virtual CPU
        - allocate this much CPU to the container, but not necessarily use all of it
- Node details
    - Capacity - check total `memory` of host machine's RAM
    - Allocatable - 
```yaml
#sample Deployment.yaml
kind: Deployment
...
spec:
  template:
    spec:
      containers:
        - name: my-container
          image: $(REPO)/$(IMAGE):$(TAG)
          resources:
            requests:
              memory: 300Mi # 1Mi = 1024Ki; 1Ki = 1024 bytes 
              cpu: 100m # 100m = 1/10 of a cpu
            limits:
              memory: 500Mi
              cpu: 200m
```
```bash
kubectl get nodes
kubectl describe node <my-node-name>

```

### Replication & Autoscaling
- Horizontal pod autoscaling - adding more nodes to cluster
- Vertical pod scaling - increase cpu power of nodes
- use cpu requests to trigger horizontal autoscaling - use horizontal pod autoscaler (HPA)
    - rule: if actual cpu usage of pod x exceeds > 50% on avg of cpu requests (this metric is determined using metric server)
    - must setup individual rules for each deployment
- `--cpu-percentage` 
    - % amt that triggers the autoscale, can be > 100%
    - % is relative to the request
    - eg. if cpu = 50m, and avg is > 25m, then autoscale triggers
- `--min` and `--max`
    - min and max of autoscaled nodes
```bash
# if minikube, ensure metrics-saver is enabled
minikube addons list
minikube addons enable metrics-saver

# create autoscale
kubectl autoscale deployment api-gateway --cpu-percent 400 --min 1 --max 4
kubectl get hpa
```

### Readiness and Liveness Probes
- Problem 
    - your pod might be in a "running" state, but your application inside is still starting up and not able to receive Requests.
    - Requests can timeout, causes instability and a downtime appearance for users
- Solution 1 - Readiness Probe
    - add k8s config to **Deployment.yaml** to test if a pod is ready to receive Requests
        - add `readinessProbe` under `containers` for **api-gateway**
    - K8s will not send requests to the pod until it is ready to receive Requests
- Solution 2 - Liveness Probe
    - similar to Readiness Probe, but will continue to run for the duration for the pod's lifetime
    - if LP fails, k8s will kill and restart the pod
- Files modified
    - workload-aws.yaml

### Ingress Controllers
- Application Load Balancers
    - used to configure routing rules
    - if incoming domain name has a particular pattern, then route to a particular DNS service
- Ingress Controllers 
    - Service that makes routing decisions based on domain name request by end user, 
        it will point to a configured Service
    - avoids multiple ALB for each Service you have
- create SHA1 pawssword using an online **Htpasswd Generator**; create secret from it
- K8s [nginx basic-auth secret](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/#basic-authentication)
- AWS
    - download [NLB AWS installation](https://kubernetes.github.io/ingress-nginx/deploy/#aws) and apply to cluster
    - after installation, check `Loadbalancers` in EC2 UI
    - ensure switch service type `LoadBalancer` -> `ClusterIP` for **fleetman-webapp**
    - edit local `/etc/hosts` and add IP and domain
        - alternatively, you can use AWS Route53 to create a DNS to link directly to your ALB
- Yaml files created/mod and applied:
    - ingress-secure-aws.yaml
    - ingress-public-aws.yaml
    - fleetman-secrets.yaml
    - services-aws.yaml
    
```bash
# create a basic-auth Secret
kubectl create secret generic my-secret-name --from-file auth

# check annotations on your ingress
kubectl describe ingress secure-ingress -o yaml

# download and apply AWS NLB installation
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/aws/deploy.yaml
kubectl apply -f deploy.yaml

# check new ingress-controller created; EC2 UI should show too
kubectl get all -n ingress-nginx

# test new ingress-nginx loadbalancer
kubectl get svc -n ingress-nginx
nslookup aad80c995338f47c9bb8340c8372b4a2-f83e88ffef150571.elb.us-east-1.amazonaws.com
# get IP address under Non-authoritative answer: 34.205.149.250
# add IP and url to /etc/hosts
``` 

### ConfigMaps and Secrets
- ConfigMap
    - object used to store non-confidential data in key-value pairs that can be shared across multiple pods
    - allows setting env variables, cli args, for config files in a volume
    - pod does not do rolling updates when a new env var set on CM, until pod is killed/restarted
- 3 ways to reference CM in deployment
    1. `env` -> `valueFrom`
    2. `envFrom` - cleaner code
    3. `volumeMount` - all values on configMap is available as separate files in a specified dir within the pod
        - creates database properties file
- Secrets
    - object used to store sensitive info; usually encoded in BASE64 
    
```bash
kubecetl get cm 
kubectl describe cm global-database-config -o yaml

# manually check by ssh into pod
kubectl exec -it position-simulator -- /bin/bash
echo $DATABASE_URL
```