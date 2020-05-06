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

适用于：git clone git@github.com:owner/git.git

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

## 工作区、缓存区

![3476025246-5c3308667511a_articlex](images/3476025246-5c3308667511a_articlex.png)

工作区即为本地的相应文件夹，通过`git add`命令，即可将文件放置缓存区；继而通过`git commit`即可将文件放置于相对应的git分支。

通过`git status`，可以判断当前文件所属位置：

- 出现 `Untracked files`，代表当前文件出在工作区，还未进入缓存区
- 出现`new file modified`等，代表文件在缓存区，还未进入当前所在分支

**工作区和缓存区内容是公共的，不从属于任何一个分支。**所以切换分支时，会将未add或未commit的内容也切换过去。

https://segmentfault.com/a/1190000017794371

## 指定 git 命令运行目录

https://stackoverflow.com/questions/3769137/use-git-log-command-in-another-folder

` git -C <directory> <command>`

例如：`git -C ~/foo status`

## commit 历史中找回已删除文件

https://stackoverflow.com/questions/7203515/how-to-find-a-deleted-file-in-the-project-commit-history

Get a list of the deleted files and copy the full path of the deleted file

```bash
git log --diff-filter=D --summary | grep delete
```

Execute the next command to find commit id of that commit and copy the commit id

```bash
git log --all -- FILEPATH
```

restore the file or directory

```bash
git checkout <commit id> -- /path/to/file
git checkout <commit id> -- /path/to/dir/**
```

## 找回丢失的本地 commit

https://blog.csdn.net/qq_38233837/article/details/84833066

本地 commit 后，没有 push 到服务器，切换分支后，本地提交丢失

```bash
# 查看所有 git 操作历史
git reflog
# 切换到最后的 commit 
git reset --hard HEAD@{7}
```

## Cherry-Pick

https://juejin.im/entry/5c24ecb6f265da61553ae35e

1. 关闭cherry-pick自动提交

![cherry-pick-01](images/cherry-pick-01.png)

2. 切换到没有对应commit的分支
3. 在git窗口切换到commit所在分支，选中（可多选）需要提交的commit，然后右键Cherry-Pick即可。