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

 ### 6. 源码地址
 [https://github.com/wiselyman/kubespray](https://github.com/wiselyman/kubespray)



## kubespray安装kubernetes完成后kubectl客户端配置

接上篇[使用kuberspay无坑安装生产级Kubernetes集群](http://www.wisely.top/2017/07/01/no-problem-kubernetes-kuberspay/),在使用`kubespray`安装好了`kubernetes之`后，我们需要在自己的客户端电脑配置kubectl，如何将集群的配置信息在本地配置呢，我们使用下面的脚本，放在`scripts\copy-kubeconfig.yaml`下，内容为:

```Yaml
---
- hosts: kube-master[0]
  gather_facts: no
  become: yes
  tasks:
  - fetch:
      src: "/etc/kubernetes/ssl/{{ item }}.pem"
      dest: "{{ playbook_dir }}/kubectl/{{ item }}.pem"
      flat: True
    with_items:
      - admin-{{ inventory_hostname }}-key
      - admin-{{ inventory_hostname }}
      - ca
  - name: export hostname
    set_fact:
      kubectl_name: "{{ inventory_hostname }}"

- hosts: localhost
  connection: local
  vars:
    kubectl_name: "{{ hostvars[groups['kube-master'][0]].kubectl_name }}"
    cluster_name: "{{ hostvars[groups['kube-master'][0]].cluster_name }}"
    kube_apiserver_port: "{{ hostvars[groups['kube-master'][0]].kube_apiserver_port }}"
    system_namespace: "{{ hostvars[groups['kube-master'][0]].system_namespace }}"
  tasks:
  - name: "check if context admin@{{ cluster_name }} exists"
    command: kubectl config get-contexts admin@{{ cluster_name }}
    register: kctl
    failed_when: kctl.rc == 0

  - block:
    - name: "create cluster {{ cluster_name }}"
      command: >
        kubectl config set-cluster {{ cluster_name }}
        --server=https://{{ kubectl_name }}:{{ kube_apiserver_port }}
        --certificate-authority={{ playbook_dir }}/kubectl/ca.pem
        --embed-certs

    - name: "create credentials admin"
      command: >
        kubectl config set-credentials admin
        --certificate-authority={{ playbook_dir }}/kubectl/ca.pem
        --client-key={{ playbook_dir }}/kubectl/admin-{{ kubectl_name }}-key.pem
        --client-certificate={{ playbook_dir }}/kubectl/admin-{{ kubectl_name }}.pem
        --embed-certs

    - name: "create context admin@{{ cluster_name }}"
      command: >
        kubectl config set-context admin@{{ cluster_name }}
        --cluster={{ cluster_name }}
        --namespace={{ system_namespace }}
        --user=admin

    - name: "use context admin@{{ cluster_name }}"
      command: kubectl config use-context admin@{{ cluster_name }}
    when: kctl.rc != 0

  - name: "clean up fetched certificates"
    file:
      state: absent
      path: "{{ playbook_dir }}/kubectl"

```



在`kubespray`根目录执行：

`ansible-playbook -i inventory/inventory.cfg scripts/copy-kubeconfig.yaml`

执行完成后，如果你本机的`hosts`文件已经配置了`node1`对应的ip，配置已经完成；如果没有配置，编辑`vi /Users/wangyunfei/.kube/config `，将`node1`修改为`192.168.1.130`