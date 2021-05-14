# OpenEBS Local PV Device

Prerequisites
<pre>
- kubernetes cluster
- For provisioning Local PV using the block devices, the Kubernetes nodes should have block devices attached to the nodes. The block devices can optionally be formatted and mounted.
</pre>
Check Your Kubernetes Cluster
<pre>
$ kubectl get nodes -o wide
NAME       STATUS   ROLES    AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kmaster    Ready    master   3m7s   v1.18.5   10.0.0.6      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker1   Ready      none   84s    v1.18.5   10.0.0.8      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker2   Ready      none   93s    v1.18.5   10.0.0.7      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker3   Ready      none   91s    v1.18.5   10.0.0.9      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
</pre>
Install OpenEBS on your cluster using kubectl
<pre>
$ kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
</pre>
check your openebs components
<pre>
$ kubectl get pods -n openebs

NAME                                           READY   STATUS    RESTARTS   AGE
maya-apiserver-857c7b4c88-phwmm                0/1     Running   0          63s
openebs-admission-server-849f688f4c-bmpj9      1/1     Running   0          62s
openebs-localpv-provisioner-79fd67c695-sfjhw   1/1     Running   0          60s
openebs-ndm-bhf9c                              1/1     Running   0          62s
openebs-ndm-btpsh                              1/1     Running   0          62s
openebs-ndm-operator-8958f5ccc-lxqhh           1/1     Running   0          62s
openebs-ndm-pbn5z                              1/1     Running   0          62s
openebs-provisioner-d595ccf4c-8wlc5            1/1     Running   0          63s
openebs-snapshot-operator-7b996dd9c7-jks7g     2/2     Running   0          62s
</pre>
As you see, maya-api pod is not ready. so you have to fix it.
<pre>
$ kubectl get validatingwebhookconfigurations

NAME                             WEBHOOKS   AGE
openebs-validation-webhook-cfg   1          2m11s

$ kubectl edit validatingwebhookconfigurations openebs-validation-webhook-cfg
change failurePolicay: fail to Ignore
</pre>
Now, maya-apiserver is ready state. OpenEbs create alot of crds like blockdevice, blockdeviceclaims
<pre>
$ kubectl get crds -n openebs
</pre>
Next step is to check your blockdevice list on all nodes. I have two blockdevice (or) EBS on one worker node. So, I have total 6 blockdevices.
<pre>
$ kubectl get blockdevices -n openebs

NAME                                           NODENAME   SIZE          CLAIMSTATE   STATUS   AGE
blockdevice-6d3e3db0a98b20d4a25714f1847e85da   kworker1   17177772032   Unclaimed    Active   10m
blockdevice-78e696e868b087cd8fdeea5756e7b0f6   kworker2   53687091200   Unclaimed    Active   10m
blockdevice-81748aa4bb4f9d0d8a5473f91f1a8ab3   kworker1   53687091200   Unclaimed    Active   10m
blockdevice-881899459b7dd2652548a28b583563cb   kworker2   17177772032   Unclaimed    Active   10m
blockdevice-c6eb6cf33856a4cb113a43484028735e   kworker3   4292870144    Unclaimed    Active   10m
blockdevice-dec1b1548fe335d7b792630d5320ce20   kworker3   53687091200   Unclaimed    Active   10m
</pre>




