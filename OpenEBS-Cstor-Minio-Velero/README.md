
# Kubernetes Cluster Backup and Restore with Minio 

* In this lab, I will use 4 nodes, one worker,master and three worker nodes. Actually, we should have 5 node, one master and 4 workers. Due to VM limits in my laptop, I will use master node as worker node. And every node has at least 2 additional attached disks for openebs cstor storage. They shouldn't be formatted.
Every node should has at least 2CPU and 4GB RAM to deploy Minio Storage on them.

* Firstly, we have to create a kubernetes cluster. Run it on all 4 nodes to install require packages.
<pre>
{
ufw disable
swapoff -a; sed -i '/swap/d' /etc/fstab
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install docker-ce containerd.io -y
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt update && apt install -y kubeadm=1.20.0-00 kubelet=1.20.0-00 kubectl=1.20.0-00
}
</pre>
Run this command to create a cluster on master node. 10.0.0.4 means master_node private ip.
<pre>
kubeadm init --apiserver-advertise-address=10.0.0.4 --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
</pre>
<pre>
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
</pre>
Deploy weave CNI plugin on master node. Don't use calico cni in this lab. You have to fix another network errors.
<pre>
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
</pre>
