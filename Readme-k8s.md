# Kubernetes basics
website to automate redaction requests

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
- 


#### setup
- [Clean Slate UI Mockup](https://docs.google.com/document/d/1gcZnEc2gabD7IOWfASWWbh_p63uYkF31phKdXx6Uc5Q/edit?ts=5ef0f465)
```bash
python3 -m venv env                                                                                
```