

| 名称         | 目录                  |
| ------------ | --------------------- |
| 配置文件     | /etc/nginx/nginx.conf |
| 程序文件     | /usr/sbin/nginx       |
| 日志         | /var/log/nginx        |
| 默认虚拟主机 | /var/www/html         |

查看端口占用：`lsof -i:80`

测试nginx.conf文件：`nginx -t -c /etc/nginx/nginx.conf`

