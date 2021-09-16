[toc]


# Basics:

## Kube components:
### On master node :
  - etcd : kv database info about cluster 
  - controler-manager : replica controller autoscaling etc ? 
    - node controller 
      - watch status
      - remediate situations
      - node monitor period = 5s
      - node monitor grace period = 40s
      - pod eviction period = 5m
    - Replication controller :
      - Monitor state of replicaset
    - more … 
  - kube scheduler : service loading containers on worker nodes
      - Select which node runs which pods 
          - e/g : according size of pod
  - kube apiserver : orchestrate all op in server
### On worker node:
  - kubelet : Agent used to make connect worker with kube master node
      - not deployed by kubeadm ( look at static pods )
  - kube-proxy : communication between services within cluster
      - creates services by creating iptables rules 

## Namespaces:
  - pod from namespace can resolve service in other namespace: 
    `<podname>.<namespace>.svc.<default cluster domain = cluster.local>`
# Scheduler :
## Taints:
  - Allow to taint a node. 
  - By default pods cannot been launched on a tainted node
  - Pod must have a "taint toleration" to run on a tainted pod
  - ` kubectl taint nodes node-name foo=bar:taint-effect`
    - taint effects are `NoSchedule | PreferNoSchedule | NoExecute`
  - Pods with `tolerations` are not garantee to run on a tainted node. 
  - Example of `tolerations` in a pod configuration : 
  ```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  tolerations:
  - key: "foo"
    value: "bar"
    operator: "equal"
    effect: "NoSchedule"
  ```

## NodeSelector:
Allow to specify on which node a pod have to run : 

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  nodeSelector:
    size: large
```

### How to label a node:

`kubectl label node <node-name> <label-key>=<label-value>`

### Limitations: 

We cannot make complex policies with nodeSelector such `size = Large OR size = medium` for this, there is Node Affinity

 ## Node Affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoreDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: size
              operator: Exists
```

## Affinity types:

- requiredDuringSchedulingIgnoredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution
- requiredDuringSchedulingRequiredDuringExecution ( Not yet available )

### During scheduling :

- State when a pod doesn't exists and is created for the first time
- When it's **required**, pod will not run if there is no matching nodes ( stuck on pending state ? )
- When it's **preferred**, pod will run if there is a matching node, if not still but on any node

## During Execution: 

- State when a pod is running.
- If it required and apply on a running pod, pod will be evicted to node if it doesn't match and will run on matching node

## Resources limits

By default, resource requests are 0.5 cpu, 256 mi memory.

Resource request example

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
    limits:
      memory: "2Gi"
      cpu: 2
```
### cpu unit:
- 1 cpu = 1 vcpu, 1code on gcp and azure, or one hyperthread
- can be expressed with "mili" like `cpu: 100m`  

### Memory unit: 
- can be expressed in G,M,K multiple of tens, 1k = 1000 bytes, 1M = 1 000 000 bytes, etc
- can be exressed in Gi, Mi, Ki multiple of 1024 

### Resources `requests`:

- minimal resources allocated to pod ? 

### Resources `limits`:
- By default : `cpu: 1`, `memory: 512Mi

CPU is throtled if there is a pod going beoyond its limitation

## Daemon sets

- Daemonsets are like replicaset but they're allow tu run only one copy of pod per node
- When a node is added, a replica of this pod is created
- When a node is removed, a replica of this pod is deleted
- Usefull for monitoring solution, log viewing
- For instance, `kube-proxy` is a daemonset.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      container:
      - name: monitoring-daemon
        image: monitoring-agent
```

## Static pods

- Kublet is watching `/etc/kubernetes/manifests` ( `--pod-manifest-path` arg value ) periodically to read yaml files to create pods

- Kubelet can only understand pod definitions

- Kubelet notify kube apiserver when a static pod is created

- We can use it to deploy control plane in a pod itself

- Static pods can be seen with `k get pods` command and their name will be suffixed by the node name which running on

## Running multiple schedulers

- a k8s cluster can have multiple scheduler ( default is kube-scheduler )

- it is possible to specify scheduler in definition

- if multiples copy of a scheduler are running, only one will be active at a time ( `--leader-elect` option ) or you can use `--lock-object-name ` to speciy which scheduler to use in case of multiple masters

- Example of pod with specific scheduler

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - image: nginx
      name: nginx
    schedulerName: my-custom-scheduler
  ```

- to see which scheduler is used in the current namespace for lasts pods creation: `kubectl get events`

## # Logging and monitoring

## Monitoring

Several solutions for monitoring ( prom, metrics server, datchien, etc...) 

- metrics server ( ex heapster ) in memory database allows to use`kubectl top node|pod`

## Logging

- `kubectl logs <podname>` if it's a monocontainer
- `kubectl logs <podname> <container name>`

# Application lifecycle management

## Rolling updates, and rollback

### Strategies

When we create a deployment, it create a rollout (revision). If deployment is updated, a new rollout is created ( revision 2 ). It's possible to go back in previous revision.

`kubectl rollout status deployment/myapp-deployment`

`kubectl rollout history  deployment/myapp-deployment`

There is 2 types deployment strategiees : 

- Recreate : all pods are killed then restart new version pod at a same time
- Rolling update : kill pod and replace it with new version one by one 

In a deployment there is `StrategyType` key to set if it's `Recreate` or `RollingUpdate`

In `RollingUpdate` strategy, Kubernetes will create another replicaset and adding one new pod version and kill one old pod version, and so on. It can be seen by `kubectl get rs`

### Rollback: 

`kubectl rollout undo deployment/myapp-deployment`

Create a rollout quickly : 

` kubectl set image deployment/<deployment-name> <container_name>=<image>:<tag>`

## Application configuration

### Docker commands

```
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

`docker run ubuntu-sleeper` will run the `sleep 5` command then exit

`docker run ubuntu-sleeper 10` will run the `sleep 10` command then exit

We can override entrypoint command by `docker run --entrypoint-command sleep2.0 ubuntu-sleeper 10`

### Commands and arguments in k8s

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: ["sleep2.0"]
      args: ["10"]      
```

### Env vars in k8s

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      env:
        - name: FOO
          value: BAR
```

### Configmaps

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      envFrom:
      - configMapRef:
          name: app-configmap
```

1. Create `configMap`
2. Inject it to the pod

- Imperative way : 

  - `kubectl create configmap <name> --from-literal=<key>=<value>  --from-literal=<key2>=<value2>`
  - `kubectl create configmap <name> --from-file=<path-to-file>`

- Declarative way:

  - `kubectl create -f `

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-config
    data:
      FOO: bar
      TRUC: fricadin
    ```

View configmap `kubectl get configmaps`

## Configure secrets in apps

Similar to configmaps but encrypted

- Imperative way :
  
- ` kubectl create secret <secret-name> generic --from-literal=DB_Host=mysql --from-literal=DB_Pass=DbPassword`
  
- Declarative way
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  # ¯\_(ツ)_/¯
  DB_Host: <encoded base64 host> 
  DB_User: <encoded base64 user> 
  DB_Password: <encoded base64 password> 
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      # Inject secret like this...
      envFrom:
      - secretRef:
          name: app-secret
      # OR 
      env:
        - name: DB_Password
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_Password
```

## Init containers: 

Init container is a ephemeral container created before creation of container of pods :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```

It's possible to specify multiple initContainers. Each of them will be executed one at time in sequential order. 
If any of `initContainers` fails, Kube will restart pod until  `initContainer` succeeds 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

More : https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

# Cluster Maintenance

## OS upgrades:

### Pods eviction : 

If node come down more than 5m ( `kube-controller-manager --pod-eviction-timeout=5m0s ...` ), pods will be terminated from that node, kubernetes will consider node as dead.

If node come back online after eviction, it is considered as blank node. New pods can be launched.

### Gracefull way :

To gracefully put a node offline, we must drain the node with `kubectl drain <node-name>`. Pods will be stopped and recreated on other node. No pods cannot be schedule on this node. To make pod schedulable again : `kubectl uncordon <node-name>`

We can set a node unschedullable with `kubectl cordon <node-name>` . No pods will be scheduled on this node but pods who are running on node will not be stopped. 

## Kubernetes releases:

Versions of k8s are formated as `<major>.<minor>.<patch>.

- minor is out every month
- all component of k8s have the same version except coredns and etcd

Refs: 

- https://kubernetes.io/docs/concepts/overview/kubernetes-api/

Here is a link to kubernetes documentation if you want to learn more about this topic (You don't need it for the exam though):

- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md

- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md



## Upgrade process:

### Components version compatibility

- `kube-apiserver` must have `x` version (e.g `1.10`)
- `controller-manager` and `kube-scheduler` can be at `x-1` : `1.9` or `1.10`
- `kubelet` and `kube-proxy` can be at `x-2` : `1.8` or `1.9` or `1.10`
- `kubectl` can be between `x-1` and `x+1` : `1.11`, `1.10`, `1.9`

It is recommanded to update one minor version at time 

### Hardway : 

- Just update manually components, lol

### Upgrade strategy

- Upgrade first master then many possibilities for nodes
  - yolops way : upgrade all nodes at once
  - one node at a time by drainning nodes
  - Adding another node with newer version, 
    - migrate pods from a node to new one
    - remove old node

### kubeadm upgrade

`kubeadm upgrade plan`  will show current versions, latest version available. **kubeadm will not upgrade kubelet**

1. upgrade kubeadm itself
2. `kubeadm upgrade apply v1.12.0`
3. upgrade kubelet version on master node. 
   1. `apt-get upgrade -y kubelet=1.12.0-00`
   2. `systemctl restart kubelet`
4. upgrade kubeadm and kubelet on DRAINED nodes:
   1. `apt-get upgrade -y kubeadm=1.12.0-00`
   2. `apt-get upgrade -y kubelet=1.12.0-00`
   3. `kubeadm upgrade node config --kubelet-version v1.12.0`
   4. `systemctl restart kubelet`
   5. uncordon node

## Backup and restore 
### Methods

Backup candidates : 

- Resource configuration
- ETCD Cluster
- Persistent volumes

#### Resource configuration

Use declarative approach and versionning them with git ( no shit ? )

Specialized tools: velero

#### ETCD Cluster

snapshot etcd server :

- `ETCD_API=3 etcdctl snapshot save snapshot.db`
- `ETCD_API=3 etcdctl status snapshot.db`

Restore etcd server :

- `systemctl stop kube-apiserver`

- `ETCD_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd/from-backup`It will configure a new cluster and configure member as new member of new cluster. New data directories will be created ( `/var/lib/etcd/from-backup` in this example)

- Update etcd.service by updating --data-dir arg with new location ( ``/var/lib/etcd/from-backup` ` )

- ```bash 
  systemctl daemon-reload
  systemctl restart etcd
  systemctl start kube-apiserver
  ```
  

Quicknote : it's important to specify following parameters when making a snapshot: 

  ```bash
  ETCDCTL_API=3 etcdctl \
     snapshot save snapshot.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/etcd/ca.crt \
     --cert=/etc/etcd/etcd-server.crt \
     --key=/etc/etcd/etcd-server.key \
  ```

  

  ### Resources:

 - https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
  
 - https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md
  
 - https://www.youtube.com/watch?v=qRPNuT080Hk

# Security

## Authentication

###2 types of users :

- users ( not managed nativaly. can be base d on ldap, certs, files....)
- services: It is possible to create service account : `kubectl create servicesaccount sa1` ( see the dedicated part )

### auth mechanism : 

- managed by apiserver
- different authentication type:
  - password file
  - token file
  - certificates
  - services ( such ldap, ad)

## Auth with static file (depreciated since 1.19) :

- To specify user file, use  `--basic-auth-file=user-details.csv` `kube-apiserver` arg.

  ```
  <password>,<username>,<uid>,<groups>
  ```

- To specify password file, use `--token-auth-file` `kube-apiserver` arg

  ```
  <token>,<username>,<userid>,<group>
  ```

- Tokens are used as bearer token

## Auth with certs

Certs are used for 

- certs for user
- certs for server ( e.g. between `kube-apiserver` and `kube-scheduler` )
  - kube-apiserver
  - etcd-server
  - kubelet-server
  - scheduler
  - kubeproxy
  - controller manager

Every component are clients of `kube-apiserver`except for etcd : `kube-apiserver` is client of `etcd` and `kubelet`

## Cert generation

Create Root CA cert :
```bash
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj= "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```
Create admin user cert:
```bash
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj= "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt  -CAkey ca.key -out admin.crt
```

Note: in `openssl req -new -key admin.key -subj= "/CN=kube-admin/O=system:masters" -out admin.csr` the `O=system:masters` specifies the group and it is required. This group is prefixed with `system:`

Other certs will be generated as the kube-admin.

This prefix group is required for:

- kubeadmin
- kube scheduler
- kube-controller-manager
- kube-proxy

How to use certs ? For instance with kube-admin : 

```bash
curl https://kube-apiserver:6443/api/v1/pods \
  --key admin.key --cert admin.crt
  --cacert ca.crt
```

( We use `kube-config.yaml` file, obviously )
Every services require `ca.crt` 

### Server-side certificates:

Same procedure as before. We also need aditionnal for peer certs.
`cat etcd.yaml` :

```yaml
- etcd
    - --key-file=/path/to/certs/etcdserver.key
    - --cert-file=/path/to/certs/etcdserver.crt
	#[...]
	- --peer-cert-file=/path/to/certs/etcdpeer1.crt
	- --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

#### Kube API Server

everything talk to him so it has several aliases : 

- kubernetes
- kubernetes.default.svc.cluster.local
- its ip adress ( host or pod depending of installation)

```bash
openssl genrsa -out apiserver.key 2048
```

```
openssl req -new -key apiserver.key -subj \
  "/CN=kube-apiserver" -out apiserver.csr
```

To specify alternate names : 
create `openssl.cnf` :

```ini
[req]
req_extension = v3_req

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.3 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.9.1
IP.2 = 172.17.0.87
```

Now, sign-in : 

``` bash
openssl x509 -req -in apiserver.csr \
  -CA ca.crt -CAkey ca.key -out apiserver.crt
```

In configuration : 

```
ExecStart=/usr/local/bin/kube-apiserver \\
  # [...]
  --etcd-cafile=...
  --etcd-certfile=...
  --etcd-keyfile=...
  # [...]
  --kubelet-certificate-authority=....
  --kubelet-client-certificate=...
  --kubelet-client-key=...
  # [...]
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key
```

#### kubectl nodes ( client cert )

each nodes needs a cert which will be named as node itself. once created, they're used in kubelet-config.yaml

Several key/certs are required :

- client cert to communicate with web api server ( subject starts with `system:node:<node-name>`)

## Viewing certs

kubeadm will generate certs automatically 

### Inspect cert file : 

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

Locate `Subject`  and `x509v3 Subject Alternative Name` field

Make sure issuer is the right one and is not expired

### Inspect service log

```bash
journalctl -u etcd.service -l
#or
kubectl logs etcd-master
```

If pod is down, use docker ( `docker logs <container> `)

## Certificates API

To create a admin access :

1. create cert signing request  ( user create key and its certificate signing request), give it to the ops
2. Ops create a signing request 
3. approve request
4. share certs to user ( cert of user and ca cert too)

Certificate signing request is an object in k8s :

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
    - system:authenticated
  usages:
    - client auth
  signerName: kubernetes.io/kube-apiserver-client
  request:
```

```bash
kubectl get csr
kubectl certificate approve jane
```

To get the generated cert jut do:
```bash
kubectl get csr jane -o yaml
```
The cert will be shown in base64

All certificate operation are hold by controller-manager 

## Kubeconfig

Its possible to specify cert and key with `kubectl` :
```bash
kubectl get pods \
  --server my-kube-playground:6443
  --client-key admin.key
  --client-certificte admin.crt
  --certificate-authority ca.crt
```

But by default kubectl is looking for default config file `$HOME/.kube/config`

Konfigfile have 3 sections: 

- clusters
- users ()
- contexts (made to connect users with clusters)

```yaml
apiVersion: v1
kind: Config

# Specify default context
current-context: my-kube-admin@kube-playground 

clusters:
  - name: my-kube-playground
    cluster:
      certificate-authority: ca.crt
      # Alternatively : 
      certificate-authority-data:
      # Base64 encoded ca.crt
      server: https://my-kube-playground:6443
      
contexts:
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-klube-playground
      user: my-kube-admin
      namespace: foo

users:
  - name: my-kube-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
```

Once file is ready, needless to create object as usual

- Specify kubeconfig file: `kubectl config view --kubeconfig=my-custom-config`
- Change context: `kubectl config use-context prod-user@production`

## API Groups
We can access to the api server at master nodes address at port 6443 : 
`curl https://kube-master:6443/<version>/` or `curl https://kube-apiserver:6443/api/v1/pods` 
API is grouped into multiples such groups :

- `/metrics`
- `/healthz`
- `/version`
- `/api`
- `/apis`
- `/logs`

Api responsible for cluster functionnality are `/api` ( core group ) and `/apis` ( named group )

- Core group is where all core functionnality exists:
```
/api/v1/{namespaces|pods|rc|events|endpoints|nodes|bindings|PV|PVC|configmaps|secrets|services|...}
```

- Named group is going forward all new features : 
```
/apis/{apps|extensions|networking.k8s.io|storage.k8s.io|authentication.k8s.io|certificates.k8s.io}
```

API doc show in which group belong the resource 

By default, we can access to some part of apis such `/version` but to use full api, it needs authentication

```bash
curl http://localhost:6443 -k --key admin.key --cert admin.crt --cacert ca.crt
```

Alternate option : 

```bash
kubectl proxy
```

will start a local proxy on remote cluster api
carefull : kube proxy != kubectl proxy

## Authorization

For instance we don't want dev allowed to modify kubes components but they could need to watch pod states, logs, etc... Same for service account.

Different authorization :  

- Node
- ABAC (Attribute Based Authorization)
- RBAC ( Role Based Authorization )
- Webhook

### Node authorization 

User access to Kube API but also kubelet too to read information about services, endpoint, nodes, pods. Kubelet also report (write) node status, pod status, events. Theses requests are hold by **Node Authorizer**.

### ABAC ( Attribute based authorization):

It's where we associate a user, or group of user with a set of permissions. (e.g. can view, create, delete pods )
To set authorization:

```json
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
```

Unfortunately, it's difficult to manage. RBAC are better:

### RBAC ( Role Based Authorization ):

Instead of associating user or group to authorization, we associate users to role and role has privileges

### Webhook
Say we want to manage authorization to externally and not with built-in mechanisms.
For Instance Open Policy Agent 

### More modes : AllwaysAllow,  AllwaysDeny

In `kube-apiserver`, the `--authorization-mode=Always-Allow` arg will set default authorization if none is provided
```--authorization-mode=Node,RBAC,Webhook```
If there is multiple modes are provided, request will be authorized using each one in same order as specified

## RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apigroups: [""]
    resources: ["pods"]
    verbs: ["list","get","create","update","delete"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
```

Once Role is created, we must bind user to role :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: developer
    apiGroup: rbac.authorization.k8s.io
```

By default, namespace concerned will be the default one. We can specify namespace in `metadata ` part of role def file 

```bash
kubectl get/view roles
kubectl get/view rolesbindings
```

To test authorization, use :

```bash
kubectl auth can-i create deployment
kubectl auth can-i delete nodes
```

If we're admin, we can impersonate this command :

```bash
kubectl auth can-i create deployment --as peepoodo
```

or

```bash
kubectl auth can-i create deployment --as peepoodo --namespace boisjoli
```

Quick note on resource name:

It's possible to target only few resource instead of all. For instance, to allow user to access to `blue` and `orange` pod :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list","get","create","update","delete"]
    resourceNames: ["blue", "orange"]
```

## Cluster Role and Role Binding

- They're created within namespaces but 

List namespaced resources : `kubectl api-resources --namespaced=true|false`

We saw in rbac chapter how to authorize users acress namespaced resources. 

To authorize users on cluster-wide resources, we use cluster role

For example : Cluster Admin, Storage Administrator ( create pv, pvcs ..)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list","get","create","delete"]
```

Like roles, we need to create `ClusterRoleBinding` object : 

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
  - kind: User
    name: culster-admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

One more thing: 
Cluster roles can be applied on namespaced resources. When we do this, user will habe authorization across all namespaces.

## Services account in k8s

- it can be an account used by an app such prometheus to watch all metrics of applications

Create service account : 

`kubectl create serviceaccount <name>`

When service account is created, a token will be generated. Possible to watch it with `kubectl describe serviceaccount <name>`. it will show a secret name. To see it : 

`kubectl describe secret <name-of-secret>`

Each namespaces has it's own default serviceaccount which can only run basic kubernetes api basic operations

When a pod is using a service account, the token will be automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount`

In this location, there is 3 files : `token, namespace, ca.crt`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
  serviceAccount: my-service-account
  # Optionnaly: 
  automountServiceAccountToken: false 
```

The serviceAccount field cannot be modified so it will need to delete and recreate pod to update it.

## Image security

image format : `<registry>/<user>/<image>`
if image is just `nginx` docker will assume its `docker.io/nginx/nginx`
How to allow kube to use private registry : 
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.com
  --docker-password=<password>
  --docker-email=user@email.com
```
to use images comming from this registry : 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: registry.com/apps/internal-app-nginx
  imagePullSecrets:
  - name: regcred
```

## Kubernetes security

Settings can be set on pod or container level 

Pod level : 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep","3600"]

```

or container level : 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep","3600"]
    securityContext:
      runAsUser: 1000
      capabilities: # Not available at pod scope
        add: ["MAC_ADMIN"] 
```

## Network policies

By default, kube set all allow rules for internal pods traffic in cluster 

To restrict access, we create allow `NetworkPolicy`
We link `NetworkPolicy` with `labelSelector`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
    policyTypes:
      - Ingress
    ingress:
      - from:
        - podSelector:
          matchLabel:
            name: api-pod
        ports:
          - protocol: TCP
            port: 3306
```

Solutions that support network policies:

- Kube-Router
- Calico
- Romana
- Weave-net

Solutions that DO NOT support network policies: 

- Flannel 

## Developing network policies

### Ingress :
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
    policyTypes:
      - Ingress
    ingress:
      - from:
        # 2 selelectors in this section works like a "and"
        - podSelector:
            matchLabels:
              name: api-pod
          namespaceSelector: # If there is a dash here, any pod in prod namespace will be allowed !!
            matchLabels:
              name: prod
        - ipBlock:
            cidr: 192.168.5.10/32
        ports:
          - protocol: TCP
            port: 3306
```
### Egress : 
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
    policyTypes:
      - Ingress
    ingress:
      - from:
        - podSelector:
            matchLabels:
              name: api-pod
        ports:
          - protocol: TCP
            port: 3306
   egress:
     - to:
        - ipBlock:
            cidr: 192.168.5.10/32
        ports:
          - protocol: TCP
            port: 80
```
