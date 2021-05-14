# istio-demo


* Firstly , create a kubernetes cluster .
<pre>
root@ubuntuvm01:~# kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
ubuntuvm01   Ready    master   4m9s    v1.18.5   172.42.42.101   <none>        Ubuntu 20.04.1 LTS   5.4.0-58-generic   docker://20.10.3
ubuntuvm02   Ready    <none>   2m14s   v1.18.5   10.0.2.15       <none>        Ubuntu 20.04.1 LTS   5.4.0-58-generic   docker://19.3.10
ubuntuvm03   Ready    <none>   2m10s   v1.18.5   172.42.42.103   <none>        Ubuntu 20.04.1 LTS   5.4.0-58-generic   docker://20.10.3
</pre>
* You need to install metallb loadbalancer for ingress access.
<pre>
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
</pre>
* Then, configure private ip range for external ip in metallb.yaml
<pre>
kubectl apply -f metallb.yaml
</pre>
* download istio from official website
<pre>
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.9.1
export PATH=$PWD/bin:$PATH
</pre>
* Install istio 
<pre>
istioctl install --set profile=demo -y
</pre>
* Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:
<pre>
kubectl label namespace default istio-injection=enabled
kubectl get ns --show-labels
</pre>
* Install istio addons
<pre>
kubectl apply -f samples/addons
</pre>
* Visualize your cluster using Kiali Dashboard
<pre>
istioctl dashboard kiali
 </pre>
* If your cluster is created inside vagrant , expose kiali service as NodePort 
<pre>
kubectl edit svc kiali -n istio-system
kubectl get svc -n istio-system | grep kiali
</pre>
* open browser, http://nodeip:NodePort

* deploy myweb demo .
<pre>
kubectl create -f mydata-v1.yaml
kubectl create -f mydata-v2.yaml
kubectl create -f myweb.yaml
kubectl create -f gateway.yaml
kubectl create -f virtual-service.yaml
</pre>
* Monitor this web app using grafana and kiali
* Test mydata v1 and v2 by splitting traffic with kiali

* access myweb app from istion ingress-gateway external ip
<pre>
kubectl get svc -n istio-system | grep ingress
</pre>
