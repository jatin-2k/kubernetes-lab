# kubernetes-lab
A step by step guide to learning kubernetes

## kubernetes setup

### Set up nodes

set up two vm's, one to act as kmaster and one as kworker and find their ip addresses. 
```mermaid
graph LR
    A[Your Machine] --> B[Kmaster VM]
    A --> C[Kworker VM]
```

NOTE: it would be better to set up ssh into those vms
```sh
ip addr
```

set up hosts resolvers to be able to name those ips, add following to `/etc/hosts` in your machine and vms
```
<master_node_ip> kmaster.example.com kmaster
<worker_node_ip> kworker.example.com kworker
```

test connection 
```sh
ping kmaster
ping kworker
```

---

### Configure nodes

make sure selinux is not enforced in your vm to avoid permission issue later on, if it is enforced disable it
```
sudo apt-get install selinux-utils
setenforce 0
getenforce
```
turn off firewall for smooth lab (find security lab in refrences for proper firewall setup in production)
```sh
su -
ufw disable
```
turn off swap to retain node isolation properties as kubernetes support for swap is still not robust
```
su -
swapoff -a; sed -i '/swap/d' /etc/fstab
```
set up sysctl networking configuration for kubernetes
```sh
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

---

### Docker Installation process

NOTE: this might have changed follow official [_documentation_](https://docs.docker.com/engine/install/ubuntu/) to install docker engine

setup docker apt repository
```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

install docker
```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

verify docker service is running by
```sh
sudo service docker start
sudo systemctl status docker
```

(optional) to avoid using sudo in every docker command add your user to docker group
```sh
sudo usermod -aG docker $USER
getent group
sudo reboot
```

---

### Kubernetes Installation process

NOTE: this might have changed follow official [_documentation_](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) to install kubeadm

setup kubernetes apt repository
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
install kubernetes
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
restart containerd to not run into container runtime issues
```sh
rm /etc/containerd/config.toml
systemctl restart containerd
kubeadm init
```

---

### Initialise Kubernetes Cluster (on kmaster)

initialise kubeadm
```sh
kubeadm init
```

setup calico networking plugin for kubernetes
```sh
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

setup kube config for user (as non root user)
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

view cluster config
```sh
kubectl config view
kubectl cluster-info
```


## References
- [Kubernetes Security Lab](https://devopstales.github.io/kubernetes/k8s-security/#use-firewalld)
- [Video Guide By Just Me And Opensource](https://www.youtube.com/watch?v=Araf8JYQn3w&list=PL34sAs7_26wNBRWM6BDhnonoA5FMERax0)
- [Control Plane Not Runnig Error](https://k21academy.com/docker-kubernetes/container-runtime-is-not-running/#:~:text=This%20is%20a%20common%20issue,toml%20file.)