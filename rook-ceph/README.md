# rook-ceph

* https://www.youtube.com/watch?v=wIRMxl_oEMM&t=163s

 * watch this video and 

* firstly goto rook/cluster/examples/kubernetes/ceph/
<pre>
kubectl create -f crds.yaml && common.yaml && operator.yaml && toolbox.yaml 
if u got mountfailed error, run 'kubectl -n rook-ceph create secret generic rook-ceph-crash-collector-keyring'
then exec toolbox and check ceph status
then cd csi/rbd --> kubectl create storageclass.yaml & pvc.yaml && wordpress.yaml
</pre>
<pre>
I created kube cluster on aws;
so yon need to add another storage devices /dev/xvdb on every single nodes
pvcreate /dev/xvdb ( logical mount )

and make filesystem with mkfs. cmd / and mount ( for bare metal )
</pre>
