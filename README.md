
目录* [k8s搭建(1\.28\.2版本)](https://github.com)
	+ [1\. 安装containerd](https://github.com)
		- [1\.1 下载tar包](https://github.com)
		- [1\.2 编写服务单元文件](https://github.com)
	+ [2\. 安装runc](https://github.com)
	+ [3\. 安装cni插件](https://github.com)
		- [3\.1 下载文件](https://github.com)
		- [3\.2 设置crictl运行端点](https://github.com)
	+ [4\. 配置containerd](https://github.com)
	+ [5\. 主机配置](https://github.com)
		- [5\.1 编辑hosts文件(可选)](https://github.com):[veee加速器](https://liuyunzhuge.com)
		- [5\.2 开启流量转发](https://github.com)
		- [5\.3 关闭防火墙以及selinux](https://github.com)
		- [5\.4 关闭swap](https://github.com)
	+ [6\. 搭建k8s](https://github.com)
		- [6\.1 配置yum源](https://github.com)
		- [6\.2 安装工具](https://github.com)
		- [6\.3 初始化](https://github.com)
	+ [7\. 网络插件](https://github.com)
		- [7\.1 安装calico](https://github.com)
		- [7\.2 配置镜像加速器地址](https://github.com)

# k8s搭建(1\.28\.2版本)


不知道从什么时候开始，openEuler已经开始支持使用containerd的k8s集群了，之前我学习的时候最高都只能支持到1\.23，所以这里再来写一篇关于部署运行时为containerd的集群



> 为什么要单独写关于openEuler的部署方式？


因为使用centos的部署方式在openEuler上部署的时候会有一些差异，而这些差异的地方就会导致无法继续往下进行，所以我单独写一篇博客来避开这些坑点


## 1\. 安装containerd


你可能想问，一个containerd有什么不会安装的，直接使用yum不就可以安装好了吗？是的，你在其他操作系统上确实可以这么干，但是在openEuler上这么干不会报错，因为yum仓库里面确实有containerd的rpm包，你确实可以装上，但是那个containerd版本太低。无法正常的使用。所以需要下载tar包来安装


### 1\.1 下载tar包



```


|  | # 确保没有使用官方仓库的containerd |
| --- | --- |
|  | [root@master ~]# yum remove containerd -y |
|  | [root@master ~]# wget https://github.com/containerd/containerd/releases/download/v1.7.16/containerd-1.7.16-linux-amd64.tar.gz |
|  | [root@master ~]# tar -zxvf containerd-1.7.16-linux-amd64.tar.gz |
|  | [root@master ~]# mv bin/* /usr/local/bin/ |


```

### 1\.2 编写服务单元文件



```


|  | [root@master ~]# vim /usr/lib/systemd/system/containerd.service |
| --- | --- |
|  |  |
|  | [Unit] |
|  | Description=containerd container runtime |
|  | Documentation=https://containerd.io |
|  | After=network.target local-fs.target |
|  |  |
|  | [Service] |
|  | ExecStartPre=-/sbin/modprobe overlay |
|  | ExecStart=/usr/local/bin/containerd |
|  |  |
|  | Type=notify |
|  | Delegate=yes |
|  | KillMode=process |
|  | Restart=always |
|  | RestartSec=5 |
|  |  |
|  | # Having non-zero Limit*s causes performance problems due to accounting overhead |
|  | # in the kernel. We recommend using cgroups to do container-local accounting. |
|  | LimitNPROC=infinity |
|  | LimitCORE=infinity |
|  |  |
|  | # Comment TasksMax if your systemd version does not supports it. |
|  | # Only systemd 226 and above support this version. |
|  | TasksMax=infinity |
|  | OOMScoreAdjust=-999 |
|  |  |
|  | [Install] |
|  | WantedBy=multi-user.target |


```

然后给containerd设计开机自启



```


|  | [root@master ~]# systemctl daemon-reload |
| --- | --- |
|  | [root@master ~]# systemctl enable --now containerd |


```

## 2\. 安装runc


这个也是一样的，不能使用yum安装的版本(至少面前不可以\-\-文章写于2024\-11\-9\)



```


|  | [root@master ~]# yum remove runc -y |
| --- | --- |
|  | [root@master ~]# wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 |
|  | [root@master ~]# install -m 755 runc.amd64 /usr/local/sbin/runc |
|  |  |


```

## 3\. 安装cni插件


### 3\.1 下载文件



```


|  | [root@master ~]# wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz |
| --- | --- |
|  | [root@master ~]# mkdir -p /opt/cni/bin |
|  | [root@master ~]# tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.1.tgz |


```

### 3\.2 设置crictl运行端点



```


|  | cat <<EOF > /etc/crictl.yaml |
| --- | --- |
|  | runtime-endpoint: unix:///run/containerd/containerd.sock |
|  | image-endpoint: unix:///run/containerd/containerd.sock |
|  | timeout: 5 |
|  | debug: false |
|  | EOF |


```

## 4\. 配置containerd



```


|  | [root@master ~]# containerd config default > /etc/containerd/config.toml |
| --- | --- |
|  | # 将cgroup打开 |
|  | [root@master ~]# vim /etc/containerd/config/toml |
|  | # 找到这一行配置，将false改为true |
|  | SystemdCgroup = true |
|  | # 修改sandbox镜像地址 |
|  | sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9" |


```

重启containerd



```


|  | [root@master ~]# systemctl restart containerd |
| --- | --- |


```

## 5\. 主机配置


### 5\.1 编辑hosts文件(可选)


将IP与主机名写入到/etc/hosts文件内，我这里就不做了。不做没有任何影响


### 5\.2 开启流量转发



```


|  | [root@master ~]# modprobe bridge |
| --- | --- |
|  | [root@master ~]# modprobe br_netfilter |
|  | [root@master ~]# vim /etc/sysctl.conf |
|  | net.bridge.bridge-nf-call-ip6tables = 1 |
|  | net.bridge.bridge-nf-call-iptables = 1 |
|  | net.ipv4.ip_forward = 1 |
|  | [root@master ~]# sysctl -p |


```

### 5\.3 关闭防火墙以及selinux



```


|  | [root@master ~]# sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config |
| --- | --- |
|  | [root@master ~]# systemctl disable --now firewalld |


```

### 5\.4 关闭swap


如果配置了swap，请关闭他



```


|  | [root@master ~]# swapoff -a |
| --- | --- |


```

然后进入到`/etc/fstab`里面注释掉swap的那一行内容


## 6\. 搭建k8s


到这里就开始搭建k8s了


### 6\.1 配置yum源



```


|  | [root@master ~]#  cat < /etc/yum.repos.d/kubernetes.repo |
| --- | --- |
|  | [kubernetes] |
|  | name=Kubernetes |
|  | baseurl=https://mirrors.huaweicloud.com/kubernetes/yum/repos/kubernetes-el7-$basearch |
|  | enabled=1 |
|  | gpgcheck=1 |
|  | repo_gpgcheck=0 |
|  | gpgkey=https://mirrors.huaweicloud.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.huaweicloud.com/kubernetes/yum/doc/rpm-package-key.gpg |
|  | EOF |


```

### 6\.2 安装工具



```


|  | [root@master ~]# yum install kubectl kubeadm kubelet -y |
| --- | --- |
|  | [root@master ~]# systemctl enable kubelet |


```

### 6\.3 初始化



```


|  | [root@master ~]#  kubeadm init --kubernetes-version=v1.28.2 --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers |
| --- | --- |


```

* 这里的kubernetes\-version后面的值修改为你自己的kubeadm的版本



```


|  | Your Kubernetes control-plane has initialized successfully! |
| --- | --- |
|  |  |
|  | To start using your cluster, you need to run the following as a regular user: |
|  |  |
|  | mkdir -p $HOME/.kube |
|  | sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config |
|  | sudo chown $(id -u):$(id -g) $HOME/.kube/config |
|  |  |
|  | Alternatively, if you are the root user, you can run: |
|  |  |
|  | export KUBECONFIG=/etc/kubernetes/admin.conf |
|  |  |
|  | You should now deploy a pod network to the cluster. |
|  | Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at: |
|  | https://kubernetes.io/docs/concepts/cluster-administration/addons/ |
|  |  |
|  | Then you can join any number of worker nodes by running the following on each as root: |
|  |  |
|  | kubeadm join 192.168.200.200:6443 --token alefrt.vuiz4k7424ljhh2i \ |
|  | --discovery-token-ca-cert-hash sha256:1c0943c98d9aeaba843bd683d60ab66a3b025d65726932fa19995f067d62d436 |


```

看到这一段信息就是初始化成功了，然后我们根据提示来创建目录



```


|  | [root@master ~]# mkdir -p $HOME/.kube |
| --- | --- |
|  | [root@master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config |
|  | [root@master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config |


```

，如果有其他节点需要加入集群那么就执行



```


|  | [root@master ~]# kubeadm join 192.168.200.200:6443 --token alefrt.vuiz4k7424ljhh2i \ |
| --- | --- |
|  | --discovery-token-ca-cert-hash sha256:1c0943c98d9aeaba843bd683d60ab66a3b025d65726932fa19995f067d62d436 |


```

然后我们可以查看节点状态



```


|  | [root@master ~]# kubectl get nodes |
| --- | --- |
|  | NAME     STATUS     ROLES           AGE   VERSION |
|  | master   NotReady   control-plane   26s   v1.28.2 |


```

接下来我们安装网络插件calico，让他的状态变为Ready


## 7\. 网络插件


### 7\.1 安装calico



```


|  | [root@master ~]#  wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml |
| --- | --- |
|  | [root@master ~]# kubectl create -f tigera-operator.yaml |


```

接下来我们来处理第二个文件



```


|  | [root@master ~]# wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml |
| --- | --- |
|  | [root@master ~]# vim custom-resources.yaml |
|  |  |
|  | # 将里面的cidr改为初始化集群使用的地址段 |
|  | cidr: 10.244.0.0/16 |


```

### 7\.2 配置镜像加速器地址


如果不配置镜像加速器地址的话。镜像是拉取不到的。



```


|  | [root@master ~]# vim /etc/containerd/config.toml |
| --- | --- |
|  | # 需要找到这一行，并添加2行 |
|  | [plugins."io.containerd.grpc.v1.cri".registry.mirrors] |
|  | [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"] |
|  | endpoint = ["镜像加速器地址1","镜像加速器地址2"] |


```

可以百度搜一下哪些镜像加速器地址还可以使用，然后替换掉里面的文字


重启containerd



```


|  | [root@master ~]# systemctl restart containerd |
| --- | --- |


```

然后等到他把所有的镜像拉取完之后集群就正常了


最终就是这样的



```


|  | [root@master ~]# kubectl get nodes |
| --- | --- |
|  | NAME     STATUS   ROLES           AGE   VERSION |
|  | master   Ready    control-plane   15m   v1.28.2 |


```

