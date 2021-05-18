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

.

.

.

.

.

.

.

.

.
