# Deploying Elastic Cloud On Kubernetes ( ECK )

 create a kubernetes cluster at first. you can check how to create k8s cluster using kubeadm in this repo.
```bash
root@kmaster:~# kubectl get nodes -o wide
NAME       STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kmaster    Ready    control-plane,master   3m52s   v1.20.0   10.0.0.4      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker1   Ready    <none>                 3m12s   v1.20.0   10.0.0.5      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
kworker2   Ready    <none>                 3m14s   v1.20.0   10.0.0.6      <none>        Ubuntu 20.04.2 LTS   5.4.0-1047-azure   docker://20.10.6
```
 Install custom resource definitions and the operator with its RBAC rules:
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/1.6.0/all-in-one.yaml
```
 check elastic operator statefulset is ready now. 
```bash
kubectl get all -n elastic-system
kubectl get sts -n elastic-system
```
Before starting eck deploying, we will create a storage class for dyanmic provisioning by using openebs cstor, we will start creating cstor persistent volume . First step is to deploy OpenEbs operators. You can read about openebs detail in official docs.
<pre>
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
</pre>
Before installing cstor operator. You must enable iscsid service on all nodes.
<pre>
systemctl enable --now iscsid
</pre>
Install latest cstor operator.
<pre>
kubectl apply -f https://openebs.github.io/charts/cstor-operator.yaml
</pre>
check all pods are running in openebs namespace.
<pre>
kubectl get pods -n openebs
</pre>
```bash
root@kmaster:~# kubectl get pods -n openebs
NAME                                              READY   STATUS    RESTARTS   AGE
cspc-operator-6d796d7b8f-qk28b                    1/1     Running   0          31m
cstor-storage-drr8-7557675774-fgbrq               3/3     Running   0          3m10s
cstor-storage-hr48-7864c69c47-hldjd               3/3     Running   0          3m9s
cstor-storage-vq4d-fd55c584c-f9mlc                3/3     Running   0          3m8s
cstor-storage-wvdw-64f6bcf459-7795b               3/3     Running   0          3m8s
cvc-operator-849dff65c-x6pzb                      1/1     Running   0          31m
maya-apiserver-55db98dc94-d8tmq                   1/1     Running   0          42m
openebs-admission-server-574cf7c984-d9tjk         1/1     Running   0          42m
openebs-cstor-admission-server-5d868c64d4-cxfc6   1/1     Running   0          31m
openebs-cstor-csi-controller-0                    6/6     Running   0          31m
openebs-cstor-csi-node-5s4ws                      2/2     Running   0          31m
openebs-cstor-csi-node-j9w4z                      2/2     Running   0          31m
openebs-cstor-csi-node-m7xt9                      2/2     Running   0          31m
openebs-cstor-csi-node-q8rg6                      2/2     Running   0          31m
openebs-localpv-provisioner-9f598b455-7pz6f       1/1     Running   0          42m
openebs-ndm-94pmf                                 1/1     Running   0          30m
openebs-ndm-dmcwm                                 1/1     Running   0          31m
openebs-ndm-fppmz                                 1/1     Running   0          31m
openebs-ndm-operator-59ffcfc949-m5w29             1/1     Running   0          31m
openebs-ndm-vslck                                 1/1     Running   0          31m
openebs-provisioner-65749f64fd-hcxr8              1/1     Running   0          42m
openebs-snapshot-operator-5cfbb6fdb5-ffxvr        2/2     Running   0          42m
```
Check that blockdevices are created:. See one blockdevice means one disk attached to node.
<pre>
kubectl get blockdevices -n openebs -o wide
</pre>
Provision a CStorPoolCluster. Use blockdevice which are not mounted or formatted. Don't use blockdevice like ext4.
```yaml
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
  name: cstor-storage
  namespace: openebs
spec:
  pools:
    - nodeSelector:
        kubernetes.io/hostname: "kmaster"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-6c0f0ac1aa99c9148ee3d187d08e32a4"
            - blockDeviceName: "blockdevice-b4458e89bd5442f64ef890e247176015"
      poolConfig:
        dataRaidGroupType: "stripe"

    - nodeSelector:
        kubernetes.io/hostname: "kworker1" 
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-67651fd6266f40a8e4ed7301c4e30a2e"
            - blockDeviceName: "blockdevice-680eb18dd7b630948a2d81f49d65eca9"
      poolConfig:
        dataRaidGroupType: "stripe"
   
    - nodeSelector:
        kubernetes.io/hostname: "kworker2"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-1e6d651fd9a01cc889a0d3e9a12df01d"
            - blockDeviceName: "blockdevice-b82a1982cbe0a26ba9092b9de26daf1c"
      poolConfig:
        dataRaidGroupType: "stripe"
    
    - nodeSelector:
        kubernetes.io/hostname: "kworker3"
      dataRaidGroups:
        - blockDevices:
            - blockDeviceName: "blockdevice-f202a757991e537a9c4e22a1823b4d7c"
            - blockDeviceName: "blockdevice-f98f72fba1346af698937aefbb70ec91"
      poolConfig:
        dataRaidGroupType: "stripe"
```
Apply the modified CSPC YAML.
<pre>
kubectl apply -f cspc.yml
</pre>
Check if the pool instances report their status as 'ONLINE'.
```bash
kubectl get cscp -n openebs
```

```bash
root@kmaster:~# kubectl get cspc -n openebs
NAME            HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
cstor-storage   4                  4                      4                  8m9s
```
```bash
kubectl get cspi -n openebs
```

```bash
root@kmaster:~# kubectl get cspi -n openebs
NAME                 HOSTNAME   FREE    CAPACITY      READONLY   PROVISIONEDREPLICAS   HEALTHYREPLICAS   STATUS   AGE
cstor-storage-drr8   kmaster    1920G   1920000074k   false      0                     0                 ONLINE   9m54s
cstor-storage-hr48   kworker1   1920G   1920000074k   false      0                     0                 ONLINE   9m53s
cstor-storage-vq4d   kworker2   1920G   1920000074k   false      0                     0                 ONLINE   9m53s
cstor-storage-wvdw   kworker3   1920G   1920000074k   false      0                     0                 ONLINE   9m52s
```
Check blockdevices status.

```bash
root@kmaster:~# kubectl get blockdevices -n openebs
NAME                                           NODENAME   SIZE            CLAIMSTATE   STATUS     AGE
blockdevice-00dbd3173aff10684a95b8c4da20a5fe   kworker2   1099501142016   Unclaimed    Active     34m
blockdevice-051458b8713c411d362836334e873ac4   kworker3   1099501142016   Unclaimed    Inactive   34m
blockdevice-0626988d63953239ac321fb2e0fdc5f9   kmaster    1099501142016   Unclaimed    Active     34m
blockdevice-1e6d651fd9a01cc889a0d3e9a12df01d   kworker2   1099511627776   Claimed      Active     73m
blockdevice-2ccd2c22fd26a64350de575e6f0f63e7   kmaster    1099501142016   Unclaimed    Active     34m
blockdevice-3f1521ca4ba60a0768997ec8af173c67   kworker3   4292870144      Unclaimed    Active     73m
blockdevice-54cd81fb0c088cdd14cc6457e82a4eb0   kworker3   1099501142016   Unclaimed    Inactive   34m
blockdevice-67651fd6266f40a8e4ed7301c4e30a2e   kworker1   1099511627776   Claimed      Active     73m
blockdevice-680eb18dd7b630948a2d81f49d65eca9   kworker1   1099511627776   Claimed      Active     73m
blockdevice-6c0f0ac1aa99c9148ee3d187d08e32a4   kmaster    1099511627776   Claimed      Active     73m
blockdevice-7a88de9fe00794289c281780f1b98cb4   kmaster    4292870144      Unclaimed    Active     73m
blockdevice-83c6820ff42807ab41817279d00862c1   kworker1   17177772032     Unclaimed    Active     73m
blockdevice-93fbbea4eb92daf878e8bfffd47f0651   kworker2   4292870144      Unclaimed    Active     73m
blockdevice-b4458e89bd5442f64ef890e247176015   kmaster    1099511627776   Claimed      Active     73m
blockdevice-b82a1982cbe0a26ba9092b9de26daf1c   kworker2   1099511627776   Claimed      Active     73m
blockdevice-c4ce891150b55fd92f210ffb3f66d3f9   kworker2   1099501142016   Unclaimed    Active     34m
blockdevice-f202a757991e537a9c4e22a1823b4d7c   kworker3   1099511627776   Claimed      Active     73m
blockdevice-f98f72fba1346af698937aefbb70ec91   kworker3   1099511627776   Claimed      Active     73m
```
You see? 8 blockdevices are claimed following cspc yaml file. Once your pool instances have come online, you can proceed with volume provisioning.Create a storageClass to dynamically provision volumes using OpenEBS CSI provisioner. A sample storageClass:
```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: cstor-csi
   provisioner: cstor.csi.openebs.io
   allowVolumeExpansion: true
   parameters:
     cas-type: cstor
     # cstorPoolCluster should have the name of the CSPC
     cstorPoolCluster: cstor-storage
     # replicaCount should be <= no. of CSPI
     replicaCount: "4"
    
 ```
Apply storage class.
<pre>
kubectl apply -f cstor-sc.yml
</pre>
You will need to specify the correct cStor CSPC from your cluster and specify the desired `replicaCount` for the volume. The `replicaCount` should be less than or equal to the max pool instances available. Openebs is a dynamic storage solution. So, you don't need to deploy pv. Openebs provide pv as container-attached storage.

```bash
root@kmaster:~# kubectl get pods -n openebs | grep provision
openebs-localpv-provisioner-9f598b455-7pz6f       1/1     Running   0          53m
openebs-provisioner-65749f64fd-hcxr8              1/1     Running   0          53m

```
```bash
kubectl get sc
```
deploy elaticsearch cluster.
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: elastic-system
spec:
  version: 7.13.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        storageClassName: cstor-csi
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```
```bash
kubectl apply -f es.yaml
```
check elasticsearch statefulset and pvc is ready. 
```
root@kmaster:~# kubectl get all -n elastic-system
NAME                          READY   STATUS    RESTARTS   AGE
pod/elastic-operator-0        1/1     Running   0          6m29s
pod/quickstart-es-default-0   1/1     Running   0          73s

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/elastic-webhook-server    ClusterIP   10.98.96.246     <none>        443/TCP    42m
service/quickstart-es-default     ClusterIP   None             <none>        9200/TCP   73s
service/quickstart-es-http        ClusterIP   10.111.216.114   <none>        9200/TCP   74s
service/quickstart-es-transport   ClusterIP   None             <none>        9300/TCP   74s

NAME                                     READY   AGE
statefulset.apps/elastic-operator        1/1     42m
statefulset.apps/quickstart-es-default   1/1     73s
```
next step is to deploy kibana instance.
```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
  namespace: elastic-system
spec:
  version: 7.13.0
  count: 1
  elasticsearchRef:
    name: quickstart
  http:
    tls:
      selfSignedCertificate:
        disabled: true 

EOF
```
check all instances are ready and running inside elastic-system namespace.
```bash
root@kmaster:~# kubectl get pods -n elastic-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
elastic-operator-0               1/1     Running   0          15m     10.47.0.12   kworker1   <none>           <none>
quickstart-es-default-0          1/1     Running   0          10m     10.47.0.13   kworker1   <none>           <none>
quickstart-kb-5b4d6869b8-hk9zc   1/1     Running   0          2m34s   10.47.0.15   kworker1   <none>           <none>
root@kmaster:~# kubectl get svc -n elastic-system 
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
elastic-webhook-server    ClusterIP   10.98.96.246     <none>        443/TCP    51m
quickstart-es-default     ClusterIP   None             <none>        9200/TCP   10m
quickstart-es-http        ClusterIP   10.111.216.114   <none>        9200/TCP   10m
quickstart-es-transport   ClusterIP   None             <none>        9300/TCP   10m
quickstart-kb-http        ClusterIP   10.101.144.16    <none>        5601/TCP   2m46s
```
next is to expose quickstart-kb-http as nodeport/loadbalancer to access from cluster outside. with loadbalancer, you will need to install metallb 
```bash
kubectl edit svc quickstart-kb-http -n elastic-system
```
now ready to access kibana dashboard.
```
root@kmaster:~# kubectl get svc -n elastic-system 
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
elastic-webhook-server    ClusterIP   10.98.96.246     <none>        443/TCP          53m
quickstart-es-default     ClusterIP   None             <none>        9200/TCP         12m
quickstart-es-http        ClusterIP   10.111.216.114   <none>        9200/TCP         12m
quickstart-es-transport   ClusterIP   None             <none>        9300/TCP         12m
quickstart-kb-http        NodePort    10.101.144.16    <none>        5601:32088/TCP   4m40s
```
browse to http://<nodeip>:<nodeport> with username: elastic and password: 
```bash
  kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' -n elastic-system | base64 --decode; echo
```
to view logs from multiple pods: we will deploy filebeat on kubernetes. download manifests file.
```
curl -L -O https://raw.githubusercontent.com/elastic/beats/7.13/deploy/kubernetes/filebeat-kubernetes.yaml
```
edit namespace , elastic host , password and ssl.verification_mode: "none". final yaml file looks like 
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: elastic-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    # To enable hints based autodiscover, remove `filebeat.inputs` configuration and uncomment this:
    #filebeat.autodiscover:
    #  providers:
    #    - type: kubernetes
    #      node: ${NODE_NAME}
    #      hints.enabled: true
    #      hints.default_config:
    #        type: container
    #        paths:
    #          - /var/log/containers/*${data.kubernetes.container.id}.log

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
      ssl.verification_mode: "none"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elastic-system
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.13.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: https://quickstart-es-http
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: p12gt2IpDX170q719ad6kvEa
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          #privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0640
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          # When filebeat runs as non-root user, this directory needs to be writable by group (g+w).
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: elastic-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
    - replicasets
  verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elastic-system
  labels:
    k8s-app: filebeat
---
```
setup ingress for kibana
```yaml
 apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  name: eck-devsa
  namespace: elastic-system
spec:
  rules:
  - host: kibana.apps-dev.devsa-rnd.frontiir.net
    http:
      paths:
      - backend:
          serviceName: quickstart-kb-http
          servicePort: 5601
        path: /
  tls:
  - hosts:
    - kibana.apps-dev.devsa-rnd.frontiir.net
    secretName: frontiir-wildcard-selfsigned-crt-secret
```
go back to kibana dashboard. create index filebeat-* . 
Thanks!
 
