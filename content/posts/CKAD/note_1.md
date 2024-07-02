+++ 
draft = false 
date = 2024-06-28T22:28:49+08:00 
title = 'CKAD筆記 1 - Basic K8S concepts and kubectl command'
description = "CKAD筆記 1 - Basic K8S concepts and kubectl command"
slug = "" 
authors = ["Jacky Cheng"]
tags = ["CKAD", "K8S", "kubectl"]
categories = ["Note"]
externalLink = "" 
series = ["CKAD筆記"]
havetoc = true
+++

[Pod -> Worker Node -> Cluster] <- Master Node

Master Node: responsible for managing worker node, manager the actual orchestration of the node

K8S component

- API server
- etcd: Key-Value store
- kubelet
- Container Runtime: e.g. Docker
- Controller
- Scheduler

## kubectl

basic

- `kutectl run hello--minikube`

- `kubectl cluster-info`

- `kubectl get nodes`

- `kubectl run nginx --image nginx`

- `kubectl get pods`

### pod definition

e.g. `pod-definition.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type:
spec:
  containers:
    - name: nginx-container
      image: nginx
```

some basic kind option with corresponding apiVersion

| kind       | apiVersion |
| ---------- | ---------- |
| Pod        | v1         |
| Service    | v1         |
| Namespace  | v1         |
| ReplicaSet | apps/v1    |
| Deployment | apps/v1    |

for kind, we can also defined our own. see [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

`metadata.name` and `metadata.labels` are fixed key, but k-v in `metadata.labels` is free to be defined.

### operation

- to create pod by yaml: `kubectl create -f pod-definition.yaml`

  - to extract the existing pod info: `kubectl get pod <pod-name> -o yaml > pod-definition.yaml`

- to view the detail of the pod: `kubectl describe pod <pod-name>`

- to delete deployment: `kubectl delete deployment nginx`

- to delete a pod: `kubectl delete pod <pod-name>`

- to edit a existing pod: `kubectl edit pod <pod-name>`

  - only the properties listed below are editable.

    - spec.containers[*].image

    - spec.initContainers[*].image

    - spec.activeDeadlineSeconds

    - spec.tolerations

    - spec.terminationGracePeriodSeconds

## Replication Controller

Purpose

- Create replication of pod, increase a availability.

- Load balancing & Scalability, in-node / cross-node load balancing.

e.g. `rc-definition.yaml`

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
        name: myapp-pod
        labels:
            app: myapp
            type:
    spec:
    containers:
        - name: nginx-container
        image: nginx

  replicas: 3
```

run

```bash
kubectl create -f rc-definition.yaml
kubectl get replicationcontroller
kubectl get pods
```

## Replica Set

similar to replication controller

e.g. `rs-definition.yaml`

```yaml {linenos=table,hl_lines=["1-2"]}
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
        name: myapp-pod
        labels:
            app: myapp
            type:
    spec:
    containers:
        - name: nginx-container
        image: nginx

  replicas: 3
  selector:
    matchLabels:
        type: front-end
```

run

```bash
kubectl create -f rs-definition.yaml
kubectl get replicaset
kubectl get pods
```

If we changed the yaml with some modifications, say:

```yaml
...
-   replicas: 3
+   replicas: 6
...
```

and that we want to replace the original yaml:

```bash
kubectl replace -f rs-definition.yaml
```

the changes is about scaling the replica set, so we can do it in these ways as well:

```bash
kubectl scale --replicas=6 -f rs-definition.yaml
# or
# kubectl scale --replicas=6 <type> <replica-set-name>
kubectl scale --replicas=6 replicaset myapp-rs
```

## Deployment

1. deploy
2. upgrade docker instance (after docker build new image and add new version in te docker registry)
3. rolling update
4. rollback
5. pause the environment
6. resume the environment

e.g. deployment-definition.yaml

```yaml {hl_lines=[2]}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-rs
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
        name: myapp-pod
        labels:
            app: myapp
            type:
    spec:
    containers:
        - name: nginx-container
        image: nginx

  replicas: 3
  selector:
    matchLabels:
        type: front-end
```

run

```bash
kubectl create -f deployment-definition.yaml
kubectl get deployments
# also check the replicaset and pods under the deployment
kubectl get replicaset
kubectl get pods
```

we can run `kubectl get all` to see all the object created at once.

## Namespace

get pods from specific namespace

```bash
kubectl get pod --namespace=<namespace-name>
```

create pod for specific namespace

```bash
kubectl create -f pod-definition.yaml --namespace=<namespace-name>
```

or edit the pod-definition.yaml and run with -f like:

```yaml {hl_lines=[5]}
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev
  labels:
    app: myapp
    type:
spec:
  containers:
    - name: nginx-container
      image: nginx
```

### namespace definition

e.g. `ns-definition.yaml`

```yaml
apiVersion: v1
kind: namespace
metadata:
  name: dev
```

to create namespace named `dev`, run

- `kubectl create -f ns-definition.yaml` or
- `kubectl create namespace dev`

set context so that the default command read another namespace

```bash
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

to get from all namespace, e.g. pod

```
kubectl get pods --all-namespaces
```

#### DNS

K8S will create a DNS entry on service created.

e.g. `db-service.dev.src.cluster.local`

in which:

- `cluster.local`: `local` is the default domain name of the DNS cluster

- `svc`: subdomain for service

- `dev`: name of the namespace

- `db-service`: name of the service

#### computer quota

set compute quota of a namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    request.cpu: "4"
    request.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

## Tips: dry-run & formatted output

`--dry-run` and `-o <format>` flat

### POD

e.g. Create an NGINX Pod

- `kubectl run nginx --image=nginx`

Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)

- `kubectl run nginx --image=nginx --dry-run=client -o yaml`

### Deployment

e.g. Create a deployment

- `kubectl create deployment --image=nginx nginx`

Generate Deployment YAML file (`-o yaml`), without creating it(`--dry-run`)

- `kubectl create deployment --image=nginx nginx --dry-run -o yaml`

Generate Deployment with 4 Replicas

- `kubectl create deployment nginx --image=nginx --replicas=4`

You can also scale deployment using the kubectl scale command.

- `kubectl scale deployment nginx --replicas=4`

Another way to do this is to save the YAML definition to a file and modify

- `kubectl create deployment nginx --image=nginx--dry-run=client -o yaml > nginx-deployment.yaml`

You can then update the YAML file with the replicas or any other field before creating the deployment.

### Service

Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379

- `kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml`

Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:

- `kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml`

Create a pod named httpd using the image httpd:alpine in the default namespace. Then, create a service with type ClusterIP by the same pod name. The target port for the service should be 80

- `kubectl run httpd --image=httpd:alpine --port=80 --expose=true`
