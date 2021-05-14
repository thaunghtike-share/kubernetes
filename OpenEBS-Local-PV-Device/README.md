# OpenEBS Local PV Device

Prerequisites
<pre>
- kubernetes cluster
- For provisioning Local PV using the block devices, the Kubernetes nodes should have block devices attached to the nodes. The block devices can optionally be formatted and mounted.
</pre>
Check Your Kubernetes Cluster
<pre>
NAME       STATUS   ROLES    AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kmaster    Ready    master   3m7s   v1.18.5   10.0.0.6      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker1   Ready    <none>   84s    v1.18.5   10.0.0.8      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker2   Ready    <none>   93s    v1.18.5   10.0.0.7      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
kworker3   Ready    <none>   91s    v1.18.5   10.0.0.9      <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   docker://20.10.6
</pre>



