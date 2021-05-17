Run this script on master and worker nodes
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
apt update && apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00
}
</pre>
Run this script on kmaster node
<pre>
$ kubeadm init --apiserver-advertise-address=<master_node_ip> --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
</pre>
Deploy calico network on your master node
<pre>
$ kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
</pre>
Run this command to get token on master
<pre>
kubeadm token create --print-join-command
</pre>
Run the token resulted on all worker nodes you want to join.
