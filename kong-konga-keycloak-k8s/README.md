# kong-konga-keycloak-k8s
* You have to create a k8s cluster at first
```bash
thaunghtikeoo@thaunghtikeoo:~$ kubectl get nodes
NAME                                STATUS    ROLES     AGE       VERSION
aks-agentpool-27434365-vmss000000   Ready     agent     16m       v1.19.11
aks-agentpool-27434365-vmss000001   Ready     agent     16m       v1.19.11
```
* Instal kong helm repo 
```bash
helm repo add kong https://charts.konghq.com
helm rep update
```
* pull kong helm charts to edit postgres pod's mount Path
```bash
helm pull kong/kong
tar xzvf kong-*
cd kong/charts/postgresql/
vim values.yaml --> change mountPath (from /bitnami/bostgresql to /bitnami )
cd
kubectl create ns kong
helm -n kong install kong ./kong/ --set admin.enabled=true --set admin.http.enabled=true --set postgresql.enabled=true   --set postgresql.postgresqlUsername=kong  --set postgresql.postgresqlDatabase=kong --set env.database=postgres --set postgresql.postgresqlPassword=postgres
```
* Check kong & postgre pod are running. You will see Postgres pod is pending. Because pvc is also pending. So, you have to create a new pv and pvc for postgre pod.
```bash
kubectl get pvc data-kong-postgresql-0 -n kong -o yaml >pvc.yaml
kubectl delet pvc data-kong-postgresql-0 -n kong --force
```
* create a pv and pvc named data-kong-postgresql-0 for postgre
```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
```
* check your created pvc bounds to pv successfully.
```bash
NAME                     STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-kong-postgresql-0   Bound     pvc-33c6155c-a8de-4ad2-bf0e-2b547fed408f   8Gi        RWO            default        2m6s
```
* check all pods are running in kong namespaces
```bash
kubectl get pods -n kong
kubectl get svc -n kong
```
* make sure kong-kong-admin and kong-kong-proxy services are exposed as LoadBalancer.
```bash
kubectl edit svc kong-kong-admin -n kong
kubectl edit svc kong-kong-proxy -n kong
```
* Install konga dashboard
```bash
 git clone https://github.com/pantsel/konga.git
 cd konga/charts/konga
 helm -n kong install konga ./
 ```
 * Check kong & konga pods are running and expose konga svc as LoadBalancer.
 ```bash
 kubectl edit svc konga -n kong
 ```
 * check services
 ```bash 
thaunghtikeoo@thaunghtikeoo:~$ kubectl get svc -n kong
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                         AGE
kong-kong-admin            LoadBalancer   10.0.239.104   52.177.245.159   8001:30070/TCP,8444:31065/TCP   27m
kong-kong-proxy            LoadBalancer   10.0.38.156    52.177.244.252   80:31476/TCP,443:31404/TCP      27m
kong-postgresql            ClusterIP      10.0.179.127   <none>           5432/TCP                        27m
kong-postgresql-headless   ClusterIP      None           <none>           5432/TCP                        27m
konga                      LoadBalancer   10.0.18.158    20.75.118.177    80:30273/TCP                    8m11s
kubernetes                 ClusterIP      10.0.0.1       <none>           443/TCP                         54m
```
* browe http://20.75.118.177 . Sing up and login . Create a new connection with name ( kong-kong-admin ) and url ( http://kong-kong-admin:8001 )

* Let’s start with creating the Keycloak deployment and service: If postgres not running: check pvc and pull helm chart to change postgres pod's mountPath -->/bitnami/
```bash
 helm repo add bitnami https://charts.bitnami.com/bitnami
 helm repo update
 helm install keycloak-db bitnami/postgresql-ha
 kubectl create -f keycloak.yaml
```
* keycloak youtube playlist
```bash
https://www.youtube.com/watch?v=g8LVIr8KKSA&list=PLPZal7ksxNs0mgScrJxrggEayV-TPZ9sA
```

