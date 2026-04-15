# 基于Gitlab+Jenkins+Tomcat的Devops平台

--- 

# 一、版本环境

## 1.1 软件版本

Gitlab：17.1.1

Jenkins：2.492.1

Tomcat：10.1.54

JDK：21

## 1.2 环境规划

| 主机名         | IP              | 用途        |
| ----------- | --------------- | --------- |
| gitlab.com  | 192.168.227.101 | 部署git服务器  |
| jenkins.com | 192.168.227.102 | 部署jenkins |
| tomcat.com  | 192.168.227.103 | 部署tomcat  |

# 二、配置

## 2.1 部署Gitlab

> 安装Gitlab

```bash
rpm -ivh gitlab-ce-17.1.1-ce.0.el7.x86_64.rpm
gitlab-ctl reconfigure
gitlab-rails console
Loading production environment (Rails 4.2.8)
irb(main):001:0> user = User.where(id:1).first
irb(main):002:0> user.password='Huawei@123'
irb(main):003:0> user.save!
=> true
ssh-keygen
cat /root/.ssh/id_rsa.pub
```

## 2.2 部署Jenkins

> 安装jenkins

<mark>一定要先启动一次jenkins，然后修改配置文件之后再启动然后登陆网站下载插件</mark>

```bash
rpm -ivh jdk-21_linux-x64_bin.rpm
yum install -y dejavu-sans-fonts fontconfig
fc-cache --force
java -jar jenkins.war
sed -i 's|updates.jenkins.io/download|mirrors.huaweicloud.com/jenkins|g' /root/.jenkins/updates/default.json
sed -i 's|www.google.com|www.baidu.com|g' /root/.jenkins/updates/default.json
vim /root/.jenkins/hudson.model.UpdateCenter.xml
cat /root/.jenkins/hudson.model.UpdateCenter.xml | grep "<url>"
<url>https://mirrors.huaweicloud.com/jenkins/updates/update-center.json</url>
```

> jenkins启动脚本

```bash
vim /etc/systemd/system/jenkins.service
cat /etc/systemd/system/jenkins.service
[Unit]
Description=Jenkins Continuous Integration Server
After=network.target

[Service]
User=root
ExecStart=/usr/bin/java -jar /root/jenkins.war --httpPort=8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

> 启动jenkins

```bash
systctl start jenkins
cat /root/.jenkins/secrets/initialAdminPassword
```

## 2.3 配置Jenkins

> 安装maven

```bash
wget https://mirrors.aliyun.com/apache/maven/maven-3/3.9.14/binaries/apache-maven-3.9.14-bin.tar.gz
tar -zxvf  apache-maven-3.9.14-bin.tar.gz -C /usr/local/
mv apache-maven-3.9.14-bin.tar.gz maven
```

> 下载这个插件

![](C:\Users\30530\AppData\Roaming\marktext\images\2026-04-14-20-54-29-image.png)

> 这个位置填写Jenkis主机的私钥

![](C:\Users\30530\AppData\Roaming\marktext\images\2026-04-14-20-55-53-image.png)

## 三、配置Tomcat

> 配置自动部署

![](C:\Users\30530\AppData\Roaming\marktext\images\2026-04-14-21-00-29-image.png)

> 安装并启动tomcat

```bash
tar -zxvf /root/apache-tomcat-10.1.54.tar.gz -C /usr/local
mv /usr/local/apache-tomcat-10.1.54.tar.gz tomcat
/usr/local/tomcat/bin/startup.sh
```
