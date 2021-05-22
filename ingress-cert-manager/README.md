# ingress-cert-manager 
Firstly deploy nginx ingress.All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file: Acutally, it doesn't work on kube version 1.20.0 . ( use 1.17 1.16 )
Option1: Let's Encrypt
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
$ kubectl apply -f dep.yml
$ kubectl apply -f ingress.yml
</pre>
we can use nginx as LB instead of haproxy
Add LB ip in /etc/hosts

If ingress doesn't work actually, degrade cluster version ( ingress apiVersion: extensions/v1 or extensions/v1beta1 )

Option2: Self-Signed Cert. Firstly Deploy Nginx ingress.
<pre>
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
$ kubectl get pods -n cert-manager
$ kubectl apply -f self-signed.yml #( all are included in single file )
$ kubectl get pods -n hotel
$ kubectl get svc -n hotel
$ kubectl get ingree -n hotel
$ kubectl get certificates -n hotel
</pre>
Configure HaProxy for loadbalancing. Add loadbalancer ip in /etc/hosts.


