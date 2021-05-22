# kube-ingress

* Firstly,create a kubernetes cluster with kubeadm ( one master and two worker nodes) and 
To install nginx ingress. create a private repo in your docker hub name nginx-ingress
<pre>
docker login
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/
git checkout v1.11.2
apt install make
make debian-image PREFIX=tho861998/nginx-ingress TARGET=container #( here tho861998 is my docker hub username )
make push PREFIX=tho861998/nginx-ingress
cd deployments
kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml
kubectl apply -f common/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f rbac/ap-rbac.yaml
kubectl apply -f common/ingress-class.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml
kubectl apply -f common/global-configuration.yaml
kubectl apply -f common/crds/appprotect.f5.com_aplogconfs.yaml
kubectl apply -f common/crds/appprotect.f5.com_appolicies.yaml
kubectl apply -f common/crds/appprotect.f5.com_apusersigs.yaml
kubectl apply -f daemon-set/nginx-ingress.yaml
kubectl get pods -n nginx-ingress
</pre>
Get Access to the Ingress Controller
<pre>
If you created a daemonset, ports 80 and 443 of the Ingress controller container are mapped to the same ports of the node where the container is running. To access the Ingress controller, use those ports and an IP address of any node of the cluster where the Ingress controller is running.
</pre>
LoadBalancing with Haproxy .create a haproxy LB with centos vm 
* vim /etc/haproxy/haproxy.cfg
<pre>

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2     #Log configuration
 
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000                
    user        haproxy             #Haproxy running under user and group "haproxy"
    group       haproxy
    daemon
 
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
 
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
frontend http_front
  bind *:80
  stats uri /haproxy?stats
  default_backend http_back

backend http_back
  balance roundrobin
  server kube  <worker-node1-ip>:80
  server kube  <worker-node2-ip>:80
</pre>
<pre>
* add haproxy ip with your ingresss domain name to all nodes in /etc/hosts file
* when a user calls domain name which binds with haproxy lb ip, haproxy will load to worker node port 80. On worker nodes port 80, ingress controller is running. ( On previous stage, you deployed ingress controller daemonset, so ingress controllers are running on master and worker nodes)
* since ingress controller is running, your domain will be redirected to service which has domain  or path) 
</pre>
* Note. All are deployed on kubernetes cluster 1.18.0 . If your cluster is 1.20.X , change your ingress configuration version and format.
<pre>
* kubectl apply -f nginx-deploy-svc-ingress-red.yaml
* kubectl apply -f nginx-deploy-blue.yaml
* kubectl apply -f nginx-svc-ingress-blue.yaml
* kubectl apply -f nginx-deploy-green.yaml
* kubectl apply -f nginx-svc-ingress-green.yaml
</pre>
* Don't forget to add haproxy ip in /etc/hosts of each node .
Note
<pre>
If u don't want to load haproxy, expose ingress as nodeport type=LB
</pr>
