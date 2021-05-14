# heketi-glusterfs

h1. Dynamic Storage Provision ( Kubernetes )

* What is Dynamic Storage Provision?
<pre>
Dynamic volume provisioning allows storage volumes to be created on-demand. Without dynamic provisioning, cluster administrators have to manually make calls to their cloud or storage provider to create new storage volumes, and then create PersistentVolume objects to represent them in Kubernetes.

In this lab, I will use 4 VMs on aws cloud. One for kubernetes master node and three for kubernetes worker nodes. I will use two worker nodes for glusterfs servers and one worker node for heketi. ( Dynamci Storage Provision like rook and heketi uses Logical free disk storage device (i.e pvcreate /dev/xvdb on each glusterfs nodes)
</pre>
<pre>
Kmaster
Kworker1 - glusterfs1
Kworker2 - glusterfs2
Kworker3 - heketi 
</pre>
* What are glusterfs and heketi?
<pre>
*heketi* is a gluster volume manager that provides a restful interface to create/manage gluster volumes. heketi makes it easy for cloud services such as kubernetes, openshift, and openstack manila to interact with gluster clusters and provision volumes as well as manage brick layout
</pre>
* this is how I deploy my kube cluster...
<pre>
https://github.com/tho861998/heketi-glusterfs/blob/main/cluster.txt
</pre>
* after cluster creating, i will setup two glusterfs servers and one heketi machine.
<pre>
https://github.com/tho861998/heketi-glusterfs/blob/main/heketi-setup.txt
</pre>
* ( In this step, make sure that you need to add all worker nodes ip to /etc/hosts on heketi
<pre>
172.16.0.105 glusterfs1.example.com glusterfs1
172.16.0.197 glusterfs2.example.com glusterfs2
172.16.0.59  heketi.example.com     heketi
</pre>
* verify heketi is working well or not
<pre>
curl localhost:8080/hello; echo
</pre>
* The next step is to create cluster on heketi node
* Login to heketi server ( On heketi node )
</pre>
export HEKETI_CLI_USER=admin
export HEKETI_CLI_KEY=secretpassword
</pre>
* create cluster
<pre>
 heketi-cli cluster create
</pre>
* add node to this cluster ( i have two glusterfs servers, so, i will add two nodes to this cluster) 
<pre>
heketi-cli node add --zone 1 --cluster 58fcef197e8c88df212e9e72dd0c59ee --management-host-name kworker1-glusterfs1 --storage-host-name 172.16.0.17
heketi-cli node add --zone 1 --cluster 58fcef197e8c88df212e9e72dd0c59ee --management-host-name kworker2-glusterfs2 --storage-host-name 172.16.0.197

58fcef197e8c88df212e9e72dd0c59ee is your cluster id --> *heketi-cli cluster list*
ip address are glusterfs1 and glusterfs2
</pre>
* add LVM storage device to each node
<pre>
 heketi-cli device add --name /dev/xvdf --node 6e2182c1f95120beabe60dd7fa8bba5c
 heketi-cli device add --name /dev/xvdf --node 657af7f8cdab199f701b3b2ef3fcb667

you will get node ip after adding each node to heketi cluster 
</pre>
* view cluster info --> currently your heketi cluster has no volume because you haven't not created any vpc to this heketi cluster ).
<pre>
 heketi-cli cluster info < your-heketi-cluster-id >
</pre>

* Now, you are ready to start pvc creating and pod deployments ***
<pre>

Firstly , you have to create a storageclass for heketi. So, run 'kubectl apply -f storage-class-glusterfs.yaml' on master node

https://github.com/tho861998/heketi-glusterfs/blob/main/storage-class-glusterfs.yaml

Provisioner   --> glusterfs
resturl       --> heketi server curl
restuser      --> your heketi user
resetuserkey  --> your heketi password
replicas2     --> this is required to replicate your pod data to both glusterfs , if you don't have this part, your pod data will be stored in one 
                  worker node which pod is running 
</pre>
* Next, create PVC 

* Notice that you must install glusterfs-client tool on all worker nodes. When you deploy pod , glusterfs-client will help you to mount pvc that is using heketi server on each worker node
<pre>
https://github.com/tho861998/heketi-glusterfs/blob/main/pvc-glusterfs.yaml

accessMode - RWX
capactiy   - 2Gi
</pre>
* Final Step is to run deployment 
<pre>
https://github.com/tho861998/heketi-glusterfs/blob/main/deployment.yaml
</pre>
* After creating pods , expose service deployment as NodePort
<pre>
kubectl expose deploy nginx-deployment --type=NodePort
</pre>
* Verify service is exposed now 
<pre>
kubectl get svc -o wide
</pre>
* Testing 
<pre>
Stop one of the glusterfs servers, 

kubectl scale deploy nginx-deployment --replicas 5
apply metallb loadbalancing for external ip 
</pre>

* one of glusterfs servers is getting down but you will see pods are still running and service is also working well
* Notice that you must install glusterfs-client tool on all worker nodes. When you deploy pod , glusterfs-client will help you to mount pvc that is using heketi server on each worker node
















