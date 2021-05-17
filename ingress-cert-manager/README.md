# ingress-cert-manager
<pre>
$ wget https://github.com/tho861998/kubernetes/blob/main/ingress-cert-manager/cert-manager-ingress.yaml
$ wget https://github.com/tho861998/kubernetes/blob/main/ingress-cert-manager/haproxy.cfg
</pre>
Go to official documentation for latest updates:
All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file:
<pre>
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
$ kubectl get all -n cert-manager ( check if cert-manger three pods are running )
$ kubectl create -f cert-manager-ingress.yaml
</pre>
don't forget to add LB (haproxy) ip in /etc/hosts
you can setup these lab on aws and add LB pulic ip in /etc/hosts with ingress url to access on public;
