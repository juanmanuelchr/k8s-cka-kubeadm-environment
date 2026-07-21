# Local Environment based on kubeadm

The idea of this environment is to have a local sandbox for preparing for Kubernetes certifications based on kubeadm (https://kubernetes.io/docs/reference/setup-tools/kubeadm/)

## Setup Instructions

### Prerequisites

- Basic knowledge of Linux and container concepts
- Terminal/Command Line familiarity
- Git installed
- [Multipass](https://multipass.run/) (for creating local and light Virtual Machines)

### Environment Setup

1. **Install Multipass:**

We can install it on Linux, Windows and MacOS.

| OS | Reference |
| ------ | ------ |
| Windows | https://multipass.run/docs/installing-on-windows |
| MacOS | https://multipass.run/docs/installing-on-macos |
| Linux | https://multipass.run/docs/installing-on-linux |

We will use Homebrew for MacOS:

   - `brew install multipass`
   - `multipass version` for verifying install and version
  ```bash
        $ multipass version

        multipass   1.16.1+mac
        multipassd  1.16.1+mac
   ```

2. **Creating Linux virtual machines for master and worker nodes**

    > You should consider:
    > - 1 master node and 1 worker node for starting (then you can add extra ones as needed)
    > - 2 GiB or more of RAM per machine--any less leaves little room for your apps.
    > - At least 2 CPUs on the machine that you use as a control-plane node.

    Launching master node:
    ```bash
    multipass launch --name master-1 --memory 2G --cpus 2 --disk 10G
    ```

    Launching worker node:
    ```bash
    multipass launch --name worker-1 --memory 2G --cpus 2 --disk 10G
    ```

    Verify nodes:
    ```bash
    multipass ls
    ```
    ```bash
    Name                    State             IPv4             Image
    master-1                Running           192.168.2.2      Ubuntu 24.04 LTS
    worker-1                Running           192.168.2.3      Ubuntu 24.04 LTS
    ```

3. **Login to the master node and install pre-requisites**

> Choose your Kubernetes version. For my case, i used version `v1.35` but you can use another version if needed.

Connecting to master node "master-1"
```
multipass shell master-1
```

Change user to root for running admin commands:
```
sudo -i
```

Run the following commands for upgrading Ubuntu to the latest version and install initial dependencies:
```
apt-get update
apt-get upgrade
apt-get install -y apt-transport-https ca-certificates curl
```

4. **Installing and configuring containerd**
```
apt-get install containerd -y
mkdir -p /etc/containerd
containerd config default /etc/containerd/config.toml
systemctl restart containerd
```

5. **Installing Kubernetes v1.35 components:**
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

6. **Disable swap:**
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

7. **Load necessary kernel modules:**
```
modprobe overlay
modprobe br_netfilter
```

8. **Set required sysctl parameters:**
```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

9. **Initialize the cluster (run only on master node):**
```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

10. **Setup kubeconfig file:**
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
```
export KUBECONFIG=$HOME/.kube/config
```

11. **Install Flannel network plugin (run only on master node):**
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

12. **Done!**
```
kubectl get all -A

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   6m29s
```

13. **Login to the worker node and install pre-requisites**

Connecting to master node "worker-1"
```
multipass shell worker-1
```

Change user to root for running admin commands:
```
sudo -i
```

Run the following commands for upgrading Ubuntu to the latest version and install initial dependencies:
```
apt-get update
apt-get upgrade
apt-get install -y apt-transport-https ca-certificates curl
```

14. **Repeat steps 4 to 8 for the worker node**

15. **Restart and enable containerd:**
```
systemctl restart containerd
systemctl enable containerd
```

16. **Enable and start kubelet:**
```
systemctl enable kubelet
systemctl start kubelet
```

17. **Generate a kubeadm join token:**
Now, back on the master node, run the following to generate the kubeadm join command you need to run from the worker to, well, join the cluster:

```
sudo -i
```

```
kubeadm token create --print-join-command
```

```
root@master-1:~# kubeadm token create --print-join-command
kubeadm join 192.168.252.2:6443 --token xmf7mu.v96hul14bbnhrzce --discovery-token-ca-cert-hash sha256:e6fee5579ce648e4efcf5efce8c86d4c755a1e9bc166f2c7dca94bffd5b377b6 
```

18. **Add the worker node to the cluster:**
And then, back on the worker, run the output that you get on the previous command (kubeadm token create):

```
sudo -i
```

```
kubeadm join 192.168.252.2:6443 --token xmf7mu.v96hul14bbnhrzce --discovery-token-ca-cert-hash sha256:e6fee5579ce648e4efcf5efce8c86d4c755a1e9bc166f2c7dca94bffd5b377b6 
```

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[preflight] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W0720 23:59:40.035043   24562 utils.go:69] The recommended value for "bindAddress" in "KubeProxyConfiguration" is: ::; the provided value is: 0.0.0.0
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 509.025125ms
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

19. **Confirm that it’s a healthy, happy cluster! On the master node, run:**
```
sudo -i
```

```
kubectl get nodes
```

```
root@master-1:~# kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   28m   v1.35.6
worker-1   Ready    <none>          45s   v1.35.6
```

20. **Label the worker node**
```
kubectl label node worker-1 node-role.kubernetes.io/worker=worker
```

21. **Create a test deployment using nginx latest image with four (4) replicas:**
```
kubectl create deploy test-deploy --image=nginx --replicas=4
```
```
deployment.apps/test-deploy created
```

```
root@master-1:~# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
test-deploy-84d9966d59-7vxv4   1/1     Running   0          9s
test-deploy-84d9966d59-b68wc   1/1     Running   0          9s
test-deploy-84d9966d59-fljzp   1/1     Running   0          9s
test-deploy-84d9966d59-whw48   1/1     Running   0          9s
```


Testing one of the pods by its local pod IP:
```
root@master-1:~# kubectl get pod test-deploy-84d9966d59-7vxv4  -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
test-deploy-84d9966d59-7vxv4   1/1     Running   0          4m38s   10.244.1.2   worker-1   <none>           <none>
root@master-1:~# curl 10.244.1.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


### Setting Up Lab Environments

#### Option 1: Using the TUI Launcher (Recommended)

The repository includes a Text-based User Interface (TUI) launcher for easy navigation and management of the lab environments:

```bash
cd setup
./lab_launcher.sh        # Launches the interactive TUI
```

This launcher provides a menu-driven interface to:
- Start/reset the Minikube environment
- Set up individual lab environments for each domain
- Check the status of resources in each namespace
- View node status

#### Option 2: Using Individual Setup Scripts

The repository contains setup scripts for each domain:

```bash
cd setup
./01_setup_storage_lab.sh        # Sets up Storage lab environment
./02_setup_workloads_lab.sh      # Sets up Workloads lab environment
./03_setup_networking_lab.sh     # Sets up Networking lab environment
./04_setup_troubleshooting_lab.sh # Sets up Troubleshooting lab environment
./05_setup_cluster_arch_lab.sh   # Sets up Cluster Architecture lab environment
```

To reset your environment at any time:

```bash
cd setup
./reset_lab_environment.sh
```

## How to Use This Repository

### Directory Structure

```
CKA_LAB/
├── 01_Storage/                # Storage domain exercises
│   ├── README.md              # Task instructions
│   ├── solutions/             # Official solution files
│   └── user_solutions/        # Your solutions (gitignored)
├── 02_Workloads/              # Workloads domain exercises
│   ├── ...
├── 03_Networking/             # Networking domain exercises
│   ├── ...
├── 04_Troubleshooting/        # Troubleshooting domain exercises
│   ├── ...
├── 05_Cluster_Architecture/   # Cluster Architecture domain exercises
│   ├── ...
└── setup/                     # Setup scripts
    ├── 01_setup_storage_lab.sh
    ├── ...
    ├── reset_lab_environment.sh
    └── verify_solutions.sh    # Script to verify your solutions
```