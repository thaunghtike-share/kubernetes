# ssl ingress demo

Prerequites
<pre>
kubernetes cluster
public domaain
</pre>

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
check if ingress pods are running
```bash
root@kmaster:~# kubectl get pods -o wide -n nginx-ingress
NAME                  READY   STATUS    RESTARTS   AGE    IP          NODE       NOMINATED NODE   READINESS GATES
nginx-ingress-8mtfp   1/1     Running   0          110m   10.44.0.1   kworker1   <none>           <none>
nginx-ingress-wnd49   1/1     Running   0          110m   10.36.0.1   kworker2   <none>           <none>
```
Instatll cert-manager latest version.
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```
Check status of cert-manager pods.
```bash
root@kmaster:~# kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7dd5854bb4-v5nd6              1/1     Running   0          111m
cert-manager-cainjector-64c949654c-ps87c   1/1     Running   0          111m
cert-manager-webhook-6bdffc7c9d-w9vs6      1/1     Running   0          111m
```
Let's see ssl ingress is running on worker nodes' port 443. 
```bash
curl https://<worker1_ip> -k ## ignore ssl
```
Here we need a loadbalancer to traffic these ingress controllers. We will configure HAporxy as LB. LoadBalancing with Haproxy .create a haproxy LB with centos vm 
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
  bind *:443
  stats uri /haproxy?stats
  default_backend http_back

backend http_back
  balance roundrobin
  server worker1  <worker-node1-ip>:443
  server worker2  <worker-node2-ip>:443
</pre>
Restart haproxy service
```bash
systemctl restart haproxy
```
Verify haproxy is working .
```bash
curl https://<haproxy_node_ip>
```
Next step is to add these haproxy node Public Ip to domain which you want to use. Please buy a domain if you don't have. Add dns type A record in this domain you purchased. Let's deploy ssl ingress. Create clusterissuer at first.
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cluster-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: zwe.zzzin@gmail.com
    privateKeySecretRef:
      name: letsencrypt-cluster-issuer-key
    solvers:
    - http01:
       ingress:
         class: nginx
```
Apply clusterissuer.yml
```bash
kubectl apply -f clusterissuer.yml
```
Check your clusterissuer
```bash
root@kmaster:~# kubectl get clusterissuer
NAME                         READY   AGE
letsencrypt-cluster-issuer   True    80m
```
Next step is to deploy simple nginx deployment.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```
```bash
kubectl apply -f nginx.yml
```
Then expose nginx deployment as service 
```bash
kubectl expose deploy nginx --port 80
```
```bash
root@kmaster:~# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   129m
nginx        ClusterIP   10.101.67.159   <none>        80/TCP    111m
```
Next, deploy your ingress yaml file.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: example-app
spec:
  tls:
  - hosts:
    - thaunghtikeoo.media  ##public domain name
    secretName: example-app-tls
  rules:
  - host: thaunghtikeoo.media ##public domain name
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: nginx
            port: 
              number: 80
```
Apply ingress
```bash
kubectl apply -f ingress.yml
```
The final step is to expose certificate .
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-app
  namespace: default
spec:
  dnsNames:
    - thaunghtikeoo.media
  secretName: example-app-tls
  issuerRef:
    name: letsencrypt-cluster-issuer
    kind: ClusterIssuer
```
```bash
kubectl apply -f certificate.yml
```
Check status of certificate you exposed.
```bash
root@kmaster:~# kubectl get certificate
NAME          READY   SECRET            AGE
example-app   True   example-app-tls   103s
```
Test your ssl ingress is working or not.
```bash
curl https://thaunghtikeoo.media (OR ) go to browser https://thaunghtikeoo.media
```
You will see an error like this
```bash
curl: (35) error:14094410:SSL routines:ssl3_read_bytes:sslv3 alert handshake failure
```
Don't worry! you can fix it by running a configmap.
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
  location-snippets: |
    add_header 'Access-Control-Allow-Origin' '*';
  ssl-protocols: "TLSv1.1 TLSv1.2 TLSv1.3"
```
```bash
kubectl apply -f ssl_cm.yml
```
You will see your ssl ingress is working right now.
```bash
curl https://thaunghtikeoo.media -k
```
Reference for haproxy ingress with ssl; https://www.haproxy.com/blog/enable-tls-with-lets-encrypt-and-the-haproxy-kubernetes-ingress-controller/ 
```bash
frontend http_front
  bind *:443
  stats uri /haproxy?stats
  default_backend http_back

backend http_back
  balance roundrobin
  server worker1  <worker-node1-ip>:<haproxy_ingress_nodeport>
  server worker2  <worker-node2-ip>:<haproxy_ingress_nodeport>
  ```
Thanks



