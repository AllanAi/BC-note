## Terminal

快速导入路径：直接将待编辑文件或文件夹拖入终端中即可

![](images/8d0427a0705812a7d1e7611041f0e9b6.gif)

不进入休眠状态：当你临时不希望电脑进入休眠状态时，可以使用 caffeinate 命令让电脑时刻清醒。

## 显示隐藏文件夹

defaults write com.apple.finder AppleShowAllFiles -bool true; killall Finder

恢复隐藏：defaults write com.apple.finder AppleShowAllFiles -bool false; killall Finder

## 设置代理

https://jdhao.github.io/2019/10/10/mac_proxy_in_terminal/

在 Unix 终端，有三个和代理相关的环境变量，分别为 `http_proxy`, `https_proxy` 和`all_proxy`。粗略来说，`http_proxy` 和 `https_proxy` 分别用来设置 http 和 https连接的代理，`all_proxy` 设置所有连接的代理。

## 快捷键

程序和窗口切换
- 程序切换：command+tab
- 程序内窗口切换：command+`(1左边的按键)

Google chrome
- 切换tab页：command+数字键 或者 command+option+左右方向键
- 打开新的浏览器标签页：command+t
- 浏览器回到页首页尾：command+上下方向键

- 文字编辑时，光标移动到行首行尾：command+左右方向键
- 截屏：command+shift+3(截屏的图片会保存到桌面)
- 鼠标选取截屏：command+shift+4(截屏的图片会保存到桌面)
- 任何应用程序的设置：command+，
- spotlight（全局搜索）：control+space

- ⇞ Page Up（Fn+↑）
- ⇟ Page Down（Fn+↓）
- Home Fn+←
- End Fn+→

## 安装 Xcode

- 安装 git https://git-scm.com/download/mac
- 安装 brew https://docs.brew.sh/Installation
- brew 安装过程中会安装 Xcode

## The default interactive shell is now zsh

Mac更新之后使用终端提示：The default interactive shell is now zsh.
vi ~/.bash_profile
export BASH_SILENCE_DEPRECATION_WARNING=1
https://www.jianshu.com/p/26d365078081

## Homebrew 卡很久

设置代理，或者在 update 的时候按一下 ctrl+c
homebrew 每次安装都会先更新 homebrew，加个环境变量禁止就行了。export HOMEBREW_NO_AUTO_UPDATE=1

## gyp: No Xcode or CLT version detected!

Step1: 输入xcode-select --print-path查看 command-line tools 的安装路径，不出意外显示的结果应该是/Library/Developer/CommandLineTools
Step2: 输入sudo rm -r -f /Library/Developer/CommandLineTools把 command-line tools 从系统移除掉
Step3: 最后输入xcode-select --install重新安装