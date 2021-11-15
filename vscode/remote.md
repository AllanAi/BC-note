[VScode Remote 远程开发与调试](https://www.jianshu.com/p/0f2fb935a9a1)

1. 安装 `Remote Development` 插件

2. Setting -> Extensions -> Remote - SSH -> 勾选 Show Login Terminal

3. 点击vscode左下角绿色图标或侧边栏的Remote Explorer -> 选择 Remote-SSH：Connect to Host -> Configure SSH Hosts -> 选择一个config

   ```
   # Host是名称，HostName是远程主机的IP地址，User是登录名
   Host test
       HostName x.x.x.x
       User root
       # ProxyCommand nc -v -x 127.0.0.1:9909 %h %p
   ```

4. 点击侧边栏的插件按钮，远程的插件需要单独安装，可以从本地的插件选取安装

5. 使用命令行在远程服务器创建项目后，使用 vscode 打开即可：

   点击侧边栏的Remote Explorer -> 选择远程主机右侧的带加号图标（Connect to Host in New Window）

6. 运行命令和调试和本地类似。

7. 不监控 rust 编译文件夹变化，否则 vscode 弹出无法监控过多文件的警告：Setting -> 搜索 watcherExclude -> 添加 `**/target/**`，或者增加默认文件监控数量：https://code.visualstudio.com/docs/setup/linux#_visual-studio-code-is-unable-to-watch-for-file-changes-in-this-large-workspace-error-enospc

其他插件

- Better TOML
- rust-analyzer
  - "rust-analyzer.checkOnSave.enable": false,
  - "rust-analyzer.lruCapacity": 1024
- IntelliJ IDEA Keybindings
