**Introduction**: https://nodejs.dev/learn

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

```
npm config set proxy http://127.0.0.1:9910
npm config set https-proxy http://127.0.0.1:9910
```

```
npm config list
```

```
npm config delete proxy
npm config delete https-proxy
```



```js
// https://github.com/gajus/global-agent
// npm install global-agent --save-dev
// yarn add global-agent --dev
// Usage:
// $ export GLOBAL_AGENT_HTTP_PROXY=http://127.0.0.1:9910
// $ node -r 'global-agent/bootstrap' your-script.js

var https = require('https');

https.get("https://www.google.com", function (res) {
    console.log("状态码: " + res.statusCode);
})
```

