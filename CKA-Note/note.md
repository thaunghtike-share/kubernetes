# CKA-Note For Exam

* Core Concepts
<pre>

$ alias k='kubectl'
$ alias w='watch kubectl'
$ use 'kubectl run nginx --image=image --port=80' instead of 'kubectl run nginx --image=nginx'  ( don't forget  containerPort && containerPort means target-port)
$ kubectl expose pod or deploy name --type --target-port --port --name
</pre>

* Scheduling 
<pre>
$ scheduling --> using nodeName filed under spec is a good practice to manual schedule your pod to node  ( use nodeSelector for deployments & replication)
<pre>
spec:
  containers:
  -  image: nginx
     name: nginx
  nodeName: node01
</pre>
$ to view node labesl --> $ kubectl get pods/nodes --show-labels |  grep (OR) kubectl get pods/nodes --selector app=nginx
$ to check taints --> kubectl describe node node-name | grep Taints (taints -->node | tolerations --> pods)
$ to tain a node --> kubectl taint node node01 spray=mortein:NoSchedule/NoExecute/PreferNoSchedule

$ If you want to deploy pods on master node, remove taints from master node . But it 's not a good idea to use for productio.
$ node affinity looks like nodeSelector ( node affinity has more options like In, Exists, NotIn parameter)
$ to assign label wiith just key ( no value ) --> kubectl label node node1 something=
$ resoure requests --> container request ထားတဲ့ resource ပေးနိင်တဲ့ node မှာ scheduled လုပ် ပေးတာ  ( pod     deploy လုပ်မယ့် node မှာ resoure ဒီလောက်ကို ပေးပါဆိုပြီး request တာ)
    resource limits --> container m limit ထားတဲ့ resource ထက် ပ်ု size များနေရင် terminated  လုပ်တာ
$ use --all-namespaces when you want to get something from all namespaces
$ kubectl get pods --all-namespaces နဲံရှာ node name နဲ့ POD နာမည် ဆုံးနေရင် static pods  ( staticPods တွေရဲ့ path ကို သိဖို့ အရင်ဆုံး /etc/systemd/system/kubeletd.service/10-config ကိုကြည့်ရမယ် Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
အဲ့က နေ /var/lib/kubelet/config.yml မှာ staticpod path ရှာ) you cant view what pods are running , but  you can't delete static pods. if you want to delete static pod, remove configure file from staticPod path
$custom-scheduler create --> different port with default-scheduler --> use to schedulerName: name field under spec in configuration file to deploy pod
</pre>

* Logging & Monitoring
<pre>
--> metric-server ( memory ပဲပြ disks logs တွေမပြ ) ---> kubectl top node/pod
</pre>

* Application Lifecycle Management
<pre>
$ rolling update---> default update strategy for updating replicas --> kubectl describe deploy နဲ့ကြည့်ရင်တတွေ့ရတယ် ( deployment မှာ image version အသစ် update လုက်မယ်ဆို rolling update သည် application down မသွားအောင် replica တစ်ခုချင်း down လိုက် update လိုက် လုပ်သွားတာ)
$ Recreate က replicas အကုန်လုံကို down  ပြီးမှ replicas အသစ်တွေကို upgrade လုပ်တာ application down သွားတယ် )
$ kubectl rollout status deploy နဲ့ replica တွေ rolling upgrade status ကိုကြည့်)
$ kubectl rollout history deploy/name နဲ့ revision ကြည့်တာ deploy တခါလုပ်ရင် revision 1)
$ kubectl rollout undo deploy/name --to-revision=revision_id (roll back တာ)
</pre>

