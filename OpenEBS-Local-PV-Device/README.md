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
Now, maya-apiserver is ready state



