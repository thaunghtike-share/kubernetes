# ingress-cert-manager
Firstly deploy nginx ingress.All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file:
<pre>
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
$ kubectl get all -n cert-manager #check if cert-manger three pods are running 
</pre>
<pre>
$ git clone https://github.com/tho861998/kubernetes.git
$ cd kubernetes/ingress-cert-manager
$ kubectl apply -f cert-manager-ingress.yaml #if you get error , run below commands (internal Server Error)
$ kubectl delete mutatingwebhookconfiguration.admissionregistration.k8s.io cert-manager-webhook
$ kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io cert-manager-webhook
$ kubectl apply -f haproxy.cfg
</pre>
we can use nginx as LB instead of haproxy
Add LB ip in /etc/hosts
