# Kubenetes 安装部署

## 节点之间使用ssh建立信任关系

生成密钥

```bash
ssh-keygen
```

将密钥复制到各个节点

```bash
ssh-copy-id admin@xxx.xxx.xxx.xxx
```

## ansible 安装

在管理节点上安装ansible

```bash
yum install ansible
```

在受控节点上安装python

```bash
yum install python
```

创建一个工作空间目录，其中创建ansible.cfg文件和hosts文件，避免与系统环境冲突
创建ansible配置文件：

```conf
[defaults]
inventory = /Users/zxs/workspaces/elk_k8s/hosts
remote_user = admin
#开启ssh长连接
ssh_args = -C -o ControlMaster=auto -o ControlPersist=5d
```

创建ansible的hosts文件：

```conf
k8s-master
k8s-node[1:2]
```

## 准备yum源

删除原有的yum源:

```bash
ansible all -a "rm -rf /etc/yum.repos.d/*" -b
ansible all -a "yum clean all" -b
```

准备centos 7源:

```bash
ansible all -a "curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo" -b
ansible all -m replace -a "path=/etc/yum.repos.d/CentOS-Base.repo regexp=\$releaserver replace=7.6.1810 backup=true"
```

准备Docker-ce的yum源文件，创建docker.repo

```conf
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```

准备Kubenetes的yum源文件,创建k8s.repo

```conf
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

发送repo文件到各个节点：

```bash
ansible all -m copy -a "src=./docker.repo dest=/etc/yum.repos.d/ state=present" -b
ansible all -m copy -a "src=./k8s.repo dest=/etc/yum.repos.d/ state=present" -b
```

## 安装Docker-ce

删除可能残留的旧版本docker，安装docker-ce

```bash
ansible all -m yum -a "name=docker* state=absent" -b
ansible all -m yum -a "name=docker-ce state=present" -b
```

## 安装Kubenetes

安装k8s组件，设置kubelet的cgroup使用systemd，并设置kublete自启动

```bash
ansible all -m yum -a "name=kubelet,kubeadm,kubectl state=present" -b
ansible all -m lineinfile -a "path=/etc/default/kubelet line=KUBELET_EXTRA_ARGS=--cgroup-driver=systemd create=yes state=present" -b
ansible all -a "systemctl daemon-reload" -b
ansible all -a "systemctl enable kubelet" -b
ansible all -m service -a "name=kubelet state=started" -b
```

## 获取Kubenetes镜像

查看kubeadm需要的镜像，在任意节点上执行：

```bash
kubeadm config images list
```

从阿里云拉取配置好的镜像，tag改为符合kubeadm需求的内容。

```bash
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/kube-controller-manager/1.14.1" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/coredns:1.3.1" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/etcd:3.3.10" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/kube-apiserver:v1.14.1" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/kube-proxy:v1.14.1" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/kube-scheduler:v1.14.1" -b &
ansible all -a "docker pull registry.cn-beijing.aliyuncs.com/coronadata/pause:3.1" -b &
registry.cn-beijing.aliyuncs.com/coronadata/pause:3.1 k8s.gcr.io/pause:3.1" -b
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/kube-apiserver:1.14.1 k8s.gcr.io/kube-apiserver:v1.14.1" -b
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/kube-scheduler:1.14.1 k8s.gcr.io/kube-scheduler:v1.14.1" -b
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/kube-proxy:1.14.1 k8s.gcr.io/kube-proxy:v1.14.1" -b
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/kube-controller-manager:1.14.1 k8s.gcr.io/kube-controller-manager:v1.14.1" -b &
ansible all -a "registry.cn-beijing.aliyuncs.com/coronadata/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10" -b &
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10" -b &
ansible all -a "docker image tag registry.cn-beijing.aliyuncs.com/coronadata/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1" -b &
```

删除多余的镜像:

```bash
ansible all -m shell -a "docker images | grep registry | awk -F' ' '{ print \$3' } '" -b
```

## 初始化Kubenetes

to be continue...