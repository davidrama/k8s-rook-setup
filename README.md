## Hardware setup

We will use 7 bare-metal hosts boostrapped from MAAS on Ubuntu 16.04LTS\
**All nodes have their storage HDD mounted on /opt partition**

### Master nodes:

```
master1 - 172.17.6.28
master2 - 172.17.6.29
master3 - 172.17.6.30
```

### Workers nodes:

```
agent1 - 172.17.6.25
agent2 - 172.17.6.27
agent3 - 172.17.6.26
agent4 - 172.17.6.24
```

note: If hosts are  deployed via MAAS, desactivate the Proxy in Settings/Network services section as this leads to apt repo errors or add the kubespray requested repositories int the Settings/Package repositories section.

Don't forget to make your system up to date on each of your nodes:

```shell
sudo apt update && sudo apt -y upgrade
sudo reboot (might not be needed)
```

## Kubespray setup

### Clone the kubespray project:

```shell
git clone https://github.com/kubernetes-mincubator/kubespray.git
```

Switch to the kubespray directory

### Install dependencies from  requirements.txt

```shell
sudo pip install -r requirements.txt
```

### Copy inventory/sample as inventory/mycluster (can change the name)

```shell
cp -rfp inventory/sample inventory/mycluster
```

### Update Ansible inventory file with inventory builder:

```shell
declare -a IPS=(172.17.6.24 172.17.6.25 172.17.6.26 172.17.6.27 172.17.6.28 172.17.6.29 172.17.6.30)

CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

### Edit kubespray/inventory/mycluster/host.ini file to set your hosts roles (example):

```shell
[all]
node1 	 ansible_host=172.17.6.24 ip=172.17.6.24
node2 	 ansible_host=172.17.6.25 ip=172.17.6.25
node3 	 ansible_host=172.17.6.26 ip=172.17.6.26
node4 	 ansible_host=172.17.6.27 ip=172.17.6.27
node5 	 ansible_host=172.17.6.28 ip=172.17.6.28
node6 	 ansible_host=172.17.6.29 ip=172.17.6.29
node7 	 ansible_host=172.17.6.30 ip=172.17.6.30

[kube-master]
node5
node6
node7

[kube-node]
node1
node2
node3
node4
node5
node6
node7

[etcd]
node5
node6
node7

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]

[vault]
node5
node6
node7
```

### Add the following at the top of the cluster.yml to remove the swap usage on nodes 

(this might be ubuntu specific, check your fstab for the right Regexp)

```yaml
---
- hosts: all
  gather_facts: False

  tasks:
  - name: remove the swap
    raw: /sbin/swapoff -a && (sed -i -e '/swap/d' /etc/fstab)
```

To add bionic docker support (as of 07/2018) edit kubespray/roles/docker/vars/ubuntu.yml

Add to the docker_versioned_pkg:** **section:

```yaml
 '17.12-bionic': docker.io=17.12.1-0ubuntu1
```

Edit kubespray/roles/docker/defaults/main.yml and update the docker version on the two top lines:

```yaml
docker_version: '17.12-bionic'
docker_selinux_version: '17.12-bionic'
```

### Review and change parameters under  kubespray/inventory/mycluster/group_vars:

```shell
cat kubespray/inventory/mycluster/group_vars/all.yml
cat kubespray/inventory/mycluster/group_vars/k8s-cluster.yml
```

## Rook integration (0.7.1):

set the kublet options for flexvolume needed for rook integration as stated here: [Flex Volume Configuration](https://rook.io/docs/rook/v0.7/flexvolume.html)

Edit kubespray/inventory/mycluster/group_vars/k8s-cluster.yml  and put at the bottom the --volume-plugin-dir=/var/lib/kubelet/volume-plugins  option (create the kubelet_custom_flags section if you don't have one):

```yaml
kubelet_custom_flags: 
  - "--eviction-hard=memory.available<800Mi" 
  - "--eviction-soft-grace-period=memory.available=10m,nodefs.available=10m,imagefs.available=10m" 
  - "--eviction-soft=memory.available<300Mi" 
  - "--eviction-max-pod-grace-period=-1" 
  - "--eviction-hard=memory.available<300Mi,nodefs.available<1Gi" 
  - "--eviction-pressure-transition-period=10m" 
  - "--volume-plugin-dir=/var/lib/kubelet/volume-plugins” 
```

This will add the following kubelet parameter for you on all your nodes in the /etc/kubernetes/kubelet.env file:

```shell
KUBELET_VOLUME_PLUGIN="--volume-plugin-dir=/var/lib/kubelet/volume-plugins"
```

in k8s-cluster.yml file don't forget to set this option to true if you want to get your kubectl config file generated in kubespray/inventory/mycluster/artifacts/

```yaml
kubeconfig_localhost: true
```

## Kubernetes Deployment

### Launch the playbook for cluster install (here as sudo user ubuntu):

```shell
ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml -b --fork=50 --become --become-user=root -e "ansible_ssh_user=ubuntu"
```

Hopefully you should get this output (take a coffee) you might sometimes get errors due to timeouts, relaunching the playbook might be the solution) :

```shell
PLAY RECAP ***************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0
node1                      : ok=306  changed=7    unreachable=0    failed=0
node2                      : ok=291  changed=5    unreachable=0    failed=0
node3                      : ok=291  changed=5    unreachable=0    failed=0
node4                      : ok=291  changed=5    unreachable=0    failed=0
node5                      : ok=750  changed=38   unreachable=0    failed=0
node6                      : ok=713  changed=19   unreachable=0    failed=0
node7                      : ok=713  changed=19   unreachable=0    failed=0
```

### Install the kubectl config file in the .kube directory:

 `cp inventory/mycluster/artifacts/admin.conf \~/.kube/config   `

Your cluster should be up and running:

```shell
k8s-rook-setup[rama]->kubectl get nodes -o wide
NAME      STATUS    ROLES         AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION         CONTAINER-RUNTIME
node1     Ready     node          10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node2     Ready     node          10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node3     Ready     node          10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node4     Ready     node          10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node5     Ready     master,node   10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node6     Ready     master,node   10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2
node7     Ready     master,node   10h       v1.10.4   <none>        Ubuntu 16.04.4 LTS   4.4.0-130-lowlatency   docker://17.3.2


k8s-rook-setup[rama]->kubectl cluster-info
Kubernetes master is running at https://172.17.6.28:6443
KubeDNS is running at https://172.17.6.28:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://172.17.6.28:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```

If this command time out first thing to check is the networking part here calico. For this ssh on one of your node and check that calico is working correctly (ie that the peering mesh is ok):

```shell
sudo calicoctl node status


IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.17.6.24  | node-to-node mesh | up    | 22:52:32 | Established |
| 172.17.6.25  | node-to-node mesh | up    | 22:52:33 | Established |
| 172.17.6.26  | node-to-node mesh | up    | 22:52:32 | Established |
| 172.17.6.27  | node-to-node mesh | up    | 22:52:31 | Established |
| 172.17.6.28  | node-to-node mesh | up    | 22:52:33 | Established |
| 172.17.6.29  | node-to-node mesh | up    | 22:52:31 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

if the BGP status is empty, you have an issue (got this using ubuntu 18.04). You might need to check your iptables/fw host setups.

### Check that you have access to the Dasboard using the url (cluster-info):

```shell
https://172.17.6.203:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```

### Get your credential for user admin (or the user you defined in the k8s-cluster.yml file in the kube_users section:

```
cat kubespray/inventory/mycluster/credentials/kube_user.creds
```

## or get the authentication Token  using the dashboard pod Token:

```shell
kubectl describe secret -n kube-system $(kubectl get secrets -n kube-system | grep dashboard-token | cut -d ' ' -f1) | grep -E '^token' | cut -f2 -d':' | tr -d '\t'
```

If you get authentication issue to acces some menus (before securing your infra) apply this ClusterRoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

## BACKUP all the the kubespray/inventory/mycluster directory as it'll be needed for day to day  kubespray operations on your kubernetes cluster (upgrade/node add/node removal etc..)

# Rook storage system setup

### Check that helm is ready to run (assuming you installed it already):

```shell
kubespray[rama]->helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

### Add the rook helm repo (alpha branch seems to be the one to use atm - 072018):

```shell
helm repo add rook-alpha https://charts.rook.io/alpha
"rook-alpha" has been added to your repositories
```

### Check that the manifest is available:

```shell
helm search rook
NAME           	CHART VERSION	APP VERSION	DESCRIPTION
rook-alpha/rook	v0.7.1       	           	File, Block, and Object Storage Services for yo...
```

### Install rook operator (here in rook-system namespace) with flexvolume plugin set:

```shell
helm install --namespace rook-system --name rook rook-alpha/rook --version=v0.7.1 --set agent.flexVolumeDirPath=/var/lib/kubelet/volume-plugins
```

### Rook operator should be running:

```shell
kubectl --namespace rook-system get pods -l "app=rook-operator"
NAME                             READY     STATUS    RESTARTS   AGE
rook-operator-7575788469-jxt7f   1/1       Running   0          2m
```

### Get the manifests needed for Rook setup (0.7.1):

```
mkdir rook-configs
cd rook-configs
```

Cluster file:

```
wget https://raw.githubusercontent.com/rook/rook/release-0.7/cluster/examples/kubernetes/rook-cluster.yaml
```

ToolBox file:

```
wget https://raw.githubusercontent.com/rook/rook/release-0.7/cluster/examples/kubernetes/rook-tools.yaml
```

StorageClass file:

```
wget https://raw.githubusercontent.com/rook/rook/release-0.7/cluster/examples/kubernetes/rook-storageclass.yaml
```

Filesystem file:

```
wget https://raw.githubusercontent.com/rook/rook/release-0.7/cluster/examples/kubernetes/rook-filesystem.yaml
```

### Edit all these files to suite your needs 

Note: Currently there is a bug that you can't use rbd with an erasure coded pool in Rook. Change the Pool to use replicated instead (as of 07/2018)

Changed the dataDirHostPath to  /opt/rook in rook-cluster.yml file as my persistent storages are mounted there

rook-cluster.yml

```
apiVersion: v1
kind: Namespace
metadata:
  name: rook
---
apiVersion: rook.io/v1alpha1
kind: Cluster
metadata:
  name: rook
  namespace: rook
spec:
  backend: ceph
  dataDirHostPath: /opt/rook
  hostNetwork: false
  monCount: 3
  resources:
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter:
    metadataDevice:
    location:
    storeConfig:
      storeType: bluestore
```

rook-storageclass.yml

```
apiVersion: rook.io/v1alpha1
kind: Pool
metadata:
  name: replicapool
  namespace: rook
spec:
  replicated:
    size: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-block
provisioner: rook.io/block
parameters:
  pool: replicapool
  fstype: ext4
```

### Applying manifests

```
kubectl apply -f rook-cluster.yaml
kubectl apply -f rook-storageclass.yaml
kubectl apply -f rook-filesystem.yaml
kubectl apply -f rook-tools.yaml
```

### Check the status of the ceph cluster

```
kubectl  -n rook exec rook-tools -- ceph status

  cluster:
    id:     7ad25335-2ba2-4cb8-8b7c-e9ddd9fb3760
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum rook-ceph-mon0,rook-ceph-mon1,rook-ceph-mon2
    mgr: rook-ceph-mgr0(active)
    mds: rookfs-1/1/1 up  {0=mkfn7x=up:active}, 1 up:standby-replay
    osd: 7 osds: 7 up, 7 in

  data:
    pools:   3 pools, 300 pgs
    objects: 21 objects, 2246 bytes
    usage:   147 GB used, 3292 GB / 3440 GB avail
    pgs:     300 active+clean

  io:
    client:   852 B/s rd, 1 op/s rd, 0 op/s wr 
    
    rook[rama]->kubectl  -n rook exec rook-tools -- ceph osd status
+----+---------------------+-------+-------+--------+---------+--------+---------+-----------+
| id |         host        |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+---------------------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | rook-ceph-osd-sqcl9 | 21.0G |  218G |    0   |     0   |    0   |     0   | exists,up |
| 1  | rook-ceph-osd-8lzw2 | 21.0G |  218G |    0   |     0   |    0   |     0   | exists,up |
| 2  | rook-ceph-osd-m9qd5 | 21.0G |  218G |    0   |     0   |    0   |     0   | exists,up |
| 3  | rook-ceph-osd-xs8r7 | 21.0G |  879G |    0   |     0   |    2   |   106   | exists,up |
| 4  | rook-ceph-osd-rrbhg | 21.0G |  879G |    0   |     0   |    0   |     0   | exists,up |
| 5  | rook-ceph-osd-4fz96 | 21.0G |  659G |    0   |     0   |    0   |     0   | exists,up |
| 6  | rook-ceph-osd-jt64x | 21.0G |  218G |    0   |     0   |    0   |     0   | exists,up |
+----+---------------------+-------+-------+--------+---------+--------+---------+-----------+
```

### Check that the storageclass exists

```
rook[rama]->kubectl get storageclass -n rook
NAME         PROVISIONER     AGE
rook-block   rook.io/block   8m
```

### Set it to default class (if you want)

```
kubectl patch storageclass rook-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Check that it is now default

```
rook[rama]->kubectl get storageclass
NAME                   PROVISIONER     AGE
rook-block (default)   rook.io/block   33m
```

# Testing the setup

We will the setup by simply deploying the mysql chart from helm repo

```
rook[rama]->kubectl get pvc --all-namespaces
No resources found.

helm install stable/mysql --name=mysql --namespace=test

rook[rama]->kubectl get all -n test
NAME                         READY     STATUS    RESTARTS   AGE
pod/mysql-58d5cb6fd4-sfzc7   1/1       Running   0          45s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/mysql   ClusterIP   10.233.28.31   <none>        3306/TCP   45s

NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql   1         1         1            1           45s

NAME                               DESIRED   CURRENT   READY     AGE
replicaset.apps/mysql-58d5cb6fd4   1         1         1         45s
```

### Chek if the volume is bound

```
rook[rama]->kubectl get pvc -n test
NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql     Bound     pvc-337ec16d-8b58-11e8-abaf-e41d2dae3526   8Gi        RWO            rook-block     1m
```

if this is still in pending please check the rook-operator 's logs:

```
kubectl logs -f $(kubectl get pods -n rook-system | grep operator | cut -d ' ' -f1) -n rook-system
```

### Clean the mysql installation

```
helm delete --purge mysql
```

## Special thanks to Alexander Trost and especially Majestic from the rook.io slack channel for their help

# Adding Ingress loadbalancing using MetalLB and Traefik

Difficulty when using Baremetal nodes is that there's no autmated way to provide External_IP to exposed services.

Using only and ingress Service il bind services ports to NodePorts and then you'll need to map the NodePorts IP to your lb FQDN. 

Meaning each service will be avalaible on NODEIP:port then handled by the LB.  

As I do not run an external Load Balancer I will use MetalLB ([MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/installation/)) in ARP mode for now

Idea behind It'll provide/use some IP's from my nodes Network (172.17.6.0/24) and bind services on them and route requests to Traefik(s) instances.

For this I'll reserve** 172.17.6.128/172.17.6.130 **IP's.

## MetalLB setting

### First install the manifest

```shell
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.1/manifests/metallb.yaml
```

### Check that the controller and speakers are up

```shell
kubectl get pods -n metallb-system
NAME                          READY     STATUS    RESTARTS   AGE
controller-5f69bbf78c-lg76z   1/1       Running   0          33s
speaker-4pdg5                 1/1       Running   0          33s
speaker-c7rmj                 1/1       Running   0          33s
speaker-fxpbv                 1/1       Running   0          33s
speaker-jltcq                 1/1       Running   0          33s
speaker-sf747                 1/1       Running   0          33s
speaker-tcc9z                 1/1       Running   0          33s
speaker-v6h2g                 1/1       Running   0          33s
```

### Now define our Layer2 setup metallb-configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.17.6.128-172.17.6.130 
```

Now MetalLB should be abble to provide External IP to exposed services taken from the pool 172.17.6.128-172.17.6.130

Now that we have a "public" IP provider running, let's set up the ingress part using Traefik.

## Traefik Setup

Follwing this guide: [Kubernetes - Træfik](https://docs.traefik.io/user-guide/kubernetes/)

As recommended we will use the DaemonSet

```
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml
```

You can edit/review the yaml file if you need it of course.

### Check the pods are ok (1 ingress per node):

```shell
rook[rama]->kubectl get pods  -n kube-system | grep traefik
traefik-ingress-controller-27b6n       1/1       Running   0          3h
traefik-ingress-controller-f98r7       1/1       Running   0          3h
traefik-ingress-controller-gj79r       1/1       Running   0          3h
traefik-ingress-controller-jwnxz       1/1       Running   0          3h
traefik-ingress-controller-pzrwr       1/1       Running   0          3h
traefik-ingress-controller-vw4tk       1/1       Running   0          3h
traefik-ingress-controller-xq9cz       1/1       Running   0          3h
```

### Traefik Service should be running now

```
kubectl get svc -n kube-system
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
kube-dns                  ClusterIP   10.233.0.3      <none>        53/UDP,53/TCP     4d
kubelet                   ClusterIP   None            <none>        10250/TCP         4d
kubernetes-dashboard      ClusterIP   10.233.4.154    <none>        443/TCP           4d
metrics-server            ClusterIP   10.233.63.112   <none>        443/TCP           4d
tiller-deploy             ClusterIP   10.233.60.61    <none>        44134/TCP         4d
traefik-ingress-service   ClusterIP   10.233.30.11    <none>        80/TCP,8080/TCP   5s
```

### Provide the service with MetalLB provided IP

Edit the traefik service and change type from **ClusterIP** to **LoadBalancer** and fix the IP yourself or let metalLB take one for you:

```
rook[rama]->kubectl edit svc -n kube-system traefik-ingress-service

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"traefik-ingress-service","namespace":"kube-system"},"spec":{"ports":[{"name":"web","port":80,"protocol":"TCP"},{"name":"admin","port":8080,"protocol":"TCP"}],"selector":{"k8s-app":"traefik-ingress-lb"}}}
  creationTimestamp: 2018-07-23T20:54:23Z
  name: traefik-ingress-service
  namespace: kube-system
  resourceVersion: "860076"
  selfLink: /api/v1/namespaces/kube-system/services/traefik-ingress-service
  uid: 943859f7-8eba-11e8-aaba-e41d2dae31c6
spec:
  clusterIP: 10.233.30.11
  ports:
  - name: web
    port: 80
    protocol: TCP
    targetPort: 80
  - name: admin
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    k8s-app: traefik-ingress-lb
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
```

### And now the ingress controller should be abble to answer "public" requests (here on 172.17.6.128)

```
rook[rama]->kubectl get svc -n kube-system
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                       AGE
kube-dns                  ClusterIP      10.233.0.3      <none>         53/UDP,53/TCP                 4d
kubelet                   ClusterIP      None            <none>         10250/TCP                     4d
kubernetes-dashboard      ClusterIP      10.233.4.154    <none>         443/TCP                       4d
metrics-server            ClusterIP      10.233.63.112   <none>         443/TCP                       4d
tiller-deploy             ClusterIP      10.233.60.61    <none>         44134/TCP                     4d
traefik-ingress-service   LoadBalancer   10.233.30.11    172.17.6.128   80:31389/TCP,8080:32471/TCP   3m
```

### Let's try the http port and the Dasboard port (https will come next)

```
rook[rama]->curl http://172.17.6.128
404 page not found

rook[rama]->curl http://172.17.6.128:8080
<a href="/dashboard/">Found</a>.

rook[rama]->nslookup traefik.ael.li
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	traefik.ael.li
Address: 172.17.6.128

rook[rama]->curl http://traefik.ael.li
404 page not found

rook[rama]->curl http://traefik.ael.li:8080
<a href="/dashboard/">Found</a>.
```

### Let's expose the dashboard directely on [http://dashboard-traefik.ael.li](http://dashboard-traefik.ael.li)

First create a CNAME record on traefik.ael.li A record in your DNS (wait for propagation if needed)

```
rook[rama]->nslookup dashboard-traefik.ael.li
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
dashboard-traefik.ael.li	canonical name = traefik.ael.li.
Name:	traefik.ael.li
Address: 172.17.6.128
```

### Create and apply the UI service (change the fqdn to suite your setup):

```
traefik[rama]->cat dashboard-traefik.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-dashboard
  namespace: kube-system
  annotations:
   kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: dashboard-traefik.ael.li
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

### Check the ingress status

```shell
traefik[rama]->kubectl get ing -n kube-system
NAME                HOSTS                      ADDRESS   PORTS     AGE
traefik-dashboard   dashboard-traefik.ael.li             80        10m
```

### Now the dashboard should be available on [http://dashboard-traefik.ael.li/](http://dashboard-traefik.ael.li/) and loadbalanced accross your nodes:

```
traefik[rama]->curl http://dashboard-traefik.ael.li/
<a href="/dashboard/">Found</a>.
```

### Beware that this might be publicly opened to the public, check the treafik documentation to setup a protected access to the ui if wanted.

## Adding SSL support with LetsEncrypt

To come
