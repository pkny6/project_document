# Ceph 安装部署

---

## 安装环境

- 操作系统：CentOS 7.7

- Ceph版本：O版

## 架构规划

| **主机名**     | **IP**          | **节点用途**    |
| ----------- | --------------- | ----------- |
| admin.com   | 192.168.227.100 | OSD         |
| monitor.com | 192.168.227.103 | OSD         |
| node1.com   | 192.168.227.101 | OSD         |
| node2.com   | 192.168.227.102 | MON,MGR,RGW |
| client.com  | 192.168.227.104 | 测试客户端       |

---

## 环境准备

> 以`admin`节点为例

1. **主机名设置**

```bash
hostnamectl set-hostname amin.com
cat >> /etc/hosts << EOF
192.168.227.100 admin.com admin
192.168.227.101 node1.com node1
192.168.227.102 node2.com node2
192.168.227.103 monitor.com monitor
192.168.227.104 client.com client
```

2. **IP地址设置**  

```bash
vim /etc/sysconfig/network/scripts/ifcfg-eth0
```

3. **yum源设置**

> CentOS-Base.repo

```bash
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

> ceph.repo

```bash
vim /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-octopus/el7/x86_64/
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-octopus/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-octopus/el7/x86_64/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
```

> epel仓库

```bash
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
```

安装完这三个`yum`源之后执行`yum makecache` 获取元数据缓存

4. **添加`cephadm`用户**

```bash
#创建用户cephadm
useradd -m -s /bin/bash cephadm 
#非交互式为cephadm添加密码
echo 123 | passwd cephadm --stdin 
#设置sudo授权
echo "cephadm ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephadm
#修改文件权限
chmod 0440 /etc/sudoers.d/cephadm
```

5. **安装`ceph-deploy`部署工具**

```bash
wget http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch/ceph-deploy-2.0.1-0.noarch.rpm
rpm -ivh ceph-deploy-2.0.1-0.noarch.rpm
```

6. **配置`ssh`互信实现免密登录**

```bash
#切换到cephadm用户
su - cephadm
#产生ssh密钥对
ssh-keygen
#分发自己的公钥到其他节点
ss-copy-id -i cephadm@monitor
#测试免密登录是否成功
ssh monitor
```

7. **下载`pecan`模块**

```bash
yum install python36-devel
pip3 install werkzeug pecan
```

---

## 正式安装

```bash
#创建集群并指定MON
su - cephadm
mkdir ceph-cluster
cd ceph-cluster
sudo ceph-deploy new monitor
在每个集群节点及客户端上安装ceph相关软件包
yum install -y ceph ceph-radosgw
#采集keyring并启动MON服务进程
ceph-deploy mon create-initial
#编辑配置文件
echo 'public network = 192.168.227.0/24''>>  ~/ceph-cluster/ceph.conf
#同步配置文件
ceph-deploy admin monitor node1 node2 client
#在MON节点创建MGR进程
ceph-deploy mgr create monitor
#添加osd节点(！！必须是奇数个)
ceph-deploy osd create --data /dev/sdb monitor
ceph-deploy osd create --data /dev/sdb node1
ceph-deploy osd create --data /dev/sdb node2
```
