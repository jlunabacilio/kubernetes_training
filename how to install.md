# How to install Kubernetes on Ubuntu

Elevate your permission to `su`

```
sudo apt-get update 
sudo apt-get upgrade -y
```

## Setup for master and worker nodes

## a) Install Docker (latest version)

### **Setup the repository**

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:<br>
```
    sudo apt-get install -y \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg \
      lsb-release
```
2. Add Docker’s official GPG key:<br>
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. Use the following command to set up the stable repository. To add the nightly or test repository, add the word nightly or test (or both) after the word stable in the commands below
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### **Install Docker Engine**

1. Update the apt package index, and install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:<br>
```
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

### **Manage Docker as a non-root user**

1. Add your user to the docker group.
```
sudo usermod -aG docker cloud_user
```
2. Restart Docker.
```
sudo systemctl restart docker.service
```
3. Log out and log back in so that your group membership is re-evaluated.

```
docker version
```
Output expected:
```
Client: Docker Engine - Community
 Version:           20.10.7
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        f0df350
 Built:             Wed Jun  2 11:56:38 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.7
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       b0f5bc3
  Built:            Wed Jun  2 11:54:50 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.6
  GitCommit:        d71fcd7d8303cbf684402823e425e9dd2e99285d
 runc:
  Version:          1.0.0-rc95
  GitCommit:        b9ee9c6314599f1b4a7f497e1f1f856fe433d3b7
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

## b) Set up Kubernetes (latest version)

### **Installing kubeadm**

1. Make sure that the `br_netfilter` module is loaded. This can be done by running `lsmod | grep br_netfilter`. To load it explicitly call `sudo modprobe br_netfilter`.<br>
Use
```
lsmod | grep br_netfilter
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; or try

```
sudo modprobe br_netfilter
```

2. As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure `net.bridge.bridge-nf-call-iptables` is set to 1 in your sysctl config, e.g.<br>
```

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```
sudo sysctl --system
```

### **Container runtimes**

1. On each of your nodes, install the Docker for your Linux distribution as per Install **Docker Engine**. You can find the latest validated version of Docker in this dependencies file.

2. Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.

```
sudo mkdir /etc/docker
```

```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

3. Restart Docker and enable on boot:

```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### **Installing kubeadm, kubelet and kubectl**

1. Update the `apt` package index and install packages needed to use the Kubernetes apt repository:

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. Download the Google Cloud public signing key:

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

3. Add the Kubernetes `apt` repository:

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Update `apt` package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Ensure that kubelet is running if not try
```
systemctl enable kubelet
```

## Control plane/Master steps
(still using sudo privileges)
1. Initialize the cluster using the IP range for Flannel - set network model. (The step takes 5 minutes aprox)
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

2. Copy the `kubeadmn join` command that is in the output. Example:
```
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.125.128:6443 --token dgckc9.z1glo2kuv0pv9rhd \
        --discovery-token-ca-cert-hash sha256:de5327bf6dbec9fe6a7ec8645dc42c1f3bc19e5869df96fa0e1f063bfcf31c47
```

3. Exit `sudo`, copy the `admin.conf` to your home directory, and take ownership.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

4. Deploy Flannel

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

5. Check the cluster state.

```
kubectl get pods --all-namespaces
```

## To add more nodes

1. Run the `join` command (from previous step **2**), this requires running the command prefaced with `sudo` on the nodes (if we hadn't run `sudo su` to begin with). Then we'll check the nodes from the master.

```
kubectl get nodes
```

---

## Create and Scale a Deployment using kubectl

1. Create a simple deployment:
```
kubectl create deployment nginx --image=nginx

kubectl create deployment apache --image=httpd

kubectl create deployment ghost --image=ghost
```

2. inspect the pod
```
kubectl get pods
```

3. Scale the deployment
```
kubectl scale deployment nginx --replicas=4
```

4. Inspect the pods in default namespace. We should have four now.
```
kubectl get pods
```

5. Deleting the pod
```
kubectl delete deployment nginx apache ghost
```
---
## Exposing a Nginx service

```
kubectl create ns nginx
```
```
kubectl get namespaces
```
```
kubectl create -f nginx-deployment.yaml
```
```
kubectl get pods -n nginx
```
```
kubectl create -f nginx-service.yaml
```
```
kubectl get svc -n nginx
```
```
kubectl scale --current-replicas=1 --replicas=2 -f nginx.yaml
```
```
kubectl get pods -n nginx
```
```
kubectl scale --current-replicas=2 --replicas=1 -f nginx.yaml
```
```
kubectl delete -f nginx-service.yaml
```
```
kubectl get svc -n nginx
```
```
kubectl describe -n nginx services/nginx-service
```
```
kubectl get pods -n nginx
```
```
kubectl delete namespace nginx
```
```
kubectl get svc -n nginx
```
```
kubectl get pods -n nginx
```
```
kubectl get namespaces
```