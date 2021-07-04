# Installing Spinnaker on K8s CLuster with S3 storage 

* create a kubernetes cluster. check your cluster is ready now
```bash
root@master:~# kubectl get nodes -o wide
NAME     STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master   Ready    control-plane,master   3m5s    v1.20.0   10.0.0.6      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.7
worker   Ready    <none>                 2m34s   v1.20.0   10.0.0.7      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.7
```
* Install java 11 on all of your nodes.
```bash
apt install default-jdk -y
java -version
```
* Install Halyard.Halyard is a command-line administration tool that manages the lifecycle of your Spinnaker deployment, including writing & validating your deployment’s configuration, deploying each of Spinnaker’s microservices, and updating the deployment.
```bash
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
hal -v
. ~/.bashrc
```
* Create a kubernetes service account
```bash
CONTEXT=$(kubectl config current-context)

# This service account uses the ClusterAdmin role -- this is not necessary, 
# more restrictive roles can by applied.
kubectl apply --context $CONTEXT \
    -f https://www.spinnaker.io/downloads/kubernetes/service-account.yml

TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
```
* configure kubernetes rbac
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: spinnaker-role
rules:
- apiGroups: [""]
  resources: ["namespaces", "configmaps", "events", "replicationcontrollers", "serviceaccounts", "pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "services", "secrets"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "get"]
- apiGroups: ["apps"]
  resources: ["controllerrevisions", "statefulsets"]
  verbs: ["list"]
- apiGroups: ["extensions", "apps"]
  resources: ["deployments", "deployments/scale", "replicasets", "ingresses"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
# These permissions are necessary for halyard to operate. We use this role also to deploy Spinnaker itself.
- apiGroups: [""]
  resources: ["services/proxy", "pods/portforward"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: spinnaker-role-binding
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: spinnaker-role
subjects:
- namespace: spinnaker
  kind: ServiceAccount
  name: spinnaker-service-account
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: spinnaker-service-account
 namespace: spinnaker
 ```
* First, make sure that the provider is enabled:
```bash
hal config provider kubernetes enable
```
* Then add the account
```bash
CONTEXT=$(kubectl config current-context)

hal config provider kubernetes account add my-k8s-account \
    --context $CONTEXT
```
* Distributed installation. Distributed installations are for development orgs with large resource footprints, and for those who can’t afford downtime during Spinnaker updates.
```bash
hal config deploy edit --type distributed --account-name my-k8s-account
```
* After you install it, you might need to update the $PATH to ensure Halyard can find it, and if Halyard was already running you might need to restart it to pick up the new $PATH:
```bash
hal shutdown
```
```bash
Then invoke any hal command to restart the Halyard daemon.
```
* S3 storage persistent . You need an AWS Account, and a role or user configured with s3 permissions. Create a role for s3 full access and get aws account access key and secret key.
<pre>
YOUR_ACCESS_KEY_ID=<your_access_key>
REGION=us-west-2
hal config storage s3 edit \
    --access-key-id $YOUR_ACCESS_KEY_ID \
    --secret-access-key \
    --region $REGION \
    --bucket spin-tho-861998-devsa-test
</pre>
* Finally, set the storage source to S3:
```bash
hal config storage edit --type s3
```
* list available version and set version you want to use
```bash
hal version list
hal config version edit --version 1.26.6
```
* deploy spinnaker 
```bash
hal deploy apply
```
* check pods and services are running in spinnaker namespaces.
```bash
root@master:~# kubectl get pods -n spinnaker
NAME                                READY   STATUS    RESTARTS   AGE
spin-clouddriver-669ff46d7d-v7bpv   1/1     Running   0          6m1s
spin-deck-b69cdb545-tndn2           1/1     Running   0          6m1s
spin-echo-548bc99dbf-jkq4n          1/1     Running   0          6m
spin-front50-76957bc5f4-m6bfm       1/1     Running   0          5m59s
spin-gate-5f788dbc9c-b72lt          1/1     Running   0          6m4s
spin-orca-f5dcc4b96-h8r77           1/1     Running   0          6m2s
spin-redis-6f84989bfd-l9zpk         1/1     Running   0          6m3s
spin-rosco-7756d9897f-mctkl         1/1     Running   0          5m59s
```
* Change the service type to either Load Balancer or NodePort
```bash
kubectl -n spinnaker edit svc spin-deck
kubectl -n spinnaker edit svc spin-gate
```
* check services
```bash
root@master:~# kubectl get svc -n spinnaker
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
spin-clouddriver   ClusterIP   10.97.109.147    <none>        7002/TCP         7m
spin-deck          NodePort    10.100.127.81    <none>        9000:32059/TCP   6m58s
spin-echo          ClusterIP   10.103.57.230    <none>        8089/TCP         6m58s
spin-front50       ClusterIP   10.97.184.34     <none>        8080/TCP         7m
spin-gate          NodePort    10.105.120.163   <none>        8084:30388/TCP   7m2s
spin-orca          ClusterIP   10.102.169.0     <none>        8083/TCP         7m1s
spin-redis         ClusterIP   10.103.59.63     <none>        6379/TCP         7m
spin-rosco         ClusterIP   10.98.23.162     <none>        8087/TCP         6m59s
```
* Update config and redeploy
```bash
hal config security ui edit --override-base-url "http://<LoadBalancerIP>:9000"
hal config security api edit --override-base-url "http://<LoadBalancerIP>:8084"
hal deploy apply
```
* If you used NodePort
```bash
hal config security ui edit --override-base-url "http://10.0.0.7:32059"
hal config security api edit --override-base-url "http://10.0.0.7:30388"
hal deploy apply
```
* check services again
```bash
kubectl get svc -n spinnaker
```
Browse http://10.0.0.7:32059 ## spin-deck service
```bash
Demo: https://thenewstack.io/build-extensible-ci-cd-pipelines-spinnaker-kubernetes/
```
```bash
You can setup Automatic Triggers: To use Docker Hub Triggers , You have to install docker registry in spinnaker . Reference to Official docs under step2 ( select cloud providers: docker reg)
```
example 
```bash
pipeline1: stage1: deploy an simple app -- ( auto trigers --> docker hub ) -- new image build --> pipeline build
pipeline2: stage1: manual judgement ( auto triggers --> pipeline1 success ) Choose Continue or Stop --> If Continue
pipeline2: stage2: delete manifest ( manifest --> simpel app )
```













