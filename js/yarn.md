# 设置代理

以下命令不要在js项目目录下运行

```
yarn config set proxy http://127.0.0.1:9910
yarn config set https-proxy http://127.0.0.1:9910
```

取消代理

```
yarn config delete proxy
yarn config delete https-proxy
```

查看所有配置

```
yarn config list
```

https://classic.yarnpkg.com/en/docs/cli/config/

npm也有同样的命令

