## UnicodeDecodeError: 'utf-8' codec can't decode byte

`open("data.txt","r").readlines()` 报错：UnicodeDecodeError: 'utf-8' codec can't decode byte...

该情况是由于出现了无法进行转换的 二进制数据 造成的。

解决方法：

```
f = open("data.txt","rb") # 二进制格式读文件
while True:
    line = f.readline()
    if not line:
        break
    else:
        line = line.decode('utf8', "ignore") # 第二个参数errors默认为严格（strict）
```

https://blog.csdn.net/sinat_25449961/article/details/83150624