## 使用kuberspay无坑安装生产级Kubernetes集群

`kuberspay`是`kargo`更名后的名称，我在前面写过一篇[使用kargo快速自动化搭建kubernetes集群](http://www.wisely.top/2017/05/16/kargo-ansible-kubernetes/)(各节点的准备信息也请参考该文)，上篇文章的部署方式的缺陷还是需要科学上网，所以还是比较麻烦的。我又在另外一篇文章[无坑畅玩minikube(利用阿里云镜像编译minikube)](http://www.wisely.top/2017/06/27/no-problems-minikube/)，本文的原理与此文一致，使用阿里云里的镜像来安装Kubernetes集群。

### 1. 安装ansible

使用自动化运维工具ansible进行安装，我本机是MacOS，使用`homebrew`安装`ansible`:

```shell
brew install ansible

```

### 2. 修改kubespray代码

代码修改分别在以下的文件里，请查看源码，修改源码时主要参考阿里云里对应的镜像和版本，以防阿里云无此镜像，查看阿里云镜像请访问[https://dev.aliyun.com/search.html](https://dev.aliyun.com/search.html)。
- `kubespray/roles/kubernetes-apps/ansible/defaults/main.yml`
- `kubespray/roles/download/defaults/main.yml`
- `kubespray/extra_playbooks/roles/download/defaults/main.yml`
- `kubespray/inventory/group_vars/k8s-cluster.yml`
- `kubespray/roles/dnsmasq/templates/dnsmasq-autoscaler.yml`

本文的源码仅为演示作用，大家使用时候可能版本已经有变动，请下载`kubespray`源码，地址为:[https://github.com/kubernetes-incubator/kubespray](https://github.com/kubernetes-incubator/kubespray)。

### 3. inventory.cfg
在`kubespray/inventory/inventory.cfg`，添加内容:
```
[all]
node1    ansible_host=192.168.1.130 ansible_user=root ip=192.168.1.130
node2    ansible_host=192.168.1.131 ansible_user=root ip=192.168.1.131
node3    ansible_host=192.168.1.132 ansible_user=root ip=192.168.1.132

[kube-master]
node1

[kube-node]
node2
node3

[etcd]
node1

[k8s-cluster:children]
kube-node
kube-master

```
### 4. 使用ansible安装

在kubespray根目录，执行:
```shell
 ansible-playbook -u centos -b -i inventory/inventory.cfg cluster.yml
```

### 5. 验证安装

- 登录130:`ssh root@192.168.1.130`
- 查看node:`kubectl get node`
    ```shell
    NAME      STATUS                     AGE       VERSION
    node1     Ready,SchedulingDisabled   49m       v1.6.1+coreos.0
    node2     Ready                      49m       v1.6.1+coreos.0
    node3     Ready                      49m       v1.6.1+coreos.0
    
    ```
- 查看pod:`kubectl get pod --all-namespaces`

    ```shell
    NAMESPACE     NAME                                  READY     STATUS    RESTARTS   AGE
    kube-system   kube-apiserver-node1                  1/1       Running   0          49m
    kube-system   kube-controller-manager-node1         1/1       Running   0          49m
    kube-system   kube-dns-69997447-783dz               3/3       Running   0          48m
    kube-system   kube-proxy-node1                      1/1       Running   0          49m
    kube-system   kube-proxy-node2                      1/1       Running   0          49m
    kube-system   kube-proxy-node3                      1/1       Running   0          49m
    kube-system   kube-scheduler-node1                  1/1       Running   0          49m
    kube-system   kubedns-autoscaler-2506230242-1vcgk   1/1       Running   0          48m
    kube-system   nginx-proxy-node2                     1/1       Running   0          48m
    kube-system   nginx-proxy-node3                     1/1       Running   0          49m
    ```