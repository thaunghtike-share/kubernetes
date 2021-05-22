Container-Native Storage for Kubernetes Using Heketi and Glusterfs
* You have to deploy a kubernetes cluster at first.
<pre>
https://github.com/tho861998/kubernetes/Cluster-Setup/README.md
</pre>
* Each node that will be running gluster needs at least one raw block device. This block device will be made into an LVM PV and fully managed by Heketi. For typical installs at least three nodes will need to be provisioned. Extra small clusters can be configured with just one node. The three Kubernetes nodes intended to run the GlusterFS Pods must have the appropriate ports opened for GlusterFS communication. Run the following commands on each of the nodes.
<pre>
iptables -N HEKETI
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m multiport --dports 49152:49251 -j ACCEPT
service iptables save
</pre>
* Deploy gluster container onto a specified node by setting the label storagenode=glusterfs on that node.
<pre>
$ kubectl label node <...node...> storagenode=glusterfs
$ kubectl apply -f glusterfs-daemonset.json ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/glusterfs-daemonset.json )
$ kubectl get pods -o wide ( check glusterfs pods are running or not )
</pre>
* Next we will create a service account for Heketi:
<pre>
$ kubectl apply -f heketi-sa.json ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/heketi-sa.json )
</pre>
* We must now establish the ability for that service account to control the gluster pods. We do this by creating a cluster role binding for our newly created service account.
<pre>
$ kubectl create clusterrolebinding heketi-gluster-admin --clusterrole=edit --serviceaccount=default:heketi-service-account
</pre>
* Now we need to create a Kubernetes secret that will hold the configuration of our Heketi instance. The configuration file must be set to use the kubernetes executor in order for the Heketi server to control the gluster pods
<pre>
$ kubectl create secret generic heketi-config-secret --from-file=./heketi.json ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/heketi.json )
</pre>
* Next we need to deploy an initial Heketi Pod and a Service to access that pod. ( * Notice that you are using heketi/heketi:9 image* )
<pre>
$ kubectl apply -f heketi-deploy.json ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/heketi-deploy.json )
</pre>
* Submit the file and verify everything is running properly as demonstrated below:
<pre>
kubectl get pods -o wide
</pre>
* Lastly, set an environment variable for the Heketi CLI client so that it knows how to reach the Heketi Server. So, you will need to install heketi-cli (version 9 ) tool on kmaster node.
<pre>
export HEKETI_CLI_SERVER=http://192.168.77.129:8080 ( ip address is pod/deploy-heketi-584757cc46-c95nb )
</pre>
<pre>
wget https://github.com/heketi/heketi/releases/download/v9.0.0/heketi-client-v9.0.0.linux.amd64.tar.gz
tar xzvf heketi*
</pre>
* After that, you will attach a LVM storage device on each node. (pvcreate /dev/xvdg ) *
* Next we are going to provide Heketi with information about the GlusterFS cluster it is to manage. We provide this information via a topology file. It will create a heketi-cluster inside heketi pod.
<pre>
heketi-client/bin/heketi-cli topology load --json=topology-sample.json ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/topology-sample.json )
</pre>
<pre>
Found node ip-172-20-0-217.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
Adding device /dev/xvdg ... OK
Found node ip-172-20-0-218.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
Adding device /dev/xvdg ... OK
Found node ip-172-20-0-219.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
Adding device /dev/xvdg ... OK
</pre>
* Next we are going to use Heketi to provision a volume for it to store its database:
<pre>
$ heketi-client/bin/heketi-cli setup-openshift-heketi-storage
</pre>
* If you got error like that ( Error: WARNING: Failed to connect to lvmetad. Falling back to device scanning.
/usr/sbin/modprobe failed: 1
thin: Required device-mapper target(s) not detected in your kernel.
Run `lvcreate --help' for more information. )

* Run 'modprobe dm_thin_pool' on every worker node. Then, run above command again.

* Create a storageclass for dynamic storage provisioner
<pre>
$ kubectl apply -f storageclass.yaml (https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/storageclass.yaml )
</pre>
* NOTE: Restuser and restuserkey can be anything since authorization is turned off ( inside heketi.json )

* Create a pvc
<pre>
$ kubectl apply -f pvc.yaml ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/pvc.yaml )
</pre>
* Notice, that the PVC is bound to a dynamically created volume. We can also view the Volume (PV):
<pre>
kubectl get pv
</pre>
* Finally, run a deployment by using created pvc
<pre>
$ kubectl apply -f deployment.yaml ( https://github.com/tho861998/containerize-storage-kubernetes-heketi/blob/main/deployment.yaml )
</pre>

* Now we will exec into the container and create an index.html file
<pre>
$ kubectl exec nginx-deployment-86c59c984c-2gs9n it - sh
</pre>
<pre>
$ cd /usr/share/nginx/html
$ echo 'Hello World from GlusterFS!!!' > index.html
$ ls
index.html
$ exit
</pre>
* Now we can curl the URL of our pod:
<pre>
curl 192.168.41.131
</pre>
* Lastly, let's check our gluster pod, to see the index.html file we wrote. Choose any of the gluster pods
$ kubectl exec < one-of-gluster-pod > it - sh
<pre>
sh-4.2# mount | grep heketi
/dev/xvda1 on /var/lib/heketi type ext4 (rw,relatime,discard)
/dev/mapper/vg_16e99e2b58722f3b212941dbf74d8c43-brick_a9ef831b809d8947ed0ba3faedc0fa58 on /var/lib/heketi/mounts/vg_16e99e2b58722f3b212941dbf74d8c43/brick_a9ef831b809d8947ed0ba3faedc0fa58 type xfs (rw,noatime,nouuid,attr2,inode64,logbufs=8,logbsize=128k,sunit=256,swidth=512,noquota)
</pre>
* Make one of the worker nodes goes down.
* Then, scale deployment
<pre>
$ kubectl scale deploy nginx-deployment --replicas=5
</pre>
* One of worker nodes downed, but you can access the nginx pod because nginx pod's index.html file is replicated to all worker nodes ( /var/lib/kubelet) . Check your pvc.yaml file and you will see 'volumetype: replicate:3'

* Thank You!

