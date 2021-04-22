# Kubernetes for Beginners

## Kubernetes Concepts

Kubectl
Run:
kubectl run nginx --image=nginx

### Demo
```
minicube start

kubectl get pods

kubectl get pods -A

kubectl run nginx --image=nginx

watch kubectl get pods

kubectl describe pod nginx

kubectl get pods -o wide
```

## YAML Introduction

[Here](https://onlineyamltools.com/convert-yaml-to-json) is a great yaml to json converter.

### Example 1
```yaml
Fruits:
  - Apple:
        Calories: 95
        Fat: 0.3
        Carbs: 25
  - Banana:
      Calories: 105
      Fat: 0.4
      Carbs: 27
  - Orange:
        Calories: 45
        Fat: 0.1
        Carbs: 11

Vegetables:
  - Carrot:
        Calories: 25
        Fat: 0.1
        Carbs: 6
  - Tomato:
      Calories: 22
      Fat: 0.2
      Carbs: 4.8
  - Cucumber:
        Calories: 8
        Fat: 0.1
        Carbs: 1.9
```
is equivalent to
```json
{
  "Fruits": [
    {
      "Apple": {
        "Calories": 95,
        "Fat": 0.3,
        "Carbs": 25
      }
    },
    {
      "Banana": {
        "Calories": 105,
        "Fat": 0.4,
        "Carbs": 27
      }
    },
    {
      "Orange": {
        "Calories": 45,
        "Fat": 0.1,
        "Carbs": 11
      }
    }
  ],
  "Vegetables": [
    {
      "Carrot": {
        "Calories": 25,
        "Fat": 0.1,
        "Carbs": 6
      }
    },
    {
      "Tomato": {
        "Calories": 22,
        "Fat": 0.2,
        "Carbs": 4.8
      }
    },
    {
      "Cucumber": {
        "Calories": 8,
        "Fat": 0.1,
        "Carbs": 1.9
      }
    }
  ]
}
```

### Example 2
```yaml
Employee:
  Name: Jacob
  Sex: Male
  Age: 30
  Title: Systems Engineer
  Projects:
    - Automation
    - Support
  Payslips:
    - Month: June
      Wage: 4000
    - Month: July
      Wage: 4500
    - Month: August
      Wage: 4000
```
is equivalent to
```json
{
  "Employee": {
    "Name": "Jacob",
    "Sex": "Male",
    "Age": 30,
    "Title": "Systems Engineer",
    "Projects": [
      "Automation",
      "Support"
    ],
    "Payslips": [
      {
        "Month": "June",
        "Wage": 4000
      },
      {
        "Month": "July",
        "Wage": 4500
      },
      {
        "Month": "August",
        "Wage": 4000
      }
    ]
  }
}
```

## Kubernetes Concepts - PODs, ReplicaSets, Deployments

### PODs with YAML

Kubernetes uses yaml to represent objects, such as pods, replicas, deployments, services, et cetera. They all follow a similar structure. A Kubernetes definition file always contains 4 required top level fields:
- `apiVersion`
- `kind`
- `metadata`
- `spec`

#### Demo
Create a pod definition file:
```yaml
# pod-definition.yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
```

```sh
# Create the pod from the definition file (`create` or `apply` could be used interchangably)
kubectl create -f pod-definition.yml
# View it in the list of pods
kubectl get pods
# Get details on pod
kubectl describe pod nginx
```

### Tips & Tricks - Developing Kubernetes Manifest files with Visual Studio Code

Recommended editor: [VS Code](https://code.visualstudio.com/)

Recommended extension: [YAML by Red Hat](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)

### Replication Controllers and ReplicaSets

Replication Controllers and ReplicaSets serve the same purpose, but the former is being phased out and being replaced by the latter.

Their job is to ensure the right number of pods are running at all times.

Example *replication controller* definition:
```yml
# rc-definition.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  template:
    # Copied from pod definition
    metadata:
      name: nginx
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
  replicas: 3
```

```sh
# Expect no results yet
kubectl get replicationcontrollers
# Create the above replication controller
kubectl create -f rc-definition.yml
# Expect one result
kubectl get replicationcontrollers
# Expect three `myapp` pods
kubectl get pods
```

Example *ReplicaSet* definition:
```yml
# replicaset-definition.yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  template:
    # Copied from pod definition
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  # Selector is the main difference from replication controller
  selector:
    matchLabels:
      type: front-end
```

```sh
# Expect no results
kubectl get replicasets
# Create the above ReplicaSet
kubectl create -f replicaset-definition.yml
# Expect one results
kubectl get replicasets
# Expect two new `myapp` pods, plus the one existing one named `myapp-pod`
kubectl get pods
```

#### Scaling ReplicaSets

Any of these three commands would scale the replicas from 3 to 6.
```sh
# Enter `replicas: 6` in the replicaset-definition.yml file first
kubectl replace -f replicaset-definition.yml
```
```sh
kubectl scale -f replicaset-definition.yml --replicas=6
```
```sh
kubectl scale replicaset myapp-replicaset --replicas=6
``` 

#### Review commands

```sh
# Create ReplicaSet from definition file (could also use `apply`)
kubectl create -f replicaset-definition.yml
# Show active ReplicaSets
kubectl get replicasets
# Delete replicaset and all underlying pods
kubectl delete replicaset myapp-replicaset
# Update ReplicaSet from definition file
kubectl replace -f replicaset-definition.yml
# Scale ReplicaSet
kubectl scale --replicas=6 -f replicaset-definition.yml
```

### Deployments

A deployment definition looks exactly like a replica set definition except the `kind` is now `Deployment`. Creating a new deployment will create a cascade including a new replica set and new pods.

#### Deployments Demo

Deployment definition:
```yaml
# deployments/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels: 
    tier: frontend
    app: nginx
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 3
  template:
    metadata:
      name: nginx-2
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx

```

```sh
# Create deployment from definition
kubectl create -f deployments/deployment.yaml
# View deployment
kubectl get deployments
# View replica set created by deployment
kubectl get replicasets
# View pods created by deployment
kubectl get pods
# View deployment details
kubectl describe deployment myapp-deployment
# View everything, including the cluster, deployment, replica set, and pods
kubectl get all
# Delete deployment
kubectl delete deployment myapp-deployment
```

### Update and Rollback

There are two udeployment strategies, recreate and rolling. Recreate will take down all running instances and then recreate them, causing downtime in between. In contrast, rolling will scale up one replacement pod at a time, and will only take down an old pod once the replacement is running succesfully. Thankfully rolling is the default.

Under the hood a new replica set is created, then one pod is scaled up in the new replica set, then one pod is scaled down in the old replica set, proceeding in this rolling fashion untill all are replaced.

```sh
# Create original deployment
kubectl create -f deployment-definition.yml
# View deployment
kubectl get deployments
# Apply changes to original deployment
kubectl apply -f deployment-definition.yml
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
# View rollout current status
kubectl rollout status deployment/myapp-deployment
# View rollout history
kubectl rollout history deployment/myapp-deployment
# Roll back a deployment
kubectl rollout undo deployment/myapp-deployment
```

#### Update & Rollback Demo

Using the same deployment from the previous demo, but changing the number of replicas to 6.

```sh
# Create the new deployment and immediately check the rollout status to watch them all roll out
# The --record flag will record the cause of the change in the rollout history
kubectl create -f deployments/deployment.yaml --record \
  && kubectl rollout status deployment/myapp-deployment
# View the rollout history
kubectl rollout history deployment/myapp-deployment
# Edit deployment and watch rolling update
kubectl edit deployment myapp-deployment --record
kubectl rollout status deployment/myapp-deployment
# Describe deployment
kubectl describe deployment myapp-deployment
# Edit deployment and watch rolling update
kubectl set image deployment myapp-deployment nginx=nginx:1.18-perl --record
kubectl rollout status deployment/myapp-deployment
# Undo the last update
kubectl rollout undo deployment myapp-deployment
# View the rollout history
kubectl rollout history deployment/myapp-deployment
# Edit the deployment image to `nginx:1.18-does-not-exist`
edit deployment myapp-deployment --record
# Notice that the update will stall out on the first pod with an ImagePullBackOff error
kubectl rollout status deployment/myapp-deployment
kubectl get pods
# Revert the latest bad change
kubectl rollout undo deployment myapp-deployment
```

## Networking in Kubernetes

Within a Kubernetes cluster, each node will have an IP address. When Kubernetes is configured an internal private network is created on each node with the address 10.244.0.0. Within a node each pod gets its own internal IP in the range 10.244.0.x.

Kubernetes expects us to configure our own networking such that:
1. All containers/pods can communicate to one another without NAT
2. All nodes can communicate with all containers and vice-versa without NAT

There are several third party solutions to this so you don't need to roll your own. Your routing solution will assign unique IP addresses to all nodes and pods so you can communicate directly with each.

## Services

### NodePort

Kubernetes services are kubernetes objects, just like pods, replicasets, & deployments. They help us connect applications together with other applications or users. For example, services enable connectivity between goups of pods, such as frontend to users, frontend to backend, and backend to database.

Service types:
1. NodePort: Expose a pod via a port on the node's IP
1. ClusterIP: Creates a virtual IP inside the cluser to enable communication between different apps/pods
1. LoadBalancer: Distribute load across different front end web servers

A NodePort service makes a pod accessible via a port on the node. For example, if the IP address of the node is 192.168.1.2, you could use a service to expose a pod within the node on 192.168.1.2:30008, and could then hit it via `curl http://192.168.1.2:3008`.

Example service definition:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30004
  selector:
    app: myapp
```

Notes:
- `metadata.labels`: Can be assigned labels, even though we didn't in this example.
- `spec.type`: Valid types are NodePort, ClusterIP, or LoadBalancer. If omitted, CluserIP is the default.
- `spec.ports[0].targetPort`: Port exposed on the app. If omitted it will be the same as `port`.
- `spec.ports[0].port`: Port on the service.
- `spec.ports[0].nodePort`: Port exposed on the node to the outside world. Default valid range is 30000-32767. If omitted it will be assigned one.
- `spec.selector`: Selector is used to specify the pod(s) to connect to. If multiple pods match then the service acts as a load balancer, assigning traffic to each randomly.
- The same service can work for each of these scenarios:
  - A single pod on a single node
  - Multiple pods on a single node
  - Multiple pods across multiple nodes. In this case they would be addressed by the IP of the node and the same port (e.g. `192.168.1.x:30008`)

#### Demo

```sh
# Create the service using the service definition above
kubectl create -f services/service.yaml
Kubectl get services
# Get the URL from minikube and visit it in a browser
minikube service myapp-service --url
```

### ClusterIP

Communicating with specific pods by their IP is problematic because they are assigned randomly when they spin up and are lost when they spin down. To resolve this you can create a ClusterIP service that will have a fixed IP and can be used to communicate with specific pods.

Sample definition:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: back-end
spec:
  type: ClusterIP
  ports:
  - targetPort: 80
    port: 80
  selector:
    app: myapp
    type: back-end
```

### LoadBalancer

When pods are split across multiple nodes and then exposed via NodePort, a problem arises where they can be accessed by the multiple IPs for the different nodes (e.g. 192.168.56.70:30035, 192.168.56.71:30035, 192.168.56.72:30035), but you just want one web address for all of them. That's where a load balancer comes in.

A load balancer service would be used with the load balancer from a supported cloud provider, such as AWS or GCP

Sample definition:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  ports:
  - targetPort: 80
    port: 80
    nodePort: 30008
```

## Microservices Architecture

Our example will be a voting app that is comprised of five microservices:
1. Voting app
1. Redis
1. Worker
1. Postgres
1. Result app

### Demo - Deploying Microservices Application on Kubernetes

We will need a pod definition for each of the five microservices. Then we will need a service definition for all but the worker.

```yaml
# voting-app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: voting-app-pod
  labels:
    name: voting-app-pod
    app: demo-voting-app
spec:
  containers:
  - name: voting-app-pod
    image: kodekloud/examplevotingapp_vote:v1
    ports:
    - containerPort: 80
```

```yaml
# redis-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-pod
  labels:
    name: redis-pod
    app: demo-voting-app
spec:
  containers:
  - name: redis
    image: redis
    ports:
    - containerPort: 6379
```


```yaml
# worker-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker-app-pod
  labels:
    name: worker-app-pod
    app: demo-voting-app
spec:
  containers:
  - name: worker-app-pod
    image: kodekloud/examplevotingapp_worker:v1
```

```yaml
# postgres-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  labels:
    name: postgres-pod
    app: demo-voting-app
spec:
  containers:
  - name: postgres
    image: postgres
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_USER
      value: "postgres"
    - name: POSTGRES_PASSWORD
      value: "postgres"
```

```yaml
# result-app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: result-app-pod
  labels:
    name: result-app-pod
    app: demo-voting-app
spec:
  containers:
  - name: result-app-pod
    image: kodekloud/examplevotingapp_result:v1
    ports:
    - containerPort: 80
```

```yaml
# voting-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: voting-service
  labels:
    name: voting-service
    app: demo-voting-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30004
  selector:
    name: voting-app-pod
    app: demo-voting-app
```

```yaml
# redis-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    name: redis-service
    app: demo-voting-app
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    name: redis-pod
    app: demo-voting-app
```

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    name: postres-service
    app: demo-voting-app
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    name: postgres-pod
    app: demo-voting-app
```

```yaml
# result-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: result-service
  labels:
    name: result-service
    app: demo-voting-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30005
  selector:
    name: result-app-pod
    app: demo-voting-app
```

```sh
# Verify that the cluser is empty to start
kubectl get all
# Create the voting app and service
kubectl create -f voting-app-pod.yaml
kubectl create -f voting-app-service.yaml
kubectl get pods,services
# Find and visit the URL
minikube service voting-service --url
# Create all remaining pods and services
kubectl create -f redis-pod.yaml
kubectl create -f redis-service.yaml

kubectl create -f postgres-pod.yaml
kubectl create -f postgres-service.yaml

kubectl create -f worker-pod.yaml

kubectl create -f result-app-pod.yaml
kubectl create -f result-app-service.yaml

kubectl get pods,services
# Get and visit the URLs for the voting and results services
minikube service voting-service --url
minikube service result-service --url
```

### Demo - Deploying Microservices Application on Kubernetes with Deployments

The above setup is a good start, but creating the microservices with pods only is not a good solution. It won't scale, it won't restart crashed pods, and updating versions will be difficult and involve downtime. The right approach is to use deployments and let them manage the pods.

```yaml
# voting-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting-app-deployment
  labels:
    name: voting-app-deployment
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: voting-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: voting-app-pod
      labels:
        name: voting-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: voting-app-pod
        image: kodekloud/examplevotingapp_vote:v1
        ports:
        - containerPort: 80
```

```yaml
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    name: redis-deployment
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: redis-pod
      app: demo-voting-app
  template:
    metadata:
      name: redis-pod
      labels:
        name: redis-pod
        app: demo-voting-app
    spec:
      containers:
      - name: redis
        image: redis
        ports:
        - containerPort: 6379
```

```yaml
# worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-deployment
  labels:
    name: worker-deployment
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: worker-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: worker-app-pod
      labels:
        name: worker-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: worker-app-pod
        image: kodekloud/examplevotingapp_worker:v1
```

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  labels:
    name: postgres-deployment
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgres-pod
      app: demo-voting-app
  template:
    metadata:
      name: postgres-pod
      labels:
        name: postgres-pod
        app: demo-voting-app
    spec:
      containers:
      - name: postgres
        image: postgres
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          value: "postgres"
```

```yaml
# result-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-deployment
  labels:
    name: result-deployment
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: result-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: result-app-pod
      labels:
        name: result-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: result-app-pod
        image: kodekloud/examplevotingapp_result:v1
        ports:
        - containerPort: 80
```

```sh
# Delete all pods and services created in the previous demo
kubectl get all
kubectl delete pods --all
kubectl delete services db redis result-service voting-service
# Create all deployments and services
kubectl create -f voting-app-deployment.yaml
kubectl create -f voting-app-service.yaml

kubectl create -f redis-deployment.yaml
kubectl create -f redis-service.yaml

kubectl create -f postgres-deployment.yaml
kubectl create -f postgres-service.yaml

kubectl create -f result-deployment.yaml
kubectl create -f result-app-service.yaml

kubectl create -f worker-deployment.yaml
# View all pods and services
kubectl get pods,services
# Get and visit the URLs for the voting and results services
minikube service voting-service --url
minikube service result-service --url
# Scale the voting app replicas to 3
kubectl scale deployment voting-app-deployment --replicas=3
# Note that each time you vote it will now be served by a different pod
```

## Kubernetes on Cloud

### Introduction

There are two main ways to deploy a Kubernetes cluster, self hosted or hosted.

Self Hosted:
- You provision VMs
- You configure VMs
- You use scripts to deploy cluster
- You maintain VMs yourself
- E.g. Kubernetes on AWS using kops or KubeOne

Hosted:
- Kubernetes as a Service
- Provider provisions VMs
- Provider installs Kubernetes
- Provider maintains VMs
- E.g. Google Container Engine (GKE)
