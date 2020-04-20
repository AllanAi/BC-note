# 更换软件源

## 更换yum官方源

https://www.cnblogs.com/ziyoufei/p/11721507.html

```shell
# 下载wget工具
yum install -y wget
# 进入yum源配置文件所在文件夹
cd /etc/yum.repos.d/
# 备份本地yum源
mv CentOS-Base.repo CentOS-Base.repo_bak
# 获取国内yum源（阿里、163二选一）
wget -O CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# wget -O CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo

# 清理yum缓存 
yum clean all
# 重建缓存 
yum makecache 
```

## 增加epel源

```shell
# 安装epel源
yum install epel-release
# 修改为阿里的epel源
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```


