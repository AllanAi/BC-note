

| 名称         | 目录                  |
| ------------ | --------------------- |
| 配置文件     | /etc/nginx/nginx.conf |
| 程序文件     | /usr/sbin/nginx       |
| 日志         | /var/log/nginx        |
| 默认虚拟主机 | /var/www/html         |

查看端口占用：`lsof -i:80`

测试nginx.conf文件：`nginx -t -c /etc/nginx/nginx.conf`

安装：apt install nginx

启动：systemctl start nginx

退出：systemctl stop nginx



## RemoteAddr 都是 127.0.0.1

因为使用nginx做了反向代理，所以导致服务端每次获取的都是nginx的地址，即127.0.0.1 。

解决方法：

```
location / { 
    ...
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://192.168.234.131;
    ...
}
```

```go
func getCurrentIP(r http.Request)(string){
    // 这里也可以通过X-Forwarded-For请求头的第一个值作为用户的ip
    // 但是要注意的是这两个请求头代表的ip都有可能是伪造的
    ip := r.Header.Get("X-Real-IP")
    if ip == ""{
        // 当请求头不存在即不存在代理时直接获取ip
        ip = strings.Split(r.RemoteAddr, ":")[0]
    }
    return ip
}
```

https://studygolang.com/articles/12971