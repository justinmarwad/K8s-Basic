# Kubernetes (K8s) Basic Demo 

Decided to play around with Kubernetes and see how it works. I'm using six Ubuntu Server 22.04 LTS VMs setup in a highly-available workflow over my six homelab desktop servers. 

## Installation

I followed this guide: https://www.linuxtechi.com/install-kubernetes-on-ubuntu-22-04/. In short, I did the following on ALL nodes. (Make sure to configure the hostname and domain name on all nodes with the hosts file or an external DNS.)

1. Disable swapping (k8s doesn't like it): `sudo swapoff -a;  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

2. Loud kernel modules: 

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

3. Set kernel parameters: 

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF 
``` 

4. Reload sysctl: `sudo sysctl --system`

5. Install containerd & requirements: 

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update; sudo apt install -y containerd.io
```

6. Configure containerd to use systemd: 

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
``` 

7. Install Kubernetes: 

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt update; sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Create Pod

1. Init command on master (if domain/hostname, then make sure it's all lowercase): ` sudo kubeadm init --control-plane-endpoint=<domain or ip>`

2. Create config: 

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Copy the output of the init command and run it on all nodes, will look like this: ` sudo kubeadm join k8smaster.example.net:6443 --token vt4ua6.wcma2y8pl4menxh2 --discovery-token-ca-cert-hash sha256:0494aa7fc6ced8f8e7b20137ec0c5d2699dc5f8e616656932ff9173c94962a36` 

4. Install CNI plugin, but follow instructions here when modifying file: https://stackoverflow.com/questions/61201252/kubernetes-calico-nodes-0-1-ready. In short, you'll need to change the IP address range in the file to your range and also add an option in for the interface (a random one is chosen if you don't).

```bash 
curl https://docs.projectcalico.org/manifests/calico.yaml -O 
vim calico.yaml
kubectl apply -f calico.yaml
```



5. Check nodes: `kubectl get nodes`. Note: you'll want to run this without root since your config file is in your user's home directory.


## Run Services

1. Run two Nginx containers: `kubectl create deployment nginx-app --image=nginx --replicas=2`

2. Check status: `kubectl get deployment nginx-app`

3. Expose the deployment: `kubectl expose deployment nginx-app --type=NodePort --port=80`

4. Check service status: 

```bash
kubectl get svc nginx-app
kubectl describe svc nginx-app
```

## Issues I Went Through So You Wouldn't Have To 

### In Case You Need To Delete Everything

In case you ever need to delete everything and starting again, run the following command on both the master and all nodes: `sudo kubeadm reset`

### Localhost:8080 API Server Not Working

If you're trying to access the API server on localhost:8080 and it's not working, you'll need to run the kubectl as a normal user instead or using sudo or root. This is because your "~/.kube/config" file is in your user's home directory and not root's. 

### Nodes "NotReady" 

This is probably an issue with the CNI plugin. Normally you run the following: `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml` but you'll need to change the IP address range in the file to your range and also add an option in for the interface (a random one is chosen if you don't). See this for more details: https://stackoverflow.com/questions/61201252/kubernetes-calico-nodes-0-1-ready.