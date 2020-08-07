`Logrotate` is a tool for managing log files created by system processes. This tool automatically compresses and removes logs to maximize the convenience of logs and conserve system resources.

`logrotate`’s behavior is determined by options set in a configuration file, typically located at `/etc/logrotate.conf`.

Run `logrotate` as a [cronjob](https://www.linode.com/docs/tools-reference/tools/schedule-tasks-with-cron) to ensures that logs will be rotated as regularly as configured. In most cases, `logrotate` is invoked from a script in the `/etc/cron.daily/` directory: `/etc/cron.daily/logrotate`

```
#!/bin/sh
logrotate /etc/logrotate.conf
```

```
Usage: logrotate [OPTION...] <configfile>
  -d, --debug               Don't do anything, just test and print debug messages
  -f, --force               Force file rotation
  -m, --mail=command        Command to send mail (instead of `/usr/bin/mail')
  -s, --state=statefile     Path of state file
  -v, --verbose             Display messages during rotation
```

Most configuration of log rotation does not occur in the `/etc/logrotate.conf` file, but rather in files located in the `/etc/logrotate.d/` directory.

```
/var/log/mail.log {
  weekly
  rotate 5
  compress
  compresscmd xz
  create 0644 postfix postfix
}
```

The above configuration rotates logs every week, saves the last five rotated logs, compresses all of the old log files with the `xz` compression tool, and recreates the log files with permissions of `0644` and `postfix` as the user and group owner. 

logrotate自身的日志存放于/var/lib/logrotate/status

```
dateext //用日期来做轮转之后的文件的后缀名
missingok	//如果日志文件不存在，不报错
notifempty //如果日志文件是空的，不轮转
delaycompress //下一次轮转的时候才压缩
sharedscripts //不管有多少文件待轮转，prerotate和postrotate 代码只执行一次
```

Run Commands Before or After Rotation

```
prerotate
    touch /srv/www/example.com/application/tmp/stop
endscript

postrotate
    touch /srv/www/example.com/application/tmp/start
endscript
```

https://www.linode.com/docs/uptime/logs/use-logrotate-to-manage-log-files/

```
/var/log/nginx/*.log {
    daily
    rotate 5
    missingok
    notifempty
    create 644 www www
    postrotate
      if [ -f /application/nginx/logs/nginx.pid ]; then
          kill -USR1 `cat /application/nginx/logs/nginx.pid`
      fi
endscript
}
```

> USR1亦通常被用来告知应用程序重载配置文件；例如，向Apache HTTP服务器发送一个USR1信号将导致以下步骤的发生：停止接受新的连接，等待当前连接停止，重新载入配置文件，重新打开日志文件，重启服务器，从而实现相对平滑的不关机的更改。
>
> 根据约定，当你发送一个挂起信号(信号1或HUP)时，大多数服务器进程(所有常用的进程)都会进行复位操作并重新加载它们的配置文件。

1. logrotate 执行 rename 系统调用（相当于 mv 命令）重命名日志文件；
2. 由于 inode 不变，这时 nginx 会继续写入重命名后的日志文件；
3. logrotate 给 nginx 发送一个 SIGHUP 信号；
4. nginx 在收到信号后会关闭当前文件，并重新打开日志文件（即创建了新的日志文件）。

https://www.cnblogs.com/clsn/p/8428257.html

在 Linux 下，如何删除一个目录下的所有 log 文件？

```
find . -name \*.log -exec rm -f {} \;
find . -name \*.log -delete 
```

```
find /home/ -type f -name "*.log*" -size +100M -exec bash -c "echo -n > '{}'" \;
```

注意，这个命令没有用 rm，而是使用重定向来清空文件。

你有遇到过删了 log 文件，但是磁盘空间却没有释放的情况吗？

这往往是因为 log 文件正被另一个进程打开。

比如在终端 1 打开 a.txt：

```
$ python
>>> f = open("a.txt")
```

然后在终端 2 可以看到该文件被 Python 打开:

```
$ lsof a.txt
COMMAND  PID ...     NODE NAME
python  2390 ... 12058942 a.txt
```

删掉 a.txt ，再查看 python 打开的文件列表：

```
$ rm a.txt
$ ls -l /proc/2390/fd
lrwx------ 1 user ... 00:04 0 -> /dev/pts/5
lrwx------ 1 user ... 00:04 1 -> /dev/pts/5
lrwx------ 1 user ... 00:04 2 -> /dev/pts/5
lr-x------ 1 user ... 00:04 3 -> /tmp/a.txt (deleted)
```

注：0 、1 、2 、3 是内核的 fd 编号。0=stdin, 1=stdout, 2=stder 。

可以看到，a.txt  被标记为已删除，但因为进程还开着它，可能会访问文件的内容，所以内核会等到进程关闭该文件（或进程退出后）才在磁盘上移除这个文件。

如何才能知道现在系统中有哪些文件已删除、但是仍被占用呢？

```
$ sudo lsof | grep deleted
COMMAND   PID  …  NAME
main   893246  …  /../nohup.out (deleted)
...
```

https://www.v2ex.com/t/687093