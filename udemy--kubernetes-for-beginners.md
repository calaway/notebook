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
```

.

.

.

.

.

.

.

.
