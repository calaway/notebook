# Certified Kubernetes Application Developer Course

## Core Concepts

### Kubernetes Architecture

A node is a machine, physical or virtual, on which Kubernetes is installed. To ensure resiliency, you'll want multiple nodes to avoid interruptions if one goes down. A cluster is a set of nodes grouped together.

Each cluster has one special control plane node (formerly referred to as the master node) that is in charge of scheduling pods on nodes and other such functions.

#### Components of Kubernetes

Control Plane Node:
- **kube-apiserver**: The front end for Kubernetes that allows the user to interact with the cluster.
- **etcd**: Higly available data store to store the state of the cluster.
- **controller**: The "brain" that monitors nodes and responds when they go down, among other tasks.
- **scheduler**: Assigns containers to nodes.

Worker nodes:
- **kubelet**: The agent that runs on all clusters that makes sure the containers are running on the nodes as expected.
- **container runtime**: Underlying software that is used to run containers.

#### kubectl

`kubectl` is the CLI provided to interact with the cluster. A few example commands are:
- `kubectl run hello-minikube`: Deploy an application to the cluster.
- `kubectl cluster-info`: View information about the cluster.
- `kubectl get nodes`: List all nodes running on the cluster.

### Pods

Assume the application we are attempting to run is already built and running in a docker container, and is available from a docker repository, such as docker hub. Also assume that the cluster is up and running with at least one node.

When Kubernetes deploys a container, it encapsulates it in an abstraction known as a pod. A pod is a single instance of an application and is the smallest object you can create on Kubernetes.

There is a 1:1 relationship between an instance of an application and the pod. When an application is scaled up to have a replica, the replica is _not_ placed in the same pod as the first. Instead a new pod is created and the replica runs in the second pod.

It is possible to run multiple tightly coupled containers within the same pod, such as an app and a redis instance for the app.

### Demo - Creating pods with yaml

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: nginx-container
      image: nginx
```

```sh
# Make sure no resources are currently running
kubectl get all
# Create the pod from the definition file
kubectl create -f pod-definition.yaml
# View the running pod
kubectl get pods
# Delete the pod
kubectl delete pod myapp-pod
# Create a new pod from the CLI without a definition file
kubectl run redis --image redis
```

### Namespaces

Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces.

A service can communicate to another service in its own namespace by referring to it only by the service name, e.g.
```python
mysql.connect("db-service")
```
A service can also communicate to another service from a different namespace, say `dev`, in the form
```python
mysql.connect("db-service.dev.svc.cluster.local")
```
To break that down:
- `cluster.local` is the domain name of the cluster
- `svc` is the subdomain for service
- `dev` is the namespace
- `db-service` is the service name

To create a pod in a given namespace, you can specify it via CLI
```sh
kubectl create pod -f pod-definition.yaml --namespace dev
# or
kubectl run redis --image redis -n dev
```
Or to make it more permanent it can be specified in the pod definition
```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev # designate namespace here
  labels:
    app: myapp
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Like any other Kubernetes object, namespaces can be created from a definition file.

```yaml
# namespece-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```sh
# Create a new namespace via CLI
kubectl create namespace dev
# List all pods in the dev namespace
kubectl get pods --namespace=dev
# Or
kubectl get pods -n dev
# Set a different namespace as the default
kubectl config set-context $(kubectl config current-context) --namespace=dev
# List all pods in the dev namespace
kubectl get pods
# List all pods in the default namespace
kubectl get pods --namespace=default
# List all pods in all namespaces
kubectl get pods --all-namespaces
```

To limit resources in a namespace, create a resource quota.

```yaml
# compute-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

### Certification Tip: Imperative Commands

Kubernetes is meant to be used declaratively, such as through definition files. However, a few imperative commands can help save a considerable amount of time during the exam.

Instead of writing a definition file from scratch, here is an example that will create the boilerplate for you, then you can modify it from there.
```sh
kubectl run redis --image=redis --dry-run=client -o yaml > pod-definition.yaml
```

Quickly create an nginx deployment with four replicas
```sh
kubectl create deployment nginx \
  --image=nginx \
  --replicas=4 \
  --dry-run=client \
  -o yaml \
  > nginx-deployment.yaml
```

Generate a service. Note that this will not specify the node port, so you will need to add it manually.
```
kubectl expose pod nginx \
  --port=80 \
  --name nginx-service \
  --type=NodePort \
  --dry-run=client \
  -o yaml
```
Alternatively, this will not use the pod's labels as selectors.
```
kubectl create service nodeport nginx \
  --tcp=80:80 \
  --node-port=30080 \
  --dry-run=client \
  -o yaml
```
The course author recommends the former.

Generator reference: https://kubernetes.io/docs/reference/kubectl/conventions/#generators

## Configuration

### Pre-Requisite - Commands and Arguments in Docker

Docker containers are only meant to run the command they are given and then exit as soon as the execution completes. The default command that will be executed is specified by the `CMD` command in the Dockerfile. For example, in the nginx image from Docker Hub, the final line is
```dockerfile
CMD ["nginx"]
```

The default command can be overridden at the end of the `docker run` command. For example, this will sleep for 5 seconds in an Ubuntu container and then exit
```sh
docker run ubuntu sleep 5
```

If you want to make `sleep 5` the default command, you can build a new image with that command baked in
```dockerfile
FROM Ubuntu

CMD ["sleep", "5"]
```
```sh
# Build the image
docker build -t ubuntu-sleeper .
# Run the image
docker run ubuntu-sleeper
```

If you want to use a default command, but be able to pass in an argument, use `ENTRYPOINT`. If you also want a default argument if one is not given at runtime, use `ENTRYPOINT` along with `CMD`
```dockerfile
FROM Ubuntu

ENTRYPOINT ["sleep"]

CMD ["5"]
```
```sh
# Sleep for 5 seconds and exit
docker run ubuntu-sleeper
# Sleep for 10 seconds and exit
docker run ubuntu-sleeper 10
# Override the entrypoint
docker run --entrypoint sleep2.0 ubuntu-sleeper 10
```

### Commands and Arguments in Kubernetes

Given the same Dockerfile from above, with an entrypoint of `sleep` and a default command of `5`, you can override the `CMD` via `spec.containers[0].args` and you can override `ENTRYPOINT` via `spec.containers[0].command`.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: ["sleep2.0"]
      args: ["10"]
```
Remember that counterintuitively, `command` does **not** override `CMD`.

### Environment Variables

To set hard-coded env vars in a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      env:
        - name: APP_COLOR
          value: pink
```

### ConfigMaps

Managing env vars in pod definition files can become cumbersome. You can alternatively manage them in a ConfigMap Kubernetes object, and then reference that from the pod. First create the configmap, and second inject it into the pod. Just like any Kubernetes object, a ConfigMap can be created imperitively, via the CLI, or declaratively, via a definition file.

#### Imperative:
```sh
# Get help
kubectl create configmap -h
# Create a new configmap named app-config with two env vars
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MOD=prod
# Create a new configmap from a file
kubectl create configmap app-config \
  --from-file=app_config.properties
# List all configmaps
kubectl get configmaps
# Describe the configmap
kubectl describe configmap app-config
```

#### Declarative:
```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data: # Note: data, not spec
  APP_COLOR: blue
  APP_MODE: prod
```

#### Inject a configmap into a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      envFrom:
        - configMapRef:
            name: app-config
```

### Secrets

Secrets are similar to a configmap, in that they are key/value pairs, except that the values are encrypted, making them useful for sensitive information.

To use a secret you first need to save it in a secrets object, and then inject it into a pod.

#### Create a secret imperatively:
```sh
# Get help
kubectl create secret generic -h
# Create a secret named my-secret with three key/value pairs
kubectl create secret generic my-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswrdz
# Create a secret with a key of the file name and a value of the file contents
kubectl create secret generic my-secret \
  --from-file=app_secret.properties
# Create a secret with a key of the file name and a value of the file contents
kubectl create secret generic my-secret \
  --from-file=app_secret.properties
```

#### Create a secret declaratively
```sh
# First base64 encode the secret, so it isn't revealed in plain text
echo -n 'mysql' | base64 # bXlzcWw=
echo -n 'root' | base64 # cm9vdA==
echo -n 'paswrd' | base64 # cGFzd3Jk
# Alternatively, base64 encode from the clipboard, so the secret isn't saved to your history
pbpaste | base64
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

#### Get secrets
```sh
# List secrets
kubectl get secrets
# Describe a secret, without revealing values
kubectl describe secret my-secret
# Show secret with base64 encoded values
kubectl get secret my-secret -o yaml
# Base64 decode a string
echo -n 'cGFzd3Jk' | base64 --decode # paswrd
# Pull a specific value from a secret and base64 decode it
kubectl get secret my-secret -o=jsonpath='{.data.DB_Password}' | base64 -d
```

#### Inject a secret into a pod:
This will inject all three secrets into the pod as env vars, `DB_Host`, `DB_User`, & `DB_Password`.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      envFrom:
        - secretRef:
            name: my-secret
```
Alternative forms:
```yaml
# Single env
env:
  - name: DB_Password
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: DB_Password
```
```yaml
# Volume
volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```

Pro Tip: If you don't remember the syntax fro the `envFrom` section, you can run `kubectl explain pods --recursive | less` and then search for the command you need (`/envFrom`).

### Security Contexts

To set a security context at the pod level, that will apply to all containers in that pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
```

To set a security context at the container level:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

### Service Accounts

Accounts on Kubernetes can be user or service accounts. User accounts are for humans, such as an admin or developer account. Service accounts are for machines, such as Prometheus or Jenkins.

```sh
# Create a new service account named `dashboard-sa`
kubectl create serviceaccount dashboard-sa
# View all service accounts
kubectl get serviceaccount
# View details on a service account
kubectl describe serviceaccount dashboard-sa
# Get the token created for the service account via secrets
kubectl describe secret dashboard-sa-token-kll64
# Use the token to call the Kubernetes API
curl https://192.168.56.70:6443/api -insecure --header "Authorization: Bearer eyJhbG..."
```

This service account token can then be used by the service to make calls to the Kubernetes API. Alternatively, if the service is running on the same cluster, you can mount the secret on the pod as a volume.

A default service account automatically exists on each namespace. This default account is very restricted and only has authorization to make basic API queries.

```sh
# Create a new pod
kubectl run nginx --image=nginx
# See the default service account on the pod description under Mounts
kubectl describe pod nginx # /var/run/secrets/kubernetes.io/serviceaccount
# View the keys for the secrets as file names
kubectl exec -it nginx -- ls /var/run/secrets/kubernetes.io/serviceaccount # ca.crt	namespace  token
# View the token value
kubectl exec -it nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token # eyJhbGci...
# Delete the service account
kubectl delete serviceaccount dashboard-sa
```

Include your new service account in the pod definition.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
  serviceAccount: dashboard-sa
```

If you want to explicityly not include the default service account, include
```yaml
...
spec:
  automountServiceAccountToken: false
...
```

### Resource Requirements

Any given pod will have certain resource requirements for CPU, memory, and disk space. The scheduler will assign the pod to a node that has sufficient resources. If there is no node with sufficient resources, the scheduler will not be able to assign the pod and it will go into an error state.

You can specify the resources required in the pod definition within the container. In this case we will give the container 1GB memory and 1 CPU.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      resources:
        requests:
          memory: "1Gi"
          cpu: 1
```

1 CPU is equal to:
- 1 AWS vCPU
- 1 GCP core
- 1 Azure core
- 1 hyperthread
CPUs below 1 can be specified as decimals down to 0.1, or as thousandths, such as 100m for 1/10th, as low as 1m.

Memory can be expressed as:
- 1G (gigabyte) = 1,000,000,000 bytes
- 1M (megabyte) = 1,000,000 bytes
- 1K (Kilobyte) = 1,000 bytes
- 1Gi (gibibyte) = 2^10 or 1024 bytes
- 1Mi (mebibyte) = 2^20 or 1048576 bytes
- 1Ki (Kibibyte) = 2^30 or 1073741824 bytes

You can also set limits on resources, such that the container will be throttled if it hits its limit.
```yaml
...
spec:
  containers:
    - resources:
        requests:
          memory: "1Gi"
          cpu: 1
        limits:
          memory: "2Gi"
          cpu: 2
...
```

### Taints and Tolerations

One or more taints can be applied to a node to repel pods from being scheduled on it. One or more tolerations can be applied to a pod, to allow it (but do not require it) to be scheduled on a node with a matching taint. This can be used to "reserve" a node for specific types of pods.

Taints are specified by a key value pair. Tainte effect options are:
1. `NoSchedule`: No new intolerant pods will be scheduled on the node.
1. `PreferNoSchedule`: Don't schedule new intolerant pods, unless there's no room elsewhere. This is a "soft" version of `NoSchedule`.
1. `NoExecute`: No new intolerant pods will be scheduled on the node, and existing intolerant pods will be evicted from the node.
```sh
kubectl taint nodes <node-name> <key>=<value>:<taint-effect>
# Add a taint to a node
kubectl taint nodes node1 app=blue:NoShedule
# See the taint on a node
kubectl describe node node1 | grep -i taint
# Remove a taint from a node
kubectl taint nodes node1 app-
```

To apply a toleration to a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  tolerations:
    # Note that values must be in double quotes
    - key: "app"
      operator: "Equal" # Case sensitive
      value: "blue"
      effect: "NoSchedule"
```
If you forget the keys required, try
```sh
kubectl explain pods --recursive | less
kubectl explain pods --recursive | grep -A 5 toleration
```

### Node Selectors

You can specify which nodes a pod will run on by labeling a node and matching it in the pod definition.
```sh
kubectl label node <node-name> <label-key>=<label-value>
kubectl label node node-1 size=Large
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: Large
```
Node selectors are useful, but limited. For example, there is no way to specify that you want your pod to be scheduled on either a Large or Medium node; or not on a Small node.

### Node Affinity

Node affinity allows for more complex expressions, such as `or` and `not`, but is also more complex to designate. This will do the same thing as the simple node selector above.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
    - name: data-processor
      image: data-processor
  affinity:
   nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - Large
```
For Large or Medium
```yaml
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - Large
                  - Medium
```
For anything but Small
```yaml
          - matchExpressions:
              - key: size
                operator: NotIn
                values:
                  - Small
```
For any node that has any value for `size`
```yaml
          - matchExpressions:
              - key: size
                operator: Exists
```

Affinity types:
1. preferredDuringSchedulingIgnoredDuringExecution
    - Can be scheduled on a node it does not have an affinity for if necessary
    - Will not be evicted if the affinity changes after scheduling
1. requiredDuringSchedulingIgnoredDuringExecution
    - Will not be scheduled if no node exists with a matching affinity
    - Will not be evicted if the affinity changes after scheduling
1. requiredDuringSchedulingRequiredDuringExecution
    - Will not be scheduled if no node exists with a matching affinity
    - Will be evicted if the affinity changes after scheduling

### Taints & Tolerations vs Node Affinity

These two concepts can be combined to give you more control over scheduling. For example, say you have blue, red, and green pods, that you want scheduled on your blue, red, and green nodes. You also want to ensure other pods are not scheduled on these. Give them an affinity for their color to get them scheduled there, then give the nodes taints for their colors to keep others away, then give the three tolerations for their colors so the taints don't keep them away.

## Multi-Container Pods

### Multi-Container Pods

Kubernetes makes running microservices easier, since each service can be managed as its own pod. However, sometimes you may want multiple tightly coupled services to be run together in a pod, such as a web server and a log agent. Multi-container pods share the same lifecycle, so they are scaled up and down together. They also share the same network space, so they can refer to each other as localhost, and they have access to the same storage volumes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
```

There are three common patterns for companion containers to the main service in a pod, sidecar, adapter, and ambassador.

A sidecar would be like the example above, of a log server that collects logs and reports them to a central log server.

An adapter example might be a log adapter, which takes in logs from different microservices in various formats and adapts them to the same format before passing them along to the log server.

An ambassador example could be a container that takes any request to a database on localhost and routes it to the correct database for that environment, i.e. dev, test, or prod.

## Observability

### Readiness and Liveness Probes

Pod lifecyle:
1. Pending: Pod has not been scheduled
1. ContainerCreating: Pod has been scheduled, but is not yet running
1. Running: Pod is succesfully running and will stay in this state until the program completes or is terminated

A problem may occur when Kubernetes has successfully started your container and starts routing traffic to it, but the container is still initializing and is not ready to accept traffic. This can be solved with a readiness probe. If a readiness probe is set, k8s will not route traffic to the pod until it has received a successful response from the test.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      readinessProbe:
        httpGet:
          path: /api/ready
          port: 8080
```

There are different types of readiness tests:

HTTP Test:
```yaml
      readinessProbe:
        httpGet:
          path: /api/ready
          port: 8080
```

TCP Test:
```yaml
      readinessProbe:
        tcpSocket:
          port: 3306
```

Exec Command:
```yaml
      readinessProbe:
        exec:
          command:
            - cat
            - /app/is_ready
```

See additional options with `kubectl explain pods.spec.containers.readinessProbe`, such as:
```yaml
      readinessProbe:
        httpGet:
          path: /api/ready
          port: 8080
        initialDelaySeconds: 10 # Wait a bit longer before ready - default 0
        periodSeconds: 5 # Interval between probes - default 1
        failureThreshold: 8 # Minimum consecutive failures before container is considered failed - default 3
```


### Liveness Probes

A liveness probe is similar to a readiness probe, except whereas a rediness probe is only tested when the container is initially launched, a liveness probe runs the test continuously to monitor the health of the container. If it receives a failed response then it will terminate the container so it can be recreated.

HTTP Test:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      livenessProbe:
        httpGet:
          path: /api/ready
          port: 8080
```

TCP Test:
```yaml
      livenessProbe:
        tcpSocket:
          port: 3306
```

Exec Command:
```yaml
      livenessProbe:
        exec:
          command:
            - cat
            - /app/is_ready
```

See additional options with `kubectl explain pods.spec.containers.livenessProbe`. See example under readiness probes.

### Container Logging

To view logs in Docker, use the `docker logs` command.
```bash
$ docker run -d kodekloud/event-simulator
d0255a5968d2697ebae018f81b506f815fc0c10ad89d1e7f5fc68d54685d3fac

$ docker logs --follow d0255
```

To view logs in Kubernetes, use the `kubectl logs` command.
```
kubectl run event-simulator --image=kodekloud/event-simulator
kubectl logs --follow event-simulator
```

If there are multiple containers in the pod, specify which at the end of the command.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
    - name: event-simulator
      image: kodekloud/event-simulator
    - name: nginx
      image: nginx
```
```
kubectl logs --follow event-simulator-pod event-simulator
```

### Monitor and Debug Applications

Kubernetes does not come with a metrics server built in, but there are multiple open source ones available. In this example we'll look at [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

```sh
# To start metrics server on Minikube
minikube addons enable metrics-server
# Without Minikube
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# See node performance metrics
kubectl top node
# See pod performance metrics
kubectl top pod
```

### Labels, Selectors, and Annotations

Labels are used to put an object into one or more groups. Selectors are used to identify objects or groups of objects by their labels.

Add labels under `metadata`.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
```

Get objects using selectors via kubectl
```sh
# --selector and -l are synonymous
kubectl get pods --selector app=App1
kubectl get pods -l app=App1
# Show all pods with their labels
kubectl get pods --show-labels
```

In this example, the `App1` label is used by the deployment to select the pod that it is creating. Multiple selectors could be used if one is not specific enough.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      name: simple-webapp
      labels:
        app: App1
        function: Front-end
    spec:
      containers:
        - name: simple-webapp
          image: simple-webapp
```

Annotations can be used to add any additional arbitrary data, usually for the purpose of integrating with some other system.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-webapp
  annotiations:
    buildversion: 1.34
spec:
  ...
```

### Rolling Updates & Rollbacks in Deployments

Rollout history is kept so you can roll back to a previous working version if necessary.

```sh
# Get the status of a rollout in progress
kubectl rollout status deployment/myapp-deployment
# Show all previous rollouts
kubectl rollout history deployment/myapp-deployment
```

There are two types of rollout strategies:
1. Recreate: destroy all pods and then recreate them all
1. Rolling update: Take down each pod only after it has been successfully replaced

The former causes downtime, so the latter is the default.

Typically a new rollout is triggered by applying a new deployment.
```sh
kubectl apply -f deployment-definition.yml
```

However, you can also set an image in an existing deployment. This has the downside of running an image that does not match what is defined in the deployment.
```sh
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
```

Rollout history is maintained by not deleting old replicasets. When you roll back it will revert to the previous replicaset.

Command recap:
```sh
# Apply a deployment manifest
kubectl create -f deployment-definition.yml
# Show deployments
kubectl get deployments
# Apply a deployment manifest
kubectl apply -f deployment-definition.yml
# Set an image on a running deployment
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
# Get the status of a rollout in progress
kubectl rollout status deployment/myapp-deployment
# Show all previous rollouts
kubectl rollout history deployment/myapp-deployment
# Roll back to the previous deployment
kubectl rollout undo deployment/myapp-deployment
```

### Jobs

Whereas the purpose of a replicaset is to keep a given number of pods running at all times, a job is meant to run a certain task to completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  template:
    spec:
      containers:
        - name: math-add
          image: ubuntu
          command: ['expr', '3', '+', '2']
      restartPolicy: Never
```

Additional options:
```yaml
spec:
  # Continue creating new pods until three exit succesfully
  completions: 3
  # Run up to two of these concurrently
  parallelism: 2
```

### CronJobs

It does exactly what you would expect it to.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: reporting-cron-job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: math-add
              image: ubuntu
              command: ['expr', '3', '+', '2']
          restartPolicy: Never
```

## Services & Networking

### Services

Services allow communication to and from pods. Without this you would only be able to commuticate with the pod by SSHing into the node, which isn't very useful.

See notes in Kubernetes for Beginners [here](./udemy--kubernetes-for-beginners.md#services).

### Ingress Networking

Ingress exposes HTTP routes from outside the cluster to services within the cluster. Ingress may provide load balancing, SSL termination and name-based virtual hosting.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
    selector:
      name: nginx-ingress

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount

---

apiVersion: extensions/v1beta1
kind: Ingres
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80 
```

Traffic can be routed according to different rules, such as:
- `www.my-online-store.com`
    - `www.my-online-store.com/`
    - `www.my-online-store.com/watch`
    - `www.my-online-store.com/listen`
- `www.wear.my-online-store.com`
    - `www.wear.my-online-store.com/`
    - `www.wear.my-online-store.com/returns`
    - `www.wear.my-online-store.com/support`
- `www.wear.my-online-store.com`
    - `www.wear.my-online-store.com/`
    - `www.wear.my-online-store.com/movies`
    - `www.wear.my-online-store.com/tv`
- `Everything else`
    - `www.listen.my-online-store.com/`
    - `www.eat.my-online-store.com/`
    - `www.drink.my-online-store.com/tv`

Ingress with rule to route different paths to different services:
```yaml
apiVersion: extensions/v1beta1
kind: Ingres
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - http:
      paths:
        - path: /wear
          backend:
            serviceName: wear-service
            servicePort: 80 
        - path: /watch
          backend:
            serviceName: watch-service
            servicePort: 80 
```

Ingress with rule to route different sub-domains to different services:
```yaml
apiVersion: extensions/v1beta1
kind: Ingres
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - host: wear.my-online-store.com
      http:
        paths:
          - backend:
              serviceName: wear-service
              servicePort: 80 
    - host: watch.my-online-store.com
      http:
        paths:
          - backend:
              serviceName: watch-service
              servicePort: 80 
```


.

.

.

.

.

.

.

.

.
