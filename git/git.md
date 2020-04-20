## Github

### 生成 SSH Keys

使用 SSH 连接 github 操作（如 `git clone git@github.com:...`）时，报错 `git@github.com: Permission denied (publickey).`

解决方法：

运行以下命令生成本机 SSH 密钥：

```bash
mkdir ~/.ssh //in case that the folder doesnt exist...
cd ~/.ssh

ssh-keygen -t rsa -C "youremail@somewhere.gr"
```

然后一路回车即可。

使用 `cat id_rsa.pub` 命令查看公钥，并将公钥加到 `https://github.com/settings/keys` 

## 设置代理

https://gist.github.com/chuyik/02d0d37a49edc162546441092efae6a1

### 1.1 HTTP 形式

适用于：git clone https://github.com/owner/git.git

#### 走 HTTP 代理

```bash
git config --global http.proxy "http://127.0.0.1:8080"
git config --global https.proxy "http://127.0.0.1:8080"
```

#### 走 socks5 代理（如 Shadowsocks）

```bash
git config --global http.proxy "socks5://127.0.0.1:1080"
git config --global https.proxy "socks5://127.0.0.1:1080"
```

#### 取消设置

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### 1.2 SSH 形式

适用于：git clone [git@github.com](mailto:git@github.com):owner/git.git

修改 `~/.ssh/config` 文件（不存在则新建）：

```
# 必须是 github.com
Host github.com
   HostName github.com
   User git
   # 走 HTTP 代理
   # ProxyCommand socat - PROXY:127.0.0.1:%h:%p,proxyport=8080
   # 走 socks5 代理（如 Shadowsocks）
   # ProxyCommand nc -v -x 127.0.0.1:1080 %h %p
```

## 查看配置信息

```shell
# 查看系统配置
git config --system --list
# 查看当前用户配置
git config --global  --list
# 查看当前仓库配置
git config --local  --list
```

# 