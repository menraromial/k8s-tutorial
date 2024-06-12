# Kubernetes Tutorial: Setting Up a K3s Cluster with a Hello-World Deployment

This tutorial will guide you through setting up a K3s cluster with one master node and two worker nodes, deploying a simple "Hello-World" application, and utilizing various `kubectl` commands to manage and inspect Kubernetes resources.

## Prerequisites

- Three Linux machines (or VMs) with access to each other.
- Basic understanding of Linux commands and Kubernetes concepts.

## Step 1: Installing K3s

### Install K3s on the Master Node

1. **SSH into the Master Node:**

    ```bash
    ssh user@master-node-ip
    ```

2. **Install K3s:**

    ```bash
    curl -sfL https://get.k3s.io |  INSTALL_K3S_VERSION=v1.27.12+k3s1 sh -s - --disable traefik --write-kubeconfig-mode 644 --node-name <NODE_NAME>
    ```

The command installs K3s version `v1.27.12+k3s1` on a node named `<NODE_NAME>`, disables the default Traefik ingress controller, and sets the permissions for the kubeconfig file to `644`.

3. **Verify Installation:**

    ```bash
    kubectl get nodes
    ```

    You should see the master node listed.

### Install K3s on the Worker Nodes

1. **SSH into each Worker Node:**

    ```bash
    ssh user@worker-node-ip
    ```

2. **Get the token from the master node**

    ```bash
    cat /var/lib/rancher/k3s/server/node-token
    ```

3. **Install K3s on Worker Nodes:**

    On each worker node, run:

    ```bash
    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.12+k3s1  K3S_NODE_NAME=YOUR_NODE_NAME K3S_URL=https://master-node-ip:6443 K3S_TOKEN=YOUR_NODE_TOKEN sh -
    ```

    You can find the `YOUR_NODE_TOKEN` value on the master node at `/var/lib/rancher/k3s/server/node-token`.

3. **Verify Worker Nodes Join the Cluster:**

    On the master node, run:

    ```bash
    kubectl get nodes
    ```

    You should see all nodes (master and workers) listed.

## Step 2: Verifying Node Status

To check the status of the nodes:

```bash
kubectl get nodes
```

You should see the master and worker nodes with the status `Ready`.

## Step 3: Creating a Pod

Create a simple pod to ensure everything is working correctly.

1. **Create a YAML file named `hello-world-pod.yaml`:**

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: hello-world
      namespace: default
    spec:
      containers:
        - name: hello-world
          image: bashofmann/rancher-demo:1.0.0
          ports:
            - containerPort: 8080
    ```

2. **Apply the YAML file to create the pod:**

    ```bash
    kubectl apply -f hello-world-pod.yaml
    ```

3. **Verify the Pod is Running:**

    ```bash
    kubectl get pods
    ```

4. **Describe the Pod:**

    ```bash
    kubectl describe pod hello-world
    ```

## Step 4: Deleting the Pod

To delete the pod:

```bash
kubectl delete pod hello-world
```

## Step 5: Deploying the Application

### Création d'un namespace

A namespace in Kubernetes is used to organize and manage resources within a cluster, providing isolation and a way to divide cluster resources among multiple users or teams.

The commands used in this process are:

1. **View Existing Namespaces**:
   ```bash
   kubectl get namespaces
   ```
   or 
   ```bash
   kubectl get ns
   ```

2. **Create a New Namespace**:
   ```bash
   kubectl create namespace demo
   ```

3. **Verify the Creation of the Namespace**:
   ```bash
   kubectl get namespaces
   ```

This process ensures you have successfully created the `demo` namespace and confirmed its presence in your Kubernetes cluster.

### Deployment YAML File

Create a file named `hello-world-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: bashofmann/rancher-demo:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: web
              protocol: TCP
          env:
            - name: COW_COLOR
              value: purple
          readinessProbe:
            httpGet:
              path: /
              port: web
          livenessProbe:
            httpGet:
              path: /
              port: web
```

### Explanation of the YAML File

- **apiVersion**: Specifies the version of the Kubernetes API to use.
- **kind**: Specifies the type of resource being created (Deployment).
- **metadata**: Contains data that helps uniquely identify the object, including a `name` and `namespace`.
- **spec**: Defines the desired state of the Deployment.
  - **replicas**: Number of pod replicas to run.
  - **selector**: Defines how the Deployment finds which Pods to manage.
  - **template**: Defines the Pods that will be created.
    - **metadata**: Labels for the Pods.
    - **spec**: Specification of the containers within the Pods.
      - **containers**: List of containers in the Pod.
        - **name**: Name of the container.
        - **image**: Docker image to use for the container.
        - **imagePullPolicy**: Policy to use for image pulls.
        - **ports**: Ports to expose from the container.
        - **env**: Environment variables for the container.
        - **readinessProbe**: Defines a probe to check if the container is ready.
        - **livenessProbe**: Defines a probe to check if the container is alive.

### Apply the Deployment

1. **Apply the YAML file to create the Deployment:**

    ```bash
    sudo k3s kubectl apply -f hello-world-deployment.yaml
    ```

2. **Verify the Deployment and Pods:**

    ```bash
    sudo k3s kubectl get deployments -n demo
    sudo k3s kubectl get pods -n demo
    ```

3. **Check the ReplicaSets:**

    ```bash
    sudo k3s kubectl get replicaset -n demo
    ```

## Step 6: Exposing the Application

Create a service to expose the Deployment.

1. **Create a ClusterIP Service:**

    ```bash
    sudo k3s kubectl expose deployment hello-world --type=ClusterIP --name=hello-world-service --port=8080 --target-port=8080 -n demo
    ```

2. **Verify the Service:**

    ```bash
    sudo k3s kubectl get svc -n demo
    ```

## Step 7: Accessing the Application

To access the application from your local machine, use `kubectl port-forward`:

```bash
sudo k3s kubectl port-forward svc/hello-world-service 8080:8080 -n demo
```

Open your browser and go to `http://localhost:8080` to see the application in action.

## Step 8: Inspecting Resources

### Checking Nodes

To get information about the nodes in the cluster:

```bash
sudo k3s kubectl get nodes
```

### Describing a Pod

To get detailed information about a specific pod:

```bash
sudo k3s kubectl describe pod <pod-name> -n demo
```

### Describing a Deployment

To get detailed information about the deployment:

```bash
sudo k3s kubectl describe deployment hello-world -n demo
```

### Viewing Pod Logs

To view the logs of a pod:

```bash
sudo k3s kubectl logs -f <pod-name> -n demo
```

## Step 9: Cleaning Up

To delete all the resources created:

```bash
sudo k3s kubectl delete namespace demo
```

## Conclusion

You've successfully set up a K3s cluster, deployed a simple application, and used various `kubectl` commands to manage and inspect Kubernetes resources. This foundation will help you explore Kubernetes further and leverage its capabilities.