
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
Run this command on master. Run resulted token starting with kubeadm on all worker nodes to join cluster.
<pre>
kubeadm token create --print-join-command
</pre>
Finally, you created  a kube cluster version 1.20.0 successfully.
```bash
root@kmaster:~# kubectl get nodes -o wide
NAME       STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kmaster    Ready    control-plane,master   55m   v1.20.0   10.0.0.4      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker1   Ready    <none>                 51m   v1.20.0   10.0.0.5      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker2   Ready    <none>                 51m   v1.20.0   10.0.0.6      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker3   Ready    <none>                 51m   v1.20.0   10.0.0.7      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
```
<pre>
kubectl get nodes -o wide
</pre>
In this lab, I will use master node as both control plane and worker node. So remove taints from worker nodes to deploy apps on master node.
<pre>
kubectl taint nodes $(kubectl get nodes --selector=node-role.kubernetes.io/master | awk 'FNR==2{print $1}') node-role.kubernetes.io/master-
</pre>
Next, we will start creating cstor persistent volume . First step is to deploy OpenEbs operators. You can read about openebs detail in official docs.
<pre>
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
</pre>
Before installing cstor operator. You must enable iscsid service on all nodes.
<pre>
systemctl enable --now iscsid
</pre>
Install latest cstor operator.
<pre>
kubectl apply -f https://openebs.github.io/charts/cstor-operator.yaml
</pre>
check all pods are running in openebs namespace.
<pre>
kubectl get pods -n openebs
</pre>
```bash
root@kmaster:~# kubectl get pods -n openebs
NAME                                              READY   STATUS    RESTARTS   AGE
cspc-operator-6d796d7b8f-qk28b                    1/1     Running   0          31m
cstor-storage-drr8-7557675774-fgbrq               3/3     Running   0          3m10s
cstor-storage-hr48-7864c69c47-hldjd               3/3     Running   0          3m9s
cstor-storage-vq4d-fd55c584c-f9mlc                3/3     Running   0          3m8s
cstor-storage-wvdw-64f6bcf459-7795b               3/3     Running   0          3m8s
cvc-operator-849dff65c-x6pzb                      1/1     Running   0          31m
maya-apiserver-55db98dc94-d8tmq                   1/1     Running   0          42m
openebs-admission-server-574cf7c984-d9tjk         1/1     Running   0          42m
openebs-cstor-admission-server-5d868c64d4-cxfc6   1/1     Running   0          31m
openebs-cstor-csi-controller-0                    6/6     Running   0          31m
openebs-cstor-csi-node-5s4ws                      2/2     Running   0          31m
openebs-cstor-csi-node-j9w4z                      2/2     Running   0          31m
openebs-cstor-csi-node-m7xt9                      2/2     Running   0          31m
openebs-cstor-csi-node-q8rg6                      2/2     Running   0          31m
openebs-localpv-provisioner-9f598b455-7pz6f       1/1     Running   0          42m
openebs-ndm-94pmf                                 1/1     Running   0          30m
openebs-ndm-dmcwm                                 1/1     Running   0          31m
openebs-ndm-fppmz                                 1/1     Running   0          31m
openebs-ndm-operator-59ffcfc949-m5w29             1/1     Running   0          31m
openebs-ndm-vslck                                 1/1     Running   0          31m
openebs-provisioner-65749f64fd-hcxr8              1/1     Running   0          42m
openebs-snapshot-operator-5cfbb6fdb5-ffxvr        2/2     Running   0          42m
```
Check that blockdevices are created:. See one blockdevice means one disk attached to node.
<pre>
kubectl get blockdevices -n openebs -o wide
</pre>
Provision a CStorPoolCluster. Use blockdevice which are not mounted or formatted. Don't use blockdevice like ext4.

```yaml
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
  name: cstor-storage
  namespace: openebs
spec:
  pools:
    - nodeSelector:
        kubernetes.io/hostname: "kmaster"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-6c0f0ac1aa99c9148ee3d187d08e32a4"
            - blockDeviceName: "blockdevice-b4458e89bd5442f64ef890e247176015"
      poolConfig:
        dataRaidGroupType: "stripe"

    - nodeSelector:
        kubernetes.io/hostname: "kworker1" 
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-67651fd6266f40a8e4ed7301c4e30a2e"
            - blockDeviceName: "blockdevice-680eb18dd7b630948a2d81f49d65eca9"
      poolConfig:
        dataRaidGroupType: "stripe"
   
    - nodeSelector:
        kubernetes.io/hostname: "kworker2"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-1e6d651fd9a01cc889a0d3e9a12df01d"
            - blockDeviceName: "blockdevice-b82a1982cbe0a26ba9092b9de26daf1c"
      poolConfig:
        dataRaidGroupType: "stripe"
    
    - nodeSelector:
        kubernetes.io/hostname: "kworker3"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-f202a757991e537a9c4e22a1823b4d7c"
            - blockDeviceName: "blockdevice-f98f72fba1346af698937aefbb70ec91"
      poolConfig:
        dataRaidGroupType: "stripe"
```
Apply the modified CSPC YAML.
<pre>
kubectl apply -f cspc.yml
</pre>
Check if the pool instances report their status as 'ONLINE'.
```bash
kubectl get cscp -n openebs
```

```bash
root@kmaster:~# kubectl get cspc -n openebs
NAME            HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
cstor-storage   4                  4                      4                  8m9s
```
```bash
kubectl get cspi -n openebs
```

```bash
root@kmaster:~# kubectl get cspi -n openebs
NAME                 HOSTNAME   FREE    CAPACITY      READONLY   PROVISIONEDREPLICAS   HEALTHYREPLICAS   STATUS   AGE
cstor-storage-drr8   kmaster    1920G   1920000074k   false      0                     0                 ONLINE   9m54s
cstor-storage-hr48   kworker1   1920G   1920000074k   false      0                     0                 ONLINE   9m53s
cstor-storage-vq4d   kworker2   1920G   1920000074k   false      0                     0                 ONLINE   9m53s
cstor-storage-wvdw   kworker3   1920G   1920000074k   false      0                     0                 ONLINE   9m52s
```
Check blockdevices status.

```bash
root@kmaster:~# kubectl get blockdevices -n openebs
NAME                                           NODENAME   SIZE            CLAIMSTATE   STATUS     AGE
blockdevice-00dbd3173aff10684a95b8c4da20a5fe   kworker2   1099501142016   Unclaimed    Active     34m
blockdevice-051458b8713c411d362836334e873ac4   kworker3   1099501142016   Unclaimed    Inactive   34m
blockdevice-0626988d63953239ac321fb2e0fdc5f9   kmaster    1099501142016   Unclaimed    Active     34m
blockdevice-1e6d651fd9a01cc889a0d3e9a12df01d   kworker2   1099511627776   Claimed      Active     73m
blockdevice-2ccd2c22fd26a64350de575e6f0f63e7   kmaster    1099501142016   Unclaimed    Active     34m
blockdevice-3f1521ca4ba60a0768997ec8af173c67   kworker3   4292870144      Unclaimed    Active     73m
blockdevice-54cd81fb0c088cdd14cc6457e82a4eb0   kworker3   1099501142016   Unclaimed    Inactive   34m
blockdevice-67651fd6266f40a8e4ed7301c4e30a2e   kworker1   1099511627776   Claimed      Active     73m
blockdevice-680eb18dd7b630948a2d81f49d65eca9   kworker1   1099511627776   Claimed      Active     73m
blockdevice-6c0f0ac1aa99c9148ee3d187d08e32a4   kmaster    1099511627776   Claimed      Active     73m
blockdevice-7a88de9fe00794289c281780f1b98cb4   kmaster    4292870144      Unclaimed    Active     73m
blockdevice-83c6820ff42807ab41817279d00862c1   kworker1   17177772032     Unclaimed    Active     73m
blockdevice-93fbbea4eb92daf878e8bfffd47f0651   kworker2   4292870144      Unclaimed    Active     73m
blockdevice-b4458e89bd5442f64ef890e247176015   kmaster    1099511627776   Claimed      Active     73m
blockdevice-b82a1982cbe0a26ba9092b9de26daf1c   kworker2   1099511627776   Claimed      Active     73m
blockdevice-c4ce891150b55fd92f210ffb3f66d3f9   kworker2   1099501142016   Unclaimed    Active     34m
blockdevice-f202a757991e537a9c4e22a1823b4d7c   kworker3   1099511627776   Claimed      Active     73m
blockdevice-f98f72fba1346af698937aefbb70ec91   kworker3   1099511627776   Claimed      Active     73m
```
You see? 8 blockdevices are claimed following cspc yaml file. Once your pool instances have come online, you can proceed with volume provisioning.Create a storageClass to dynamically provision volumes using OpenEBS CSI provisioner. A sample storageClass:
```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: cstor-csi
   provisioner: cstor.csi.openebs.io
   allowVolumeExpansion: true
   parameters:
     cas-type: cstor
     # cstorPoolCluster should have the name of the CSPC
     cstorPoolCluster: cstor-storage
     # replicaCount should be <= no. of CSPI
     replicaCount: "4"
    
 ```
Apply storage class.
<pre>
kubectl apply -f cstor-sc.yml
</pre>
You will need to specify the correct cStor CSPC from your cluster and specify the desired `replicaCount` for the volume. The `replicaCount` should be less than or equal to the max pool instances available. Openebs is a dynamic storage solution. So, you don't need to deploy pv. Openebs provide pv as container-attached storage.

```bash
root@kmaster:~# kubectl get pods -n openebs | grep provision
openebs-localpv-provisioner-9f598b455-7pz6f       1/1     Running   0          53m
openebs-provisioner-65749f64fd-hcxr8              1/1     Running   0          53m

```
Create a pvc using created storage class named cstor-csi.

```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: demo-cstor-vol
    spec:
      storageClassName: cstor-csi
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```
```bash
kubectl apply -f pvc.yml
```
verify thatthe PVC has been successfully created and bound to a PersistentVolume (PV).
```bash
root@kmaster:~# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
demo-cstor-vol   Bound    pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817   5Gi        RWO            cstor-csi      96s
```
After creating a pvc, target pods will be created in openebs namespace. Target pod will make sure to connect to cluster pool instance pods to replicate persistent data. Target pod results from creating pvc. Pool pods result from creating cspi.
```bash
root@kmaster:~# kubectl get pods -n openebs -o wide | grep cstor ### pool pods are created on every nodes to store data
cstor-storage-drr8-7557675774-fgbrq                               3/3     Running   0          24m     10.32.0.4   kmaster    <none>           <none>
cstor-storage-hr48-7864c69c47-hldjd                               3/3     Running   0          24m     10.42.0.6   kworker1   <none>           <none>
cstor-storage-vq4d-fd55c584c-f9mlc                                3/3     Running   0          24m     10.44.0.4   kworker2   <none>           <none>
cstor-storage-wvdw-64f6bcf459-7795b                               3/3     Running   0          24m     10.36.0.3   kworker3   <none>           <none>
```

```bash
root@kmaster:~# kubectl get pods -n openebs -o wide | grep pvc ### target pod will make to replicate persistend data from cstorvolumereplicas to cspi.
pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817-target-77f9b8dcc6krsmn   3/3     Running   0          6m27s   10.42.0.7   kworker1   <none>           <none>
```
Verify cstorvolume and its replicas are in `Healthy` state.

```bash
root@kmaster:~# kubectl get cstorvolume,cvr -n openebs
NAME                                                                    CAPACITY   STATUS    AGE
cstorvolume.cstor.openebs.io/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817   5Gi        Healthy   9m15s

NAME                                                                                              ALLOCATED   USED   STATUS    AGE
cstorvolumereplica.cstor.openebs.io/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817-cstor-storage-drr8   6K          6K     Healthy   9m14s
cstorvolumereplica.cstor.openebs.io/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817-cstor-storage-hr48   6K          6K     Healthy   9m14s
cstorvolumereplica.cstor.openebs.io/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817-cstor-storage-vq4d   6K          6K     Healthy   9m14s
cstorvolumereplica.cstor.openebs.io/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817-cstor-storage-wvdw   6K          6K     Healthy   9m14s
```
after deploying cstorvolume will create a isci LUN disk to node where pod will be deployed. cstorvolumereplicas make sure to replicate data from LUN disk to all cstor pool instances. to replicate data is the duty of pvc-### pods. to keep resulted data is the duty of pool pods cstor-storage-##. Create an application and use the above created PVC.

```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: default
    spec:
      containers:
      - command:
           - sh
           - -c
           - 'date >> /mnt/openebs-csi/date.txt; hostname >> /mnt/openebs-csi/hostname.txt; sync; sleep 5; sync; tail -f /dev/null;'
        image: busybox
        imagePullPolicy: Always
        name: busybox
        volumeMounts:
        - mountPath: /mnt/openebs-csi
          name: demo-vol
      volumes:
      - name: demo-vol
        persistentVolumeClaim:
          claimName: demo-cstor-vol
```
```bash
kubectl apply -f pod.yml
```
```bash
root@kmaster:~# kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
busybox   1/1     Running   0          35s   10.42.0.8   kworker1   <none>           <none>
```
pod is deployed on worker1. so check disks on worker1 with lsblk cmd. 

```bash
root@kworker1:~# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0     7:0    0 55.5M  1 loop /snap/core18/1997
loop1     7:1    0 67.6M  1 loop /snap/lxd/20326
loop2     7:2    0 32.1M  1 loop /snap/snapd/11841
sda       8:0    0   16G  0 disk 
└─sda1    8:1    0   16G  0 part /mnt
sdb       8:16   0   30G  0 disk 
├─sdb1    8:17   0 29.9G  0 part /
├─sdb14   8:30   0    4M  0 part 
└─sdb15   8:31   0  106M  0 part /boot/efi
sdc       8:32   0    1T  0 disk 
├─sdc1    8:33   0 1024G  0 part 
└─sdc9    8:41   0    8M  0 part 
sdd       8:48   0    1T  0 disk 
├─sdd1    8:49   0 1024G  0 part 
└─sdd9    8:57   0    8M  0 part 
sde       8:64   0    5G  0 disk /var/lib/kubelet/pods/434bcd1f-265c-4235-b2b1-3a1b93753a61/volumes/kubernetes.io~csi/pvc-4774a4ff-fbc6-4700-a6b4-c911
sr0      11:0    1  628K  0 rom  
```
here /dev/sde is a LUN cscsi disk created from cstorvolume. pod data are stored in worker1 . so how replicate to all worker nodes. It is the role of cstorvolumereplicas and pvc-##-## pod .

The example busybox application will write the current date into the mounted path at `/mnt/openebs-csi/date.txt` when it starts.

 ```bash
 $ kubectl exec -it busybox -- cat /mnt/openebs-csi/date.txt
 Wed Jul 12 07:00:26 UTC 2020
````
check ls on worker 1 LUN disk.

```bash
root@kworker1:~# ls /var/lib/kubelet/pods/434bcd1f-265c-4235-b2b1-3a1b93753a61/volumes/kubernetes.io~csi/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817/mount/
date.txt  hostname.txt  

root@kworker1:~# cat /var/lib/kubelet/pods/434bcd1f-265c-4235-b2b1-3a1b93753a61/volumes/kubernetes.io~csi/pvc-4774a4ff-fbc6-4700-a6b4-c911314bd817/mount/date.txt 
Sun May 23 03:52:27 UTC 2021
```

