# 一、创建带ssh服务的CentOS Docker镜像

## 1、dockerfile

```dockerfile
#基础镜像
FROM centos:latest

#RUN yum update -y
RUN yum install -y openssh-server

#生成ssh-key与配置ssh-key
RUN ssh-keygen -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key
RUN echo 'HostKey /etc/ssh/ssh_host_rsa_key' > /etc/ssh/sshd_config
RUN mkdir -p /root/.ssh
RUN echo '~/.ssh/id_rsa.pub的内容'>/root/.ssh/authorized_keys

#修改root用户登录密码
RUN echo 'root:123'|chpasswd

#开放22端口
EXPOSE 22

#镜像运行时，启动sshd
CMD /usr/sbin/sshd -D
```

## 2、启动方式

docker run -d -p 222:22 imageid

ssh -p 222 root@ip

## 3、参考

https://www.jianshu.com/p/21c873469e73

https://github.com/Freedoms1988/centos7-sshd

https://cloud.tencent.com/developer/article/1081673

https://www.jianshu.com/p/540407aeb55b

# 二、Portainer

Portainer是一款Docker可视化管理工具。

```bash
https://www.portainer.io/installation/
# 拉取镜像
docker pull portainer/portainer

# 一键部署
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

浏览器访问`localhost:9000`即可。

# 三、其他

```bash
docker ps -a #查看所有容器
docker stats #查看统计信息
```

