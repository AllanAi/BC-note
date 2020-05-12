## tail -f

### 查看多个日志文件

`tail -f 1.log 2.log`

可以分段打印每一个文件的log

```
==> 1.log <==
...

==> 2.log <==
...
```

https://blog.csdn.net/weixin_41585557/article/details/82724381

### 多种颜色高亮关键字

```bash
tail -f sys.log | perl -pe 's/(关键词1)|(关键词2)|(关键词3)/\e[1;颜色1$1\e[0m\e[1;颜色2$2\e[0m\e[1;颜色3$3\e[0m/g' 
tail -f sys.log | perl -pe 's/(DEBUG)|(INFO)|(ERROR)/\e[1;34m$1\e[0m\e[1;33m$2\e[0m\e[1;31m$3\e[0m/g'
```

字体颜色设置：30m-37m 黑、红、绿、黄、蓝、紫、青、白

背景颜色设置：40-47 黑、红、绿、黄、蓝、紫、青、白

其他参数说明
[1; 设置高亮加粗
[4; 下划线
[5; 闪烁

例子：
黄字，高亮加粗显示
[1;33m
红底黄字，高亮加粗显示
[1;41;33m

https://blog.csdn.net/qq_27686779/article/details/81180254

## 编辑当前命令

在默认的 Bash 环境下，只要在命令行中按下 `ctrl-x, ctrl-e` 就会把当前命令的内容调入到环境变量 `$EDITOR` 指示的编辑器(默认为 emacs)去编辑，编辑后保存退出就会立即执行。如果未安装 `Emacs` 编辑器，则报错 `-bash: emacs: command not found`。

`export EDITOR=vi` # 使用 `vi` 来编辑当前命令

```bash
$ bind -p | grep -F "\C-x\C-e"
"\C-x\C-e": edit-and-execute-command
```

`bind -p` 可以显示所有的键绑定

https://yanbin.blog/bash-zsh-call-emacs-vim-edit-current-command/

## 无法使用 ALT+*快捷键

1. MAC 自带终端

打开 `mac` 自带的终端工具，按 `command + ,` 打开设置界面，点击上面的 `描述文件` 选项卡，然后在左侧的风格列表中点击你当前使用的风格，然后在右侧出现的选项卡中点击 `键盘` 然后，勾选当前页面的 **将**`Option`**键用作**`meta`**键**

2. iTerm

- 首先用 `command+o` 快捷键打开 `profiles` 设置面板

- 点击左下角的 `Edit Profiles...` 按钮

- 然后就打开了 `Preferences` 设置面板，确保在该面板的 `Profiles` 选项卡中。

- 点击下方右侧的选项卡标签 `Keys`。

- 然后将下方默认的 `Normal` 选项换成 `Esc+` 选项

https://cloud.tencent.com/developer/article/1015508

## 根据历史自动补全

```
~/.inputrc
# Key bindings, up/down arrow searches through history
"\e[A": history-search-backward
"\e[B": history-search-forward
"\eOA": history-search-backward
"\eOB": history-search-forward
```

使用快捷键 CTRL-X CTRL-R 或者`bind -f  ~/.inputrc`重载 .inputrc 文件

也可以在`~/.bashrc`中配置

```
bind '"\e[A": history-search-backward'
```

https://unix.stackexchange.com/questions/5366/command-line-completion-from-command-history

https://superuser.com/questions/241187/how-do-i-reload-inputrc

## 删除乱码或特殊字符文件

```bash
# 查询文件的inode
ls -i
# 查找对应inode文件并删除
find ./ -inum $inode -exec rm {} \;
# 删除文件名含有空格的文件
rm 'ab cd'
```

