# Infrastructure and Kubernetes Setup

## Infrastructure Description

This infrastructure setup includes:

- **Two Worker Nodes**: These nodes handle the execution of workloads and applications.
- **One Master Node (Control Plane)**: Manages the worker nodes and the overall Kubernetes cluster.

## Operating System Layer

Each machine in this setup is running **Ubuntu 24.04** as the base operating system.

## Kubernetes Layer

### Kubernetes Control Plane Components

- **kube-apiserver**: Exposes the API to interact with the cluster
- **etcd**: A highly available key-value store that stores the current and disired state of the cluster.
- **kube-controller-manager**: Ensures that the cluster's desired state is synchronized with its current state, managing controllers that handle routine tasks.
- **kube-scheduler**: Responsible for scheduling pods onto the appropriate worker nodes based on resource availability and other constraints.

## Kubernetes Worker Node Components

- **kubelet**: Responsible for managing pods on the node and sending statuses to the control plane.
- **kube-proxy**: Handles routing traffic to pods.
- **container runtime**: Manages the execution of containers.
- **Docker**: Used as the container runtime in this setup.

## Installation Steps for Kubernetes Components

1. **Update package lists:**

    ```bash
    sudo apt-get update
    ```

2. **Turn off swap memory:**

    ```bash
    sudo swapoff -a
    ```

3. **Install necessary packages:**

    ```bash
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    ```

4. **Download the Google public signing key:**

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

5. **Add the Kubernetes apt repository:**

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

6. **Update package lists again:**

    ```bash
    sudo apt-get update
    ```

7. **Install Kubernetes components:**

    ```bash
    sudo apt-get install -y kubelet kubeadm kubectl
    ```

8. **Hold the versions of Kubernetes components:**

    ```bash
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## Steps to Initialize the Kubernetes Cluster

1. **Pull control plane component images:**

    ```bash
    kubeadm config images pull
    ```

2. **Initialize the Kubernetes cluster:**

    ```bash
    kubeadm init --pod-network-cidr=192.168.0.0/16
    ```

3. **Deploy Calico CNI for cluster networking:**

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
    ```

4. **Join worker nodes to the cluster:**

    ```bash
    kubeadm join <MASTER NODE IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <CERT HASH>
    ```

    Replace `<MASTER NODE IP>`, `<TOKEN>`, and `<CERT HASH>` with the appropriate values from your `kubeadm init` output.


## Workload Deployment on Kubernetes Cluster

I have deployed two applications on this self-managed Kubernetes cluster: `nginx` and `postgresql`. Helm charts were used for these deployments, simplifying the process with predefined settings and easy management.

### Deployment of NGINX

For deploying NGINX, I created a custom Helm chart and used my own `api_server_values.yaml` file to configure various settings, including:

- **Image Name and Tag**: Specifies the container image to avoid any exception if any new version with tag is realeased.
- **Service Type and Port**: Defines how the service is exposed.
- **Resource Limits and Requests**: Sets the resource constraints for the containers.
- **NGINX ConfigMap**: Adds default pages and configurations.

I also added a feature to conditionally create the ConfigMap. By setting `nginxconfigmap.enabled:true`, the ConfigMap will be created, making the Helm chart versatile and easy to adapt for other applications by simply modifying the `values.yaml` file.

**Deployment Command:**
    ```bash
      helm install app -f values/app_server_values.yaml ./server-app -n app --create-namespace
    ```

### Deployment of PostgreSQL

For PostgreSQL, I utilized the Bitnami `postgresql-ha` Helm chart, which provides high availability. Bitnami offers two charts for PostgreSQL:

- **`postgres` Chart**: Implements a master-slave model with one master node for write operations and one slave node for read operations. Queries must be manually directed to the appropriate node.
- **`postgresql-ha` Chart**: Deploys a master node, a slave node, and a `pgpool` node. `pgpool` automatically directs write queries to the master and read queries to the slave, simplifying database management and load distribution.

So, I preferred to use `postgresql-ha` chart to directed query to specific node with more effieciently to make the database cluster highly available and fast.

**Deployment Command:**
    ```bash
      helm install postgresql -f postgres.yaml bitnami/postgresql-ha -n database  --create-namaspace
    ```


## Note on Database Credentials

When deploying the database chart, you need to set up credentials for database users. I used Kubernetes Secrets to store these credentials, but since I also push the Helm chart to GitHub, the secrets are included in the repository. This isn't ideal for security.

For production environments, it's better to:

- **Create Secrets Manually**: Set up the secrets directly in your Kubernetes cluster without including them in version control.
- **Use a Third-Party Service**: Use tools like AWS Secrets Manager or HashiCorp Vault to keep your credentials safe.
