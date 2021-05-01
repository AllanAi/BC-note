# 安装与配置

## 安装

```bash
# 通过包管理工具安装
# centos
yum info golang	# 查看 golang 信息
yum install golang	# 安装 golang
# ubuntu
apt-get update
apt-get install golang-go
# mac
brew install go

# 通过压缩包安装 https://studygolang.com/dl
wget https://studygolang.com/dl/golang/go1.14.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.14.linux-amd64.tar.gz
rm go1.14.linux-amd64.tar.gz
```

## 环境变量

```bash
# 配置环境变量
vi /etc/profile
# 添加环境变量
export GOPATH=$HOME/gopath
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
# 使环境变量生效
source /etc/profile

# 查看版本
go version
# 查看环境变量
go env
```

## 修改 goproxy

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

更详细的 go mod 使用方法  https://blog.csdn.net/qq_42403866/article/details/93654421

# 用法

## nil

1. NIL 是不能比较的，`==` 对于 nil 来说是一种未定义的操作

```go
fmt.Println(nil==nil)
//编译错误：invalid operation: nil == nil (operator == not defined on nil)
```

2. NIL 是 MAP，SLICE，POINTER，CHANNEL，FUNC，INTERFACE 的零值
3. 不同类型 NIL 的 ADDRESS 是一样的

```go
var m map[int]string
var ptr *int
fmt.Printf("%p", m)
fmt.Printf("%p", ptr)
```

m 和 ptr 的 address 都是 0x0

4. 不同类型的 NIL 是不能比较的

```go
fmt.Printf(m == ptr)
//编译错误：Invalid operation: m == ptr (mismatched types map[int]string and *int) 
```

https://www.cnblogs.com/reve-wang/articles/7235449.html

5. struct 类型不能和 nil 比较

Pointers, slices, channels, interfaces and maps are the only types that can be compared to nil. A struct value cannot be compared to nil.

If all fields in the struct are comparable, then you can compare with a known zero value of the struct:

` fmt.Println(value == Empty{})`

If the struct contains fields that are **not comparable (slices, maps, channels)**, then you need to write code to check each field:

```go
 func IsAllZero(v someStructType) bool {
     return v.intField == 0 && v.stringField == "" && v.sliceField == nil
 }
 ...
 fmt.Println(isAllZero(value))
```

**Struct values are comparable if all their fields are comparable.** Two struct values are equal if their corresponding non-blank fields are equal.

https://stackoverflow.com/questions/26065525/it-is-an-empty-value-struct-nil

## import internal

"use of internal package not allowed"

An import of a path containing the element “internal” disallowed if the importing code is outside the tree rooted at the parent of the “internal” directory. There is no mechanism for exceptions.

https://stackoverflow.com/questions/41060764/golang-disable-use-of-internal-package-not-allowed

## json

### 格式化输出

`func Indent(dst *bytes.Buffer, src []byte, prefix string, indent string) error`
Indent appends to dst an indented form of the JSON-encoded src. Each element in a JSON object or array begins on a new, indented line beginning with prefix followed by one or more copies of indent according to the indentation nesting. The data appended to dst does not begin with the prefix nor any indentation, to make it easier to embed inside other formatted JSON data. Although leading space characters (space, tab, carriage return, newline) at the beginning of src are dropped, trailing space characters at the end of src are preserved and copied to dst. For example, if src has no trailing spaces, neither will dst; if src ends in a trailing newline, so will dst.

Example
Code:

```go
type Road struct{ 
		Name string
		Number int
	}
	roads := []Road{
		{"Diamond Fork", 29}, 
		{"Sheep Creek", 51}
	}
	b, err := json.Marshal(roads)
	if err != nil {
		log.Fatal(err)
	}
	var out bytes.Buffer
	json.Indent(&out, b, "", "\t")
	out.WriteTo(os.Stdout)
```

Output:

```json
[
	{
		"Name": "Diamond Fork",
		"Number": 29
	},
	{
		"Name": "Sheep Creek",
		"Number": 51
	}
]
```

```go
data := `{"args": {},"headers": {"Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8","Accept-Encoding": "gzip, deflate","Accept-Language": "zh-CN,zh;q=0.9","Connection": "close","Host": "httpbin.org","Upgrade-Insecure-Requests": "1","User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"},"origin": "103.*.*.*","url": "http://httpbin.org/get"}`

var str bytes.Buffer
_ = json.Indent(&str, []byte(data), "", "    ")
fmt.Println("formated: ", str.String())
```

https://www.cnblogs.com/Detector/p/9801339.html

### Tag

```
ProductID int64 `json:"-"` // 表示不进行序列化
Name string `json:"name,omitempty"` //序列化时忽略0值或者空值
Number int `json:"number,string"`//结构体类型和json类型不一致时可以指定,支持string,number和boolean
```

### 自定义解析

实现 UnmarshalJSON 和 MarshalJSON 两个方法

https://www.cnblogs.com/yorkyang/p/8990570.html

## substring

1. 原生方法，直接使用slice切片实现，如果字符串中包含中文，则结果会出现乱码

```go
s:="abcde"
fmt.Println(s[0:2]);
//输出 ab

s2 := "我是中国人"
fmt.Println(s2[0:2])
//输出 ��
```

2. 通过rune来实现

```go
//获取source的子串,如果start小于0或者end大于source长度则返回""
//start:开始index，从0开始，包括0
//end:结束index，以end结束，但不包括end
func substring(source string, start int, end int) string {
    var r = []rune(source)
    length := len(r)

    if start < 0 || end > length || start > end {
        return ""
    }

    if start == 0 && end == length {
        return source
    }

    return string(r[start : end])
}
```

https://blog.csdn.net/psyuhen/article/details/51998223

## Big Integer

math/big.Int

```
//转成字符串
func (x *Int) Text(base int) string
//字符串转bigint
func (z *Int) SetString(s string, base int) (*Int, bool)
//int64转bigint
func NewInt(x int64) *Int
//bigint转int64
func (x *Int) Int64() int64
func (x *Int) Uint64() 

//实现了MarshalJSON和UnmarshalJSON方法
//支持json序列化，其他的序列化方式不支持
func (x *Int) MarshalJSON() ([]byte, error)
func (z *Int) UnmarshalJSON(text []byte) error
```

https://medium.com/orbs-network/big-integers-in-go-14534d0e490d

## 高并发数

1. 并发数高于128时，出现 connection reset by peer 错误 [mac os]

somaxconn参数定义了系统中每一个端口最大的监听队列的长度，默认值为128，限制了每个端口接收新tcp连接侦听队列的大小。

服务器端修改内核参数：`sudo sysctl -w kern.ipc.somaxconn=1024`

查看参数：`sysctl -a | grep somaxconn`

https://blog.csdn.net/jackyechina/article/details/70992308

https://blog.csdn.net/c359719435/article/details/80300433

https://github.com/golang/go/issues/20960#issuecomment-465998114

2. 提高连接数，效果不明显，有明显瓶颈

By default, the Golang HTTP client will do connection pooling. 

```go
//transport.go
var DefaultTransport RoundTripper = &Transport{
        ... 
  MaxIdleConns:          100,
  IdleConnTimeout:       90 * time.Second,
        ... 
}

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2
```

- The `MaxIdleConns: 100` setting sets the size of the connection pool to 100 connections, but *with one major caveat*: this is on a per-host basis.
- The `IdleConnTimeout` is set to 90 seconds, meaning that after a connection stays in the pool and is unused for 90 seconds, it will be removed from the pool and closed.
- The `DefaultMaxIdleConnsPerHost = 2` setting below it. What this means is that even though the entire connection pool is set to 100, **there is a *per-host* cap of only 2 connections**!

默认情况下，针对每个 host，最多只会建立两个长连接，其他的都是短连接（不断建立连接、断开连接）

```go
var HttpClient *http.Client

defaultRoundTripper := http.DefaultTransport
defaultTransportPointer, ok := defaultRoundTripper.(*http.Transport)
if !ok {
	panic(fmt.Sprintf("defaultRoundTripper not an *http.Transport"))
}
defaultTransport := *defaultTransportPointer
//二者需要同时设置
defaultTransport.MaxIdleConns = 200
defaultTransport.MaxIdleConnsPerHost = 200

HttpClient = &http.Client{Transport: &defaultTransport}
```

http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/

# Debug

## IDEA 调试

1. 基础演示 https://studygolang.com/articles/20746
2. 高级功能 https://blog.jetbrains.com/cn/2019/04/%e4%bb%a5-goland-%e8%b0%83%e8%af%95-%e9%ab%98%e7%ba%a7%e8%b0%83%e8%af%95%e5%8a%9f%e8%83%bd/

核心转储调试 (Core Dump debugging) 和可逆调试器 Mozilla rr

## 远程调试

1. 服务器安装Delve

https://github.com/derekparker/delve

2. 查看IDE说明

菜单栏 `Run` -> `EditConfigurations` -> `+` -> `GoRemote`

Before running this configuration, please start your application and Delve as described bellow.  Allow Delve to compile your application: 
`dlv debug --headless --listen=:2345 --api-version=2 --accept-multiclient`
 Or compile the application using Go 1.10 or newer: 
`go build -gcflags \"all=-N -l\" github.com/app/demo`
 and then run it via Delve with the following command: 
`dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./demo`

3. 服务器编译程序

`go build -gcflags \"all=-N -l\" github.com/app/demo`

4. 使用dlv启动程序

`dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./demo`

带参数

`dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./demo -- -c=/config`

等同于`./demo -c=/config`

也可以使用attach命令调试不是dlv启动的程序，首先获取程序进程的pid

`dlv attach $PID --headless --api-version=2 --log --listen=:2345`

5. 本地配置Debug

在刚刚的 `EditConfigurations` 中添加 `GoRemote` 并在Host和Port中配置服务器IP地址和端口号，然后点击调试按钮即可。

https://juejin.im/entry/5d5ce39ef265da039a288b85

https://www.cnblogs.com/mrblue/p/10283936.html



## 打印调用堆栈

```go
fmt.Printf("%s", debug.Stack())
debug.PrintStack()
```

## 计算方法耗时

```go
func elapsed(what string) func() {
    start := time.Now()
    return func() {
        fmt.Printf("%s took %v\n", what, time.Since(start))
    }
}

func main() {
    defer elapsed("page")()  // <-- The trailing () is the deferred call
    time.Sleep(time.Second * 2)
}
```

https://stackoverflow.com/questions/45766572/is-there-an-efficient-way-to-calculate-execution-time-in-golang

## 获取调用信息

```go
package runtime
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
```

skip=0表示返回调用Caller的函数信息，1表示调用该函数的上一层函数，以此类推

```go
//获取函数信息
func FuncForPC(pc uintptr) *Func

func (f *Func) Name() string
func (f *Func) FileLine(pc uintptr) (file string, line int)
```

输出 file:line （两边为空格）到控制台时，可以点击直达对应源码

https://studygolang.com/articles/2327

