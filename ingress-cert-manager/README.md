# ingress-cert-manager
Firstly deploy nginx ingress.All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file:
<pre>
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0-alpha.1/cert-manager.crds.yaml
$ snap install helm --classic
$ kubectl create ns cert-manager
$ helm repo add jetstack https://charts.jetstack.io
$ helm install --name-template my-release --namespace cert-manager jetstack/cert-manager
$ kubectl get all -n cert-manager #check if cert-manger three pods are running 
</pre>
<pre>
$ git clone https://github.com/tho861998/kubernetes.git
$ cd kubernetes/ingress-cert-manager
$ kubectl apply -f cert.yaml #if you get error , run below commands (internal Server Error) #if you got no error, don't delete 2 resource
$ kubectl delete mutatingwebhookconfiguration.admissionregistration.k8s.io cert-manager-webhook
$ kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io cert-manager-webhook
$ kubectl apply -f dep.yml
$ kubectl apply -f haproxy.cfg
</pre>
we can use nginx as LB instead of haproxy
Add LB ip in /etc/hosts
