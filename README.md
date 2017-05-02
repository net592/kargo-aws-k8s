
![Kubernetes Logo](https://s28.postimg.org/lf3q4ocpp/k8s.png)

## 说明 目前K8S Aws 部署图
- 使用阿里云镜像副本
- 使用默认calico 网络插件
- 使用AMi CentOS Linux release 7.3.1611 (Core)
- 镜像仓库使用Harbor(自行安装)


![Kubernetes Logo](http://omwdjgaw1.bkt.clouddn.com/aws-k8s.png)


## 部署安装流程 该项目已经配置直接使用好了
- [1]:初始环境 关闭selinux 防火墙
- [2]:先安装 Ansible
- [3]:克隆项目 初始化安装配置
git clone https://github.com/net592/kargo-aws


- [1]:初始环境 关闭selinux 防火墙
```
# 设置机器名
hostnamectl set-hostname K8S-LIN-NODE01
#停防火墙
systemctl stop firewalld
systemctl disable firewalld
systemctl disable firewalld

#SELINUX OFF
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config 

sestatus
getenforce

#Host更新 gooogle 镜像库
cat <<EOF >> /etc/hosts
61.91.161.217 google.com
61.91.161.217 gcr.io    #在国内，你可能需要指定gcr.io的host，如不指定你需要手动下载google-containers. 如何下载其他文档说的方法挺多
61.91.161.217 www.gcr.io
61.91.161.217 console.cloud.google.com
61.91.161.217 storage.googleapis.com
EOF

#更新 Repo

curl -o /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo 

curl -o  /etc/yum.repos.d/epel-7.repo  http://mirrors.aliyun.com/repo/epel-7.repo

```

- [2]:初始环境 安装Ansible
```
# 安装ansible
yum install -y python-pip python34 python-netaddr python34-pip ansible lrzsz

# 安装pip
curl -O https://bootstrap.pypa.io/get-pip.py
python get-pip.py 

# 升级 否则报错 no test named 'equalto'

pip install --upgrade Jinja2==2.8
pip install netaddr

# Check 
[centos@ip-10-0-0-1 ~]$ ansible --version
ansible 2.2.1.0

# 同步ssh key 用户ansible
上传awskey 自带key
chmor 0600 awsdockerkey.pem

# 若非key 登陆 使用自己ssh key也行
ssh-keygen -t rsa -N ""

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.0.0.1

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.0.0.2
省略10.0.0.2 10.0.0.5

```

- [3]:克隆项目 初始化安装配置
```
cd kargo-aws-k8s

# 如果需要变更OS 
vim inventory/group_vars/k8s-cluster.yml

# 启动集群的基础系统
bootstrap_os: centos

# 其他选型之前改了
# 如果需要变更IP 网段和DNS 请修改

# 服务地址分配
kube_service_addresses: 10.233.0.0/18

# pod 地址分配
kube_pods_subnet: 10.233.64.0/18

# 默认 dns 后缀
cluster_name: cluster.local


```
## 生成集群配置
* 配置完基本集群参数后，还需要生成一个集群配置文件，用于指定需要在哪几台服务器安装，和指定 master、node 节点分布，以及 etcd 集群等安装在那几台机器上
```
# 定义集群 IP
IPS=(10.0.0.1 10.0.0.2 10.0.0.3  10.0.0.4 10.0.0.5 )
# 生成配置
cd kargo-aws-k8s
CONFIG_FILE=inventory/inventory.cfg python3 contrib/inventory_builder/inventory.py ${IPS}
```
```
vim inventory/inventory.cfg

[all]
node1    ansible_host=192.168.1.11 ip=10.0.0.1
node2    ansible_host=192.168.1.12 ip=10.0.0.2
node3    ansible_host=192.168.1.13 ip=10.0.0.3
node4    ansible_host=192.168.1.14 ip=10.0.0.4
node5    ansible_host=192.168.1.15 ip=10.0.0.5

[kube-master]
node1
node2
node3

[kube-node]
node1
node2
node3
node4
node5

[etcd]
node1
node2
node3

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]
```

## 一键部署
# 走你(没梯子先 load 好镜像)
cd kargo-aws-k8s
# 1.使用AWS 自带的Key 如 awsdockerkey.pem

ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=./awsdockerkey.pem -u centos


# 2.使用私钥指定的是每个虚拟机 ssh 目录下的私钥
ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=~/.ssh/id_rsa



# 重置清除所有配置
# 卸载

cd kargo

ansible-playbook -i inventory/inventory.cfg reset.yml -b -v --private-key=~/.ssh/id_rsa
# 增加节点 如增加 node6
```
# 定义集群 IP
IPS=(10.0.0.1 10.0.0.2 10.0.0.3  10.0.0.4 10.0.0.5 10.0.0.6)
# 生成配置
cd kargo-aws-k8s
CONFIG_FILE=inventory/inventory.cfg python3 contrib/inventory_builder/inventory.py ${IPS}
ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=./awsdockerkey.pem -u centos --limit node6
```


## 下面文档不用看 官方写的不适用中国环境 Deploy a production ready kubernetes cluster

If you have questions, join us on the [kubernetes slack](https://slack.k8s.io), channel **#kargo**.

- Can be deployed on **AWS, GCE, Azure, OpenStack or Baremetal**
- **High available** cluster
- **Composable** (Choice of the network plugin for instance)
- Support most popular **Linux distributions**
- **Continuous integration tests**


To deploy the cluster you can use :

[**kargo-cli**](https://github.com/kubespray/kargo-cli) <br>
**Ansible** usual commands and [**inventory builder**](https://github.com/kubernetes-incubator/kargo/blob/master/contrib/inventory_builder/inventory.py) <br>
**vagrant** by simply running `vagrant up` (for tests purposes) <br>


*  [Requirements](#requirements)
*  [Kargo vs ...](docs/comparisons.md)
*  [Getting started](docs/getting-started.md)
*  [Ansible inventory and tags](docs/ansible.md)
*  [Deployment data variables](docs/vars.md)
*  [DNS stack](docs/dns-stack.md)
*  [HA mode](docs/ha-mode.md)
*  [Network plugins](#network-plugins)
*  [Vagrant install](docs/vagrant.md)
*  [CoreOS bootstrap](docs/coreos.md)
*  [Downloaded artifacts](docs/downloads.md)
*  [Cloud providers](docs/cloud.md)
*  [OpenStack](docs/openstack.md)
*  [AWS](docs/aws.md)
*  [Azure](docs/azure.md)
*  [Large deployments](docs/large-deployments.md)
*  [Upgrades basics](docs/upgrades.md)
*  [Roadmap](docs/roadmap.md)

Supported Linux distributions
===============

* **Container Linux by CoreOS**
* **Debian** Jessie
* **Ubuntu** 16.04
* **CentOS/RHEL** 7

Note: Upstart/SysV init based OS types are not supported.

Versions of supported components
--------------------------------

[kubernetes](https://github.com/kubernetes/kubernetes/releases) v1.5.1 <br>
[etcd](https://github.com/coreos/etcd/releases) v3.0.17 <br>
[flanneld](https://github.com/coreos/flannel/releases) v0.6.2 <br>
[calicoctl](https://github.com/projectcalico/calico-docker/releases) v0.23.0 <br>
[canal](https://github.com/projectcalico/canal) (given calico/flannel versions) <br>
[weave](http://weave.works/) v1.8.2 <br>
[docker](https://www.docker.com/) v1.12.5 <br>
[rkt](https://coreos.com/rkt/docs/latest/) v1.21.0 <br>

Note: rkt support as docker alternative is limited to control plane (etcd and
kubelet). Docker is still used for Kubernetes cluster workloads and network
plugins' related OS services. Also note, only one of the supported network
plugins can be deployed for a given single cluster.

Requirements
--------------

* **Ansible v2.2 (or newer) and python-netaddr is installed on the machine
  that will run Ansible commands**
* **Jinja 2.8 (or newer) is required to run the Ansible Playbooks**
* The target servers must have **access to the Internet** in order to pull docker images.
* The target servers are configured to allow **IPv4 forwarding**.
* **Your ssh key must be copied** to all the servers part of your inventory.
* The **firewalls are not managed**, you'll need to implement your own rules the way you used to.
in order to avoid any issue during deployment you should disable your firewall.


## Network plugins
You can choose between 4 network plugins. (default: `calico`)

* [**flannel**](docs/flannel.md): gre/vxlan (layer 2) networking.

* [**calico**](docs/calico.md): bgp (layer 3) networking.

* [**canal**](https://github.com/projectcalico/canal): a composition of calico and flannel plugins.

* **weave**: Weave is a lightweight container overlay network that doesn't require an external K/V database cluster. <br>
(Please refer to `weave` [troubleshooting documentation](http://docs.weave.works/weave/latest_release/troubleshooting.html)).

The choice is defined with the variable `kube_network_plugin`. There is also an
option to leverage built-in cloud provider networking instead.
See also [Network checker](docs/netcheck.md).

## Community docs and resources
 - [kubernetes.io/docs/getting-started-guides/kargo/](https://kubernetes.io/docs/getting-started-guides/kargo/)
 - [kargo, monitoring and logging](https://github.com/gregbkr/kubernetes-kargo-logging-monitoring) by @gregbkr
 - [Deploy Kubernetes w/ Ansible & Terraform](https://rsmitty.github.io/Terraform-Ansible-Kubernetes/) by @rsmitty
 - [Deploy a Kubernets Cluster with Kargo (video)](https://www.youtube.com/watch?v=N9q51JgbWu8)

## Tools and projects on top of Kargo
 - [Digital Rebar](https://github.com/digitalrebar/digitalrebar)
 - [Kargo-cli](https://github.com/kubespray/kargo-cli)
 - [Fuel-ccp-installer](https://github.com/openstack/fuel-ccp-installer)
 - [Terraform Contrib](https://github.com/kubernetes-incubator/kargo/tree/master/contrib/terraform)

## CI Tests

![Gitlab Logo](https://s27.postimg.org/wmtaig1wz/gitlabci.png)

[![Build graphs](https://gitlab.com/kargo-ci/kubernetes-incubator__kargo/badges/master/build.svg)](https://gitlab.com/kargo-ci/kubernetes-incubator__kargo/pipelines) </br>

CI/end-to-end tests sponsored by Google (GCE), DigitalOcean, [teuto.net](https://teuto.net/) (openstack).
See the [test matrix](docs/test_cases.md) for details.
