## frontiir-k8s-cluster
<pre>
  docs mhr node 9 khu pl shi tl
  test yin rke run phox vm 1 khu lo ml
  so , total vm 10 khu
  1 khu ka host yl nginx ko LB a phyit use phox
  3 khu ka rancher 3khu
  6 khu ka downstreamed cluster
 </pre>
 * keepalived nk haproxy ma use wooo
 <pre>
  ancher ko helm nk deploy pee yin service ya lr ml
  ae svc ko mha NodePort expose lite
  ae ya lar dae nodeportip nk node ko Nginx Vm mhr LB apyit pyun use
  ae Nginx vm yae Ip ko domain add /etc/hosts mhr
 </pre>
*  Use the following as the contents of /etc/nginx/sites-available/default:
<pre>
  upstream web_backend {
  server 10.11.12.51;
  server 10.11.12.52;
  }
  server {
  listen 80;
  location / {
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_pass http://web_backend;
  }
  }
</pre>
