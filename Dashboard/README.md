# Kubernetes Dashboard Setup

First, we will deploy the k8s dashboard using the kubectl command in the terminal.
<pre>
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml 
</pre>
Now, we deploy dashboard service
<pre>
$ wget https://github.com/tho861998/kubernetes/blob/main/Dashboard/kubernetes-dashboard-deployment.yml
$ kubectl apply -f kubernetes-dashboard-deployment.yml
</pre>
Now we will check the Dashboard's creation and deployment status using this command.
<pre>
$  kubectl get deployments -n kubernetes-dashboard
</pre>
First, we will create a service account manifest file in which we will define the administrative user for kube-admin and the associated namespace they have access to.
<pre>
$ wget https://github.com/tho861998/kubernetes/blob/main/Dashboard/admin-sa.yml
$ kubectl apply -f admin-sa.yml
</pre>
Next, we will bind the cluster-admin role to the created user.
<pre>
$ wget https://github.com/tho861998/kubernetes/blob/main/Dashboard/admin-rbac.yml
$ kubectl apply -f admin-rbac.yml
</pre>
In this step, we store the specific name of the service account.
<pre>
$ SA_NAME="kube-admin"
</pre>
Now we will generate a token for the account. This is necessary for security and further employment of the user in other systems, namespaces, or clusters.
<pre>
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep ${SA_NAME} | awk '{print $1}')
</pre>
Before accessing the dashboard, it should be noted that it deploys a minimal RBAC configuration by default and requires a Bearer Token to log in. This is the token we created above. To locate the port and IP address, run this command.
<pre>
$ kubectl get service -n kubernetes-dashboard | grep dashboard
  kubernetes-dashboard NodePort 10.98.129.73 <none> 443:30741/TCP 27m
</pre>
In our setup, we used port 30741, as you can see in the second line of output of the previous command. You can log in using this port on any server. browses https://node_ip:30741.
We can also access the dashboard using the following kubectl command.
<pre>
$ kubectl proxy
</pre>

















