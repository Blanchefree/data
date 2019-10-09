
包放到Tomcat /webapps下


调整Nginx.conf 监听服务和域名server_name


server {
        listen        80;
        server_name  www.e-learning.online.sh.cn;
        server_tokens off;
        access_log  logs/action-doctor.access.log ;

        location /doctor {
            proxy_pass http://127.0.0.1:8090/doctor;
                    proxy_connect_timeout 300;
        proxy_read_timeout 300;
        proxy_send_timeout 300;
        proxy_ignore_client_abort on;
        }



https://blog.csdn.net/JENREY/article/details/84205838


kubernetes + Docker
   cd /etc/yum.repos.d/
   wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   

   vim kubernetes.repo
 
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1

   yum clean all
   yum repolist
 
   cd
   wget https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   rpm --import yum-key.gpg

   wget https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   rpm --import rpm-package-key.gpg

  Docker   master、node 都需要yum源  yum-key.gpg   rpm-package-key.gpg

master安装 yum -y install docker-ce kubelet kubeadm kubectl
node  安装 yum -y install docker-ce kubelet kubeadm


  Swap需要关闭
Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。 关闭系统的Swap方法如下:

swapoff -a
修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭。 swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：

vm.swappiness=0
执行sysctl -p /etc/sysctl.d/k8s.conf使修改生效。
因为这里本次用于测试两台主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以这里修改kubelet的配置去掉这个限制。 使用kubelet的启动参数–fail-swap-on=false去掉必须关闭Swap的限制，修改/etc/sysconfig/kubelet，加入：

vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--fail-swap-on=false
 

[root@master1 ~]# echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables 
[root@master1 ~]# echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
[root@master1 ~]# echo 1 > /proc/sys/net/ipv4/ip_forward


 [root@master ~]# vim /usr/lib/systemd/system/docker.service

Environment="HTTPS_PROXY=http://www.ik8s.io:10080" 

Environment="NO_PROXY=127.0.0.0/8,10.0.0.0/16"

[root@master1 ~]# systemctl daemon-reload
[root@master1 ~]# systemctl start docker

[root@master1 ~]# docker info

......

HTTPS Proxy: http://www.ik8s.io:10080

No Proxy: 127.0.0.0/8,10.0.0.0/16

 在后边测试时候会报以下错误

[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-scheduler:v1.12.0: output: Error response from daemon: Get https://k8s.gcr.io/v2/: proxyconnect tcp: dial tcp 172.96.236.117:10080: connect: connection refused  


（2）由于不能访问谷歌镜像仓库（www.ik8s.io），故从我的阿里云仓库下载

推荐中转地址docker.io/mirrorgooglecontainers（有许多版本），但是唯独没有coredns，coredns需要从coredns/coredns:版本号 获取（这里就不再演示）


# 拼接命令pull镜像
[root@master1 ~]# kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/rsq_kubeadm#g' | sh -x

# 打标签
[root@master1 ~]# docker images |grep rsq_kubeadm |awk '{print "docker tag",$1":"$2,$1":"$2}' |sed -e 's#registry.cn-hangzhou.aliyuncs.com/rsq_kubeadm#k8s.gcr.io#2' |sh -x

# 删除旧镜像
[root@master1 ~]# docker images |grep rsq_kubeadm |awk '{print "docker rmi -f", $1":"$2}' |sh -x

# 关闭Swap选项
[root@master1 ~]# vim /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS="--fail-swap-on=false"


kubeadm初始化： 加上--token-ttl=0使得token永不过期，即此token可永久使用
[root@master1 ~]# kubeadm init --kubernetes-version=v1.13.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.13.0
[preflight] Running pre-flight checks
	[WARNING Swap]: running with swap on is not supported. Please disable swap
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.0. Latest validated version: 18.06
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
[certs] apiserver serving cert is signed for DNS names [master1.rsq.com kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.100]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master1.rsq.com localhost] and IPs [10.0.0.100 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master1.rsq.com localhost] and IPs [10.0.0.100 127.0.0.1 ::1]
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
[apiclient] All control plane components are healthy after 21.001730 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "master1.rsq.com" as an annotation
[mark-control-plane] Marking the node master1.rsq.com as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master1.rsq.com as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: qxl5b3.5b78nwu3gm1r4u6o
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.0.0.100:6443 --token awwl45.ejyr39p0cgwgo53c --discovery-token-ca-cert-hash sha256:17f6dc5827bf00b1bdc2ea5333c918fff881bd17627949043551ad1a8201798a





systemctl start docker 
docker info  

rpm -ql kubelet
清单目录/etc/bubernetes/manifests

配置文件/etc/sysconfig/kubelet

/etc/systemd/system/kubelet.service

主程序/usr/bin/kubelet


多选：
创建或修改/etc/docker/daemon.json：

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}


systemctl start kubelet.service



















master -] kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

 kubectl get pods -n kube-system命令查看，我们要看到flannel正常启动并运行的状态才算ok。
我们要使用docker image ls 要看到flannel镜像拖下来才能看见启动起来。我们只需要等待一会即可。耐心一点

