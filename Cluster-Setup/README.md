Download kube.sh file on all nodes ( including master)
<pre>
$ git clone https://github.com/tho861998/kubernetes.git
$ cd kubernetes/Cluster-Setup
$ chmod +x kube.sh
$ ./kube.sh
</pre>
Run this script on kmaster node
<pre>
$ kubeadm init --apiserver-advertise-address=<master_node_ip> --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
</pre>
Deploy calico network on your master node
<pre>
$ kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
(or)
$ flannel / weave 
</pre>
Run this command to get token on master
<pre>
kubeadm token create --print-join-command
</pre>
Run the token resulted on all worker nodes you want to join.
