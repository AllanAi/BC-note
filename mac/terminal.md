# Terminal

快速导入路径：直接将待编辑文件或文件夹拖入终端中即可

![](images/8d0427a0705812a7d1e7611041f0e9b6.gif)

不进入休眠状态：当你临时不希望电脑进入休眠状态时，可以使用 caffeinate 命令让电脑时刻清醒。

显示隐藏文件夹：defaults write com.apple.finder AppleShowAllFiles -bool true; killall Finder

恢复隐藏：defaults write com.apple.finder AppleShowAllFiles -bool false; killall Finder