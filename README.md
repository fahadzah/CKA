# CKA

**steps to build kubernetes cluster through kubeadm**

Run the below steps on the Master VM

1- **SsH into the Master EC2 server or which ever platform your virtual machine resides. For example VMware** 

2- **Disable Swap using the below commands**

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

3- **#Forwarding IPv4 and letting iptables see bridged traffic**

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

**sysctl params required by setup, params persist across reboots**
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

**Apply sysctl params without reboot**

sudo sysctl --system

**Verify that the br_netfilter, overlay modules are loaded by running the following commands:**

lsmod | grep br_netfilter

lsmod | grep overlay

**# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:**

4- **Install container runtime**

curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz

sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz

curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo mkdir -p /usr/local/lib/systemd/system/

sudo mv containerd.service /usr/local/lib/systemd/system/

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl daemon-reload

sudo systemctl enable --now containerd

**Check that containerd service is up and running**

systemctl status containerd

5- **Install RunC**

curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc

6- **Install CNI Plugin**

curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz

sudo mkdir -p /opt/cni/bin

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz

7- **Install Kubeadm, Kubectl and Kubelet**
sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages

sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version

kubelet --version

kubectl version --client

8- **Confiugre Crictl to work with containerd**

sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock

9- **Initialize Control-plane**

sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=172.31.89.68 --node-name (node hostname)

10- **Prepare Kubeconfig**

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

11- **Install Calico**

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml

**Worker Nodes Steps**

Perform steps 1-8 on both the nodes

Run the command generated in step 9 on the Master node which is similar to below

sudo kubeadm join 172.31.71.210:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxx

"If you forgot to copy the command, you can execute below command on master node to generate the join command again"

kubeadm token create --print-join-command

# Validations

If all the above steps were completed, you should be able to run kubectl get nodes on the master node, and it should return all the 3 nodes in ready status.


Also, make sure all the pods are up and running by using the command as below:  kubectl get pods -A


If your Calico-node pods are not healthy, please perform the below steps:


Disabled source/destination checks for master and worker nodes too.


Configure Security group rules, Bidirectional, all hosts,TCP 179(Attach it to master and worker nodes)


Update the ds using the command: kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=interface=ens5 Where ens5 is your default interface, you can confirm by running ifconfig on all the machines


IP_AUTODETECTION_METHOD is set to first-found to let Calico automatically select the appropriate interface on each node.


Wait for some time or delete the calico-node pods and it should be up and running.


If you are still facing the issue, you can follow the below workaround


Install Calico CNI addon using manifest instead of Operator and CR, and all calico pods will be up and running kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml


This is not the latest version of calico though(v.3.25). This deploys CNI in kube-system NS.


