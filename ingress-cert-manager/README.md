# ingress-cert-manager
First deploy nginx ingress.All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file:
<pre>
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
$ kubectl get all -n cert-manager ( check if cert-manger three pods are running )
$ kubectl create -f cert-manager-ingress.yaml
</pre>
<pre>
$ git clone https://github.com/tho861998/kubernetes.git
$ cd kubernetes/ingress-cert-manager
$ kubectl apply -f cert-manager-ingress.yaml
$ kubectl apply -f haproxy.cfg
</pre>
