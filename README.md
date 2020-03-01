# kubernetes-cluster-openstack
Firstly, create four Ubuntu16.04 instances on Openstack using just one key pair.
Actually,I just need 3 machines for this cluster, but I create a instance with wrong version os. So, this machine is just
a springboard bound to a floating IP
## Node configuration
#
Host name|IP|Role|OS|CPU/Memory|
:--:|:--:|:--:|:--:|:--:|
k8s-master-01|10.0.0.36|Master| Ubuntu Server X86_6416.04 LTS|2VCPU 4G
k8s-node-01|10.0.0.6|Slave| Ubuntu Server X86_64 16.04 LTS|2VCPU 4G
k8s-node-02|10.0.0.62|Slave| Ubuntu Server X86_64 16.04 LTS|2VCPU 4G
## Ubuntu configuration
### Use SSH
Generate the key pair, put the private key in```~/.ssh/id_rsa```, and the public key in ```~/.ssh/id_rsa.pub```
One machine is bound to a floating IP，while other machines can use the proxy as a springboard.
#
![avatar](/Users/liuchangkundeimac/Desktop/上海交大/科研经历/k8s-openstack/ip.png)
Then,revise

```vim ~/.ssh/config```

```
Host k8s-master
    Host k8s-master
    HostName 202.120.40.8
    User ubuntu
    Port 30220
  IdentityFile ~/.ssh/id_rsa

Host k8s-node-01
        HostName 10.0.0.6
        User ubuntu
        Port 22
  ProxyCommand ssh k8s-master -W %h:%p
  IdentityFile ~/.ssh/id_rsa

Host k8s-node-02
        HostName 10.0.0.62
        User ubuntu
        Port 22
  ProxyCommand ssh k8s-master -W %h:%p
  IdentityFile ~/.ssh/id_rsa

Host k8s-master-01
        HostName 10.0.0.36
        User ubuntu
        Port 22
  ProxyCommand ssh k8s-master -W %h:%p
  IdentityFile ~/.ssh/id_rsa
  ```
  
 Then,you can just connect with each machine with ```ssh ${machine name}``` without IP and password.
 
 Even though this is convenient enough, for Mac users, 
 SSH connections can be interrupted after a long time without a response, 
 so you'll need to change some SSH Settings.
 ```vi /etc/ssh/ssh_config```
 below ```Host *```
add
```ServerAliveInterval 60``` 
###  Set root autority
```sudo passwd root```
then execute ```su``` 

input the new password you set to give root access.
## Uniformly configure the kubernetes environment for all nodes
### Configure host name
``` vim  /etc/hosts ```
add host name like ```k8s-master-01``` behind ```127.0.0.1 localhost```
as ```127.0.0.1 k82-master-01```
### Close exchange space
```swapoff -a```
### Close firewall
```ufw disable```
### Install Docker
```
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# install GPG certificate
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# install new software repository
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get -y update
```
I recommend install docker-ce 18.xx, we can just view the alternative version
```
apt-cache show docker-ce | grep -i version
sudo apt-get -y install docker-ce = ${version}
# open Docker Service
systemctl enable docker.service
```
### Configure Docker accelerator
Domestic mirror accelerator may be slow, please replace your ali cloud image accelerator, 
address such as ```https://yourself.mirror.aliyuncs.com, ```
in ali cloud console mirror container service - > image accelerator can be found in the menu

```vim /etc/docker/daemon.json``` to change cgroup boot to systemd as kubernetes recommend
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": [
    "https://yourself.mirror.aliyuncs.com/",
    "https://dockerhub.azk8s.cn",
    "https://registry.docker-cn.com"
  ],
  "storage-driver": "overlay2"
}
```
## Install kubernetes tools
```
apt-get update && apt-get install -y apt-transport-https
# install GPG certificate
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# notice：the Ubuntu16.04 name is xenial，so we use  kubernetes-xenial
cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
#install
apt-get update && apt-get install -y kubelet kubeadm kubectl
```
## Synchronization time
Set time zone and select Asian-Shanghai
```dpkg-reconfigure tzdata```
### Set the time
```
apt-get install ntpdate
# Synchronize the os time with Internet time（cn.pool.ntp.org 位于中国的公共 NTP 服务器）
ntpdate cn.pool.ntp.org
```
## Deploy Cluster(The operations in this section are done in master node)
Set a deploy file
```
mkdir /usr/local/kubernetes/cluster
cd /usr/local/kubernetes/cluster
kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
```
Then, revise the ```kubeadm.yml```
```
vim kubeadm.yml
```
```
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  # Change to  IP of master node
  advertiseAddress: 10.0.0.36
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kubernetes-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
# 国内不能访问 Google，修改为阿里云
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
# Change version same as Kubernetes tools(kubeadm,kubectl,kubelet)
kubernetesVersion: v1.17.3
networking:
  dnsDomain: cluster.local
  # 配置 POD 所在网段为我们虚拟机不重叠的网段（这里用的是 Flannel 默认网段）
  podSubnet: "10.244.0.0/16"
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```
### View desired images
```
kubeadm config images list --config kubeadm.yml
```
### Pull images
```
kubeadm config images pull --config kubeadm.yml
```
```
kubeadm init --config=kubeadm.yml --upload-certs | tee kubeadm-init.log
# the output 
Flag --experimental-upload-certs has been deprecated, use --upload-certs instead
[init] Using Kubernetes version: v1.17.3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.36]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubernetes-master localhost] and IPs [10.0.0.36 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubernetes-master localhost] and IPs [10.0.0.36 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.006217 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
4a351218cf34c15b04b08b7d916c27a625418145d8d3b281ec125fd2b2ac7ab5
[mark-control-plane] Marking the node kubernetes-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubernetes-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 10.0.0.36:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f0759e0d352c1a5de4444782b4a676460b2ea7a2876fa0accab572b8629b72c8 
```
### Configure kubectl
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
### Verify success
```
kubectl get node
# Output
NAME                 STATUS     ROLES    AGE     VERSION
kubernetes-master01   NotReady   master   4m38s   v1.17.3
```
### Configure slave nodes
```
kubeadm join 10.0.0.36:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f0759e0d352c1a5de4444782b4a676460b2ea7a2876fa0accab572b8629b72c8
# Output
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
### Verify success again
Return to master node terminal
```
kubectl get node
# 输出如下
NAME                 STATUS     ROLES    AGE   VERSION
kubernetes-master01  NotReady   master   20m   v1.17.3
kubernetes-node-01   NotReady   <none>   16s   v1.17.3
kubernetes-node-02   NotReady   <none>   6s    v1.17.3
```
notice:Note: if there is a problem with the configuration when the Node joins the Master, 
you can reset the configuration on the Node using ```kubeadm reset ```and then rejoin using the ```kubeadm join``` command. 
To delete nodes at the master node, you can use ```kubectl delete nodes <NAME>.```

### View condition of Pods
```
kubectl get pod -n kube-system -o wide
```
All but coredns should be in a running state.
### Install Calico
```
cd /usr/local/kubernetes/cluster
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
vi calico.yaml
```
Revise line 611，change ```192.168.0.0/16``` to ```10.244.0.0/16```
```
kubectl apply -f calico.yaml
# 输出如下
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
deployment.extensions/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```
### Verify success of Calico
```
watch kubectl get pods --all-namespaces
# 输出如下
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-658558ddf8-9zzjg    1/1     Running   0          90s
kube-system   calico-node-9cr5f                           1/1     Running   0          91s
kube-system   calico-node-n99mz                           1/1     Running   0          91s
kube-system   calico-node-nl67v                           1/1     Running   0          91s
kube-system   coredns-bccdc95cf-9s4bm                     1/1     Running   0          56m
kube-system   coredns-bccdc95cf-s8ggd                     1/1     Running   0          56m
kube-system   etcd-kubernetes-master                      1/1     Running   0          55m
kube-system   kube-apiserver-kubernetes-master            1/1     Running   0          55m
kube-system   kube-controller-manager-kubernetes-master   1/1     Running   0          55m
kube-system   kube-proxy-8s87d                            1/1     Running   0          36m
kube-system   kube-proxy-cbnlb                            1/1     Running   0          36m
kube-system   kube-proxy-vwhxj                            1/1     Running   0          56m
kube-system   kube-scheduler-kubernetes-master            1/1     Running   0          55m
```
```
kubectl get node
# 输出如下
NAME                 STATUS   ROLES    AGE   VERSION
kubernetes-master    Ready    master   57m   v1.17.3
kubernetes-node-01   Ready    <none>   37m   v1.17.3
kubernetes-node-02   Ready    <none>   36m   v1.17.3
```
Viewing the node in state Ready indicates that the installation was successful!
