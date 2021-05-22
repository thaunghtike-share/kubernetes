# kube-ingress

* Firstly,create a kubernetes cluster with kubeadm ( one master and two worker nodes) and 
To install nginx ingress. create a private repo in your docker hub name nginx-ingress
<pre>
$ docker login
$ git clone https://github.com/nginxinc/kubernetes-ingress/
$ cd kubernetes-ingress/
$ git checkout v1.11.2
$ apt install make
$ make debian-image PREFIX=tho861998/nginx-ingress TARGET=container ( here tho861998 is my docker hub username )
$ make push PREFIX=tho861998/nginx-ingress
* Clone my github repo
<pre>
$ git clone https://github.com/tho861998/kubernetes.git
$ cd kubernetes/ingress/
</pre>
<pre>
* Go to nginx official documentation and make sure Docker Login 
* and create a private repo named dockerhub-username/nginx-ingress in your docker hub registry.
* Then run kubectl apply blah blah like common/sa-and-ns.yaml ( run all kubectl cmd staring from ns till nodeport svc)
* Check kubectl get all -n nginx-ingress ( if nginx-controller pods are running)
</pre>
* create a haproxy LB with centos vm 
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
