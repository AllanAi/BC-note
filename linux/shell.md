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

### tail -F

tail -f 等同于--follow=descriptor，根据文件描述符进行追踪，当文件改名或被删除，追踪停止

tail -F 等同于--follow=name  --retry，根据文件名进行追踪，并保持重试，即该文件被删除或改名后，如果再次创建相同的文件名，会继续追踪

## 目录跳转

```shell
cd	# 跳转到$HOME
cd - # 回到上一次的工作目录
```

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

## 删除文件名乱码或含有特殊字符文件

```bash
# 查询文件的inode
ls -i
# 查找对应inode文件并删除
find ./ -inum $inode -exec rm {} \;
# 删除文件名含有空格的文件
rm 'ab cd'
```

## 字符串中包含换行符等

```shell
STR=$'Hello\nWorld' # single quotes
echo "$STR" # double quotes
Hello
World
```

```
   Words of the form $'string' are treated specially.  The word expands to
   string, with backslash-escaped characters replaced as specified by  the
   ANSI  C  standard.  Backslash escape sequences, if present, are decoded
   as follows:
          \a     alert (bell)
          \b     backspace
          \e
          \E     an escape character
          \f     form feed
          \n     new line
          \r     carriage return
          \t     horizontal tab
          \v     vertical tab
          \\     backslash
          \'     single quote
          \"     double quote
          \nnn   the eight-bit character whose value is  the  octal  value
                 nnn (one to three digits)
          \xHH   the  eight-bit  character  whose value is the hexadecimal
                 value HH (one or two hex digits)
          \cx    a control-x character

   The expanded result is single-quoted, as if the  dollar  sign  had  not
   been present.

   A double-quoted string preceded by a dollar sign ($"string") will cause
   the string to be translated according to the current  locale.   If  the
   current  locale  is  C  or  POSIX,  the dollar sign is ignored.  If the
   string is translated and replaced, the replacement is double-quoted.
```

https://stackoverflow.com/questions/3005963/how-can-i-have-a-newline-in-a-string-in-sh

## 打印管道中间结果

`tail log.txt | tee /dev/tty | grep ERROR`

https://stackoverflow.com/questions/570984

## 管道中使用 curl

不支持：`grep http file.txt | curl`

可以使用：`curl "$(grep http file.txt)"`

https://unix.stackexchange.com/questions/323604

## 判断文件中是否包含指定字符串

```shell
if grep -q SomeString "$File"; then
  Some Actions # SomeString was found
fi
```

Add `-q` option when you don't need the string displayed when it was found.

The `grep` command returns 0 or 1 in the exit code depending on the result of search. 0 if something was found; 1 otherwise.

https://stackoverflow.com/questions/11287861

## Ubuntu Shell 语法报错：Syntax error: Bad for loop variable

例如：`for((i=1;i<=9;i++));do`

以上代码对于标准的bash来说没有错误，原因在于系统默认用的是dash，所以报错。
原因是Ubuntu为了加快开机速度，用dash代替了bash，所以导致了错误。wiki 里面有官方的解释，https://wiki.ubuntu.com/DashAsBinSh，主要原因是dash更小，运行更快，还与POSIX兼容。
取消dash办法：sudo dpkg-reconfigure dash 执行命令后在界面选项中选No，回车确定就OK了。
检查是否已切换到bash上：echo $SHELL ,如是/bin/bash即切换成功。

另一种解决方法：使用`bash some.sh`替代 `sh some.sh`

https://blog.csdn.net/q315099997/article/details/53158353

## source 命令

source命令也称为"点命令"，也就是一个点符号(.)，是bash的内部命令。

功能：使Shell读入指定的Shell程序文件并依次执行文件中的所有语句，source命令通常用于重新执行刚修改的初始化文件，使之立即生效，而不必注销并重新登录。

用法：source filename 或 . filename

区别：

- sh filename 或 ./filename 会建立一个子shell，在子shell中执行脚本里面的语句，该子shell继承父shell的环境变量，但子shell新建的、改变的变量不会被带回父shell。

- source filename：这个命令其实只是简单地读取脚本里面的语句依次在当前shell里面执行，没有建立新的子shell。那么脚本里面所有新建、改变变量的语句都会保存在当前shell里面。

举例：

- 如果想通过脚本切换当前工作目录（使用cd命令），则需要使用 source filename，如果使用sh filename 或 ./filename，则只会改变子shell的工作目录，脚本执行完毕，当前的工作目录并不会改变。

## Help message for shell scripts

Add your message with all the required information on top of your file, just right after the shebang.

```bash
#!/bin/bash
###
### my-script — does one thing well
###
### Usage:
###   my-script <input> <output>
###
### Options:
###   <input>   Input file to read.
###   <output>  Output file to write. Use '-' for stdout.
###   -h        Show this message.
```

Halfway done, now need to get this message in runtime with sed.

```bash
help() {
    sed -rn 's/^### ?//;T;p' "$0"
}
```

`$0` means a filename of a file that is being executed.

A bit about the magic that is going here:

- `s` — stands for substitute the following pattern;
- `/` — keyword to define start/end of the pattern;
- `^### ?` — match a string starting with ### and an optional space after;
- `//` — there could be something between the slashes, but since nothing is here, replace the pattern with an empty string;
- `T` — Jumps to the end of sed-script if no s/// has done a successful substitution;
- `p` — Prints the result of the substitution.

Now just call the `help` function if an arg `-h` or no args passed.

```bash
if [[ $# == 0 ]] || [[ "$1" == "-h" ]]; then
    help
    exit 1
fi
```

https://samizdat.dev/help-message-for-shell-scripts/

## 挂载磁盘

```shell
fdisk -l	# 查看磁盘
mkfs.ext4 /dev/sdb	# 格式化磁盘
```

```shell
# 挂载磁盘
vim /etc/fstab
/dev/sdb                /home/sdb               ext4    defaults        0 0
```

```shell
mount -a	# 重新读取/etc/fstab
df -h	# 查看当前所有文件系统可用空间和使用情况
```

https://blog.51cto.com/zqxiang/1572652

https://www.cnblogs.com/Life-Record/p/5900252.html

## 查看文件夹中文件/文件夹的大小

```
du -sh *
du -sh .[!.]* *
du -- display disk usage statistics
```

## 挂了代理之后，使用hosts中的别名建立ssh连接时报错

`kex_exchange_identification: Connection closed by remote host`
这是因为代理服务器没有本机hosts中的映射，从而导致连接失败
直接使用ip就没有问题

```sh
function sshx(){
        params=("$@")
        i=0
        while [[ i -lt ${#params[@]} ]]; do
                if echo "${params[i]}" | grep @; then
                        #info=($(echo "${params[i]}" | sed 's/@/ /g'))
                        info=(${params[i]/@/ })
                        info[1]=$(dscacheutil -q host -a name "${info[1]}" | grep ip | cut -d' ' -f2)
                        params[i]=${info[0]}@${info[1]}
                fi
                i=$((i + 1))
        done
        ssh -o "ProxyCommand nc -v -x 127.0.0.1:9909 %h %p" ${params[*]}
}
```

## ubuntu 19.10 官方源不提供支持的问题

```
vi /etc/apt/sources.list
:%s/mirrors.linode/old-releases.ubuntu/g
apt update
```

https://www.digitalocean.com/community/questions/i-cannot-update-from-19-04-to-19-10-no-longer-has-a-release-file

## 关机、重启等命令

Use **reboot** to reboot the system. Use **halt** to halt the system without powering it off. Under the SysV init system **halt** had the same effect as **poweroff**, but with systemd as init system (the default since jessie) their effects are different. To power off the machine, use **poweroff** or **shutdown -h now**. The systemd init system provides additional commands that perform the same functions; for example **systemctl reboot** or **systemctl poweroff**.

