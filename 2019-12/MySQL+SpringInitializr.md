# MySQL

## 连接 MySQL 8.0 以上版本

https://www.runoob.com/java/java-mysql-connect.html

1. 使用MySQL 8.0 以上版本驱动包版本

2. com.mysql.jdbc.Driver 更换为 com.mysql.cj.jdbc.Driver。

3. MySQL 8.0 以上版本不需要建立 SSL 连接的，需要显示关闭。

4. 最后还需要设置 CST。

**8.0及以上版本不兼容5.X版本接口，二者不可以互相替代。**

## 数据库名称中包含短横线

https://blog.csdn.net/liumiaocn/article/details/89403642

```
mysql> create database jeecg-boot charset=utf8;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '-boot charset=utf8' at line 1

//使用反引号
mysql> create database `jeecg-boot` charset=utf8;
Query OK, 1 row affected (0.01 sec)
```

## 将命令行客户端日志输出到文件

https://blog.csdn.net/yang131631/article/details/78720177

```
//在客户端中输入以下命令
tee 文件名完整路径
```

## 导入sql文件

https://www.jianshu.com/p/aa64a3a6d8c5

```
//切换到对应数据库
use xxx;
//导入文件，文件路径不能有引号
source ...\xxx.sql
```

### 一次导入多个sql文件

新建一个sql文件，内容如下：

```
SOURCE test1.sql;
SOURCE test2.sql;
```

# Redis Windows

下载地址：https://github.com/microsoftarchive/redis/releases

# Spring Initializr

https://start.spring.io/ 访问不稳定，编译一个本地版本使用

项目：https://github.com/spring-io/start.spring.io

## 编译

下载项目后，修改根目录pom.xml中的以下插件的版本

```
<plugin>
	<groupId>com.github.eirslett</groupId>
	<artifactId>frontend-maven-plugin</artifactId>
	<version>1.8.0</version>
</plugin>
```

在项目根目录运行：mvn -DskipTests clean install

安装过程中，可能出现yarn相关的错误，多尝试几次，或者删除 start.spring.io-master\start-client\node\yarn\dist 目录下的文件重试，或者到https://github.com/yarnpkg/yarn/releases下载 tar.gz 解压到上述目录。

安装过程中，可能出现文件格式错误，按照提示运行相关命令即可。

## 启动

```bash
cd start-site
mvn spring-boot:run
```



# Docker Compose

## Install Compose

https://docs.docker.com/compose/install/

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

docker-compose命令和docker命令一样，需要管理员权限，在命令前面加上sudo，否则报错：“ERROR: Couldn't connect to Docker daemon at……”。