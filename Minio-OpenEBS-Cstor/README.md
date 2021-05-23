# Kubernetes Cluster Backup and Restore with Minio


* In this lab, I will use 4 nodes, one worker,master and three worker nodes. Actually, we should have 5 node, one master and 4 workers. Due to VM limits in my laptop, I will use master node as worker node. And every node has at least 2 additional attached disks for openebs cstor storage. They shouldn't be formatted.
Every node should has at least 2CPU and 4GB RAM to deploy Minio Storage on them.

* Firstly, we have to create a kubernetes cluster. Run it on all 4 nodes to install require packages.
```bash
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
apt update && apt install -y kubeadm=1.18.0-00 kubelet=1.18.0-00 kubectl=1.18.0-00
}
```
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
kmaster    Ready    control-plane,master   55m   v1.18.0   10.0.0.4      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker1   Ready    <none>                 51m   v1.18.0   10.0.0.5      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker2   Ready    <none>                 51m   v1.18.0   10.0.0.6      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker3   Ready    <none>                 51m   v1.18.0   10.0.0.7      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
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
```bash
kubectl get sc
```
Install the MinIO operator deployment. First run krew.
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"${OS}_${ARCH}" &&
  "$KREW" install krew
)
```
```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```
Let’s get started by initializing the MinIO operator deployment. This is a one time process.

```bash
kubectl krew install minio
kubectl minio init
kubectl get pods -n minio-operator
```
Install the MinIO cluster

A tenant is a MinIO cluster created and managed by the operator. Before creating a tenant, please ensure you have requisite nodes and drives in place. In this guide, we are using 4 Nodes with one 100Gi block device attached per each node. Using the MinIO operator, the following command will generate a YAML file as per the given requirement and the file can be modified as per user specific requirements.
```bash
kubectl minio tenant create tenant1 --servers 4 --volumes 4 --capacity 400Gi -o > tenant.yaml
```
```yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  creationTimestamp: null
  name: tenant1
  namespace: minio-operator
scheduler:
  name: ""
spec:
  certConfig: {}
  console:
    consoleSecret:
      name: tenant1-console-secret
    image: minio/console:v0.6.8
    replicas: 2
    resources: {}
  credsSecret:
    name: tenant1-creds-secret
  image: minio/minio:RELEASE.2021-04-06T23-11-00Z
  imagePullSecret: {}
  mountPath: /export
  pools:
  - affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: v1.min.io/tenant
              operator: In
              values:
              - tenant1
          topologyKey: kubernetes.io/hostname
    resources: {}
    servers: 4
    volumeClaimTemplate:
      apiVersion: v1
      kind: persistentvolumeclaims
      metadata:
        creationTimestamp: null
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: cstor-csi
      status: {}
    volumesPerServer: 1
  requestAutoCert: false
status:
  availableReplicas: 0
  certificates: {}
  currentState: ""
  pools: null
  revision: 0
  syncVersion: ""

---
apiVersion: v1
data:
  accesskey: NTRiNmJiZGItZTk0Ny00MjM2LTk5MTktMmEzYWQwN2FiYWVl
  secretkey: MjNhMGQ4MWYtZTdhNi00N2Q5LTk1ZmYtNGExYmUxMzZmOWVl
kind: Secret
metadata:
  creationTimestamp: null
  name: tenant1-creds-secret
  namespace: minio-operator

---
apiVersion: v1
data:
  CONSOLE_ACCESS_KEY: YWRtaW4=
  CONSOLE_PBKDF_PASSPHRASE: ODcwMzk2YTktNTY0MS00MGQyLTg5OWQtN2JlZTFmM2QxODM0
  CONSOLE_PBKDF_SALT: YmY2ODc0ZDMtNTdmOS00YjhjLTlkZGUtNzNhN2ZjNDI2MzU4
  CONSOLE_SECRET_KEY: ZWE5YTllY2MtYzZhMy00MDdkLTkwZTktZTc1YzA5NDk5ZTcx
kind: Secret
metadata:
  creationTimestamp: null
  name: tenant1-console-secret
  namespace: minio-operator
```
Apply tenant1.yaml.
```bash
$ kubectl apply -f tenant.yaml

tenant.minio.min.io/tenant1 created
secret/tenant1-creds-secret created
secret/tenant1-console-secret created
```
Verify the MinIO cluster creation is successfully running 

```bash
root@kmaster:~# kubectl get pods -n minio-operator
NAME                              READY   STATUS    RESTARTS   AGE
console-67b748cdc8-lk5hx          1/1     Running   0          16m
minio-operator-85dc48fc66-dhsn6   1/1     Running   0          16m
tenant1-ss-0-0                    1/1     Running   0          5m40s
tenant1-ss-0-1                    1/1     Running   0          5m40s
tenant1-ss-0-2                    1/1     Running   0          5m40s
tenant1-ss-0-3                    1/1     Running   0          5m40s
```
Verify MinIO service status.

```bash
root@kmaster:~# kubectl get svc -n minio-operator
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
console           ClusterIP   10.100.69.146    <none>        9090/TCP,9443/TCP   18m
minio             ClusterIP   10.111.119.194   <none>        80/TCP              8m47s
operator          ClusterIP   10.106.155.72    <none>        4222/TCP,4233/TCP   18m
tenant1-console   ClusterIP   10.110.238.172   <none>        9090/TCP            25s
tenant1-hl        ClusterIP   None             <none>        9000/TCP            8m47s
```
Expose minio and console service as loadbalancer. Fisrtly deploy metalb.
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Configre LB ip range.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.0.0.20-10.0.0.25
```
Apply metallb.
```bash
kubectl apply -f metallb.yml
```
Check minio service status.
```bash
root@kmaster:~# kubectl get svc -n minio-operator
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
console           NodePort       10.100.69.146    <none>        9090:30317/TCP,9443:30003/TCP   4h49m
minio             LoadBalancer   10.111.119.194   10.0.0.20     80:30245/TCP                    4h40m
operator          ClusterIP      10.106.155.72    <none>        4222/TCP,4233/TCP               4h49m
tenant1-console   ClusterIP      10.110.238.172   <none>        9090/TCP                        4h32m
tenant1-hl        ClusterIP      None             <none>        9000/TCP                        4h40m
```
Access minio via LB ip.
```bash
http://10.0.0.20
```
```
You should enter the Access key and Secret key to login into the user console. These credentials can be obtained from the secret.
```bash
kubectl get secret tenant1-creds-secret -o yaml -n minio-operator
```
```yaml
apiVersion: v1
data:
  accesskey: NTRiNmJiZGItZTk0Ny00MjM2LTk5MTktMmEzYWQwN2FiYWVl
  secretkey: MjNhMGQ4MWYtZTdhNi00N2Q5LTk1ZmYtNGExYmUxMzZmOWVl
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"accesskey":"NTRiNmJiZGItZTk0Ny00MjM2LTk5MTktMmEzYWQwN2FiYWVl","secretkey":"MjNhMGQ4MWYtZTdhNi00N2Q5LTk1ZmYtNGExYmUxMzZmOWVl"},"kind":"Secret","metadata":{"annotations":{},"creationTimestamp":null,"name":"tenant1-creds-secret","namespace":"minio-operator"}}
  creationTimestamp: "2021-05-23T07:48:40Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:accesskey: {}
        f:secretkey: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:type: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-05-23T07:48:40Z"
  name: tenant1-creds-secret
  namespace: minio-operator
  resourceVersion: "6452"
  uid: 7ea7fb59-57ff-43fc-9951-62ff137defb9
type: Opaque
```
Decoding of the above credentials can be retrieved by following way.
Access key
```bash
echo -n "NTRiNmJiZGItZTk0Ny00MjM2LTk5MTktMmEzYWQwN2FiYWVl" | base64 -d; echo
```
Secret Key
```bash
 echo "MjNhMGQ4MWYtZTdhNi00N2Q5LTk1ZmYtNGExYmUxMzZmOWVl" | base64 -d; echo
 ```
After login to minio successfully, install latest velero binary and extract it 
```bash
wget https://github.com/heptio/velero/releases/download/v1.6.0/velero-v1.6.0-linux-amd64.tar.gz
tar zxf velero-v1.6.0-linux-amd64.tar.gz
sudo mv velero-v1.6.0-linux-amd64/velero /usr/local/bin/
rm -rf velero*
```
Create credentials file (Needed for velero initialization)
```bash
cat <<EOF>> minio.credentials
[default]
aws_access_key_id=54b6bbdb-e947-4236-9919-2a3ad07abaee
aws_secret_access_key=23a0d81f-e7a6-47d9-95ff-4a1be136f9ee
EOF
```
Install Velero in the Kubernetes Cluster. Before deploying velero in kubernetes cluster. Go to minio , create a bucket which will use as a backup storage for velero. In this lab, I will create a bucket named bucketdemo. Then deploy velero in kubernetes cluster



