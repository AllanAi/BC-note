# 安装与配置

```bash
# 通过包管理工具安装
# centos
yum info golang	# 查看 golang 信息
yum install golang	# 安装 golang
# ubuntu
apt-get update
apt-get install golang-go
# mac
brew install go

# 通过压缩包安装 https://studygolang.com/dl
wget https://studygolang.com/dl/golang/go1.14.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.14.linux-amd64.tar.gz
rm go1.14.linux-amd64.tar.gz
```

## 环境变量

```bash
# 配置环境变量
vi /etc/profile
# 添加环境变量
export GOPATH=$HOME/gopath
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
# 使环境变量生效
source /etc/profile

# 查看版本
go version
# 查看环境变量
go env
```

## 修改 goproxy

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

更详细的 go mod 使用方法  https://blog.csdn.net/qq_42403866/article/details/93654421

# Debug

## 1. 打印调用堆栈

```go
fmt.Printf("%s", debug.Stack())
debug.PrintStack()
```

