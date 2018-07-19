## Hardware setup

We will use 7 bare-metal hosts boostrapped from MAAS on Ubuntu 16.04LTS

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

note: If host deployed via MAAS, desactivate the Proxy in Settings/Network services section as this leads to apt repo errors or add the kubespray requested repositories int the Settings/Package repositories section.

Don't forget to make your system up to date on each of your nodes:\
`sudo apt update && sudo apt -y upgrade\
sudo reboot (might not be needed)`

## Kubespray setup

### Clone the kubespray project:

`**git clone **`[`**https://github.com/kubernetes-mincubator/kubespray.git**`](https://github.com/kubernetes-mincubator/kubespray.git)

Switch to the kubespray directory

### Install dependencies from  requirements.txt

```
sudo pip install -r requirements.txt
```

### Copy inventory/sample as inventory/mycluster (can change the name)

`cp -rfp inventory/sample inventory/mycluster`\

### Update Ansible inventory file with inventory builder:

`declare -a IPS=(172.17.6.24 172.17.6.25 172.17.6.26 172.17.6.27 172.17.6.28 172.17.6.29 172.17.6.30)
CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS\[@\]}`

### Edit kubespray/inventory/mycluster/host.ini file to set your hosts roles (example):

```
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

\(this might be ubuntu specific, check your fstab for the right Regexp)

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

Edit **kubespray/roles/docker/defaults/main.yml **and update the docker version on the two top lines:

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

Edit kubespray/inventory/mycluster/group_vars/k8s-cluster.yml  and put at the bottom the --volume-plugin-dir=/var/lib/kubelet/volumeplugins  option (create the kubelet_custom_flags section if you don't have one):

```yaml
kubelet_custom_flags: 
  - "--eviction-hard=memory.available<800Mi" 
  - "--eviction-soft-grace-period=memory.available=10m,nodefs.available=10m,imagefs.available=10m" 
  - "--eviction-soft=memory.available<300Mi" 
  - "--eviction-max-pod-grace-period=-1" 
  - "--eviction-hard=memory.available<300Mi,nodefs.available<1Gi" 
  - "--eviction-pressure-transition-period=10m" 
  - "--volume-plugin-dir=/var/lib/kubelet/volumeplugins” 
```

This will add the following kubelet parameter for you on all your nodes:

```shell
KUBELET_VOLUME_PLUGIN="--volume-plugin-dir=/var/lib/kubelet/volume-plugins"
```

in k8s-cluster.yml file don't forget to set option if you want to get your kubectl config file generated in kubespray/inventory/mycluster/artifacts/

```yaml
kubeconfig_localhost: true
```

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
kubespray[rama]->kubectl get nodes -o wide
NAME      STATUS    ROLES         AGE       VERSION   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION         CONTAINER-RUNTIME
node1     Ready     node          1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node2     Ready     node          1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node3     Ready     node          1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node4     Ready     node          1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node5     Ready     master,node   1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node6     Ready     master,node   1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce
node7     Ready     master,node   1h        v1.10.4   <none>        Ubuntu 18.04 LTS   4.15.0-23-lowlatency   docker://17.12.1-ce

kubespray[rama]->kubectl cluster-info
Kubernetes master is running at https://172.17.6.203:6443
KubeDNS is running at https://172.17.6.203:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://172.17.6.203:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
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

### Check that you have access to the Dasboard using the url:

```shell
kubernetes-dashboard is running at https://172.17.6.203:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```

You can use Token authentication using the dashboard pod Token. To get it use:

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

# Rook storage system setup

### Check that helm is ready to run (assuming you installed helm already):

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