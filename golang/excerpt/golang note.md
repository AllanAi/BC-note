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

## Big Integer/Float

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

i := big.NewInt(12345)
f := new(big.Float).SetInt(i)
```

https://medium.com/orbs-network/big-integers-in-go-14534d0e490d

## time.After

```go
ticker := time.NewTicker(time.Second)
for {
  select {
    case <-ticker.C:
    	// do something
    case <-time.After(5 * time.Second): // DON'T USE!!!
    	return	// unreachable
  }
}
```

The code above will **never return**, because each time you execute `time.After(4 * time.Second)` you create a new timer channel.

```go
timeout := time.After(5 * time.Second)
pollInt := time.Second

for {
    select {
    case <-endSignal:
        fmt.Println("The end!")
        return
    case <-timeout:
        fmt.Println("There's no more time to this. Exiting!")
        return
    default:
        fmt.Println("still waiting")
    }
    time.Sleep(pollInt)
}
```

https://stackoverflow.com/questions/39212333/how-can-i-use-time-after-and-default-in-golang

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

## pprof 检查内存泄漏

```go
import _ "net/http/pprof"
go func() {
  // web:	http://localhost:6061/debug/pprof
  // terminal:
  // go tool pprof -http=:8081 http://localhost:6061/debug/pprof/profile
  // curl http://localhost:6061/debug/pprof/trace?debug=1 > trace.file
  // go tool trace -http=:8081 trace.file
  fmt.Println(http.ListenAndServe("localhost:6061", nil))
}()
```

https://colobu.com/2019/08/20/use-pprof-to-compare-go-memory-usage/

像Java的一些profiler工具一样， `pprof`也可以比较两个时间点的分配的内存的差值，通过比较差值，就容易看到哪些地方产生的内存"残留"的比较多，没有被内存释放，极有可能是内存泄漏的点。

1. 导出**时间点1**的堆的profile: `curl -s http://localhost:6061/debug/pprof/heap > base.heap`, 我们把它作为基准点
2. 喝杯茶，等待一段时间后导出**时间点2**的堆的profile: `curl -s http://localhost:6061/debug/pprof/heap > current.heap`
3. 现在你就可以比较这两个时间点的堆的差异了: `go tool pprof --base base.heap current.heap`
4. 或者你直接使用命令打开web界面: `go tool pprof --http :9090 --base base.heap current.heap`

## 执行静态分析

https://www.alexedwards.net/blog/an-overview-of-go-tooling

`go vet`工具*会对*您的代码进行静态分析，并警告您某些*可能*与您的代码有问题但编译器不会处理的问题。

```bash
$ go vet foo.go     # Vet the foo.go file
$ go vet .          # Vet all files in the current directory
$ go vet ./...      # Vet all files in the current directory and sub-directories
$ go vet ./foo/bar  # Vet all files in the ./foo/bar directory
```

# 程序设计

https://tomotoes.com/blog/the-top-10-most-common-mistakes-ive-seen-in-go-projects/

## 更完美的封装

一个常见错误是将文件名传递给函数。

假设我们实现一个函数来计算文件中的空行数。最初的实现是这样的：

```go
func count(filename string) (int, error) {
  file, err := os.Open(filename)
  if err != nil {
    return 0, errors.Wrapf(err, "unable to open %s", filename)
  }
  defer file.Close()

  scanner := bufio.NewScanner(file)
  count := 0
  for scanner.Scan() {
    if scanner.Text() == "" {
      count++
    }
  } 
  return count, nil
}
```

假设我们希望在此函数之上实现单元测试，并使用普通文件，空文件，具有不同编码类型的文件等进行测试。代码很容易变得非常难以维护。

此外，如果我们想对于`HTTP Body`实现相同的逻辑，将不得不为此创建另一个函数。

Go 设计了两个很棒的接口：`io.Reader` 和 `io.Writer` (译者注：常见 IO 命令行，文件，网络等)

所以可以传递一个抽象数据源的`io.Reader`，而不是传递文件名。

仔细想一想统计的只是文件吗？一个 HTTP 正文？字节缓冲区？

答案并不重要，重要的是无论`Reader`读取的是什么类型的数据，我们都会使用相同的`Read`方法。

在我们的例子中，甚至可以缓冲输入以逐行读取它（使用`bufio.Reader`及其`ReadLine`方法）：

```go
func count(reader *bufio.Reader) (int, error) {
  count := 0
  for {
    line, _, err := reader.ReadLine()
    if err != nil {
      switch err {
      default:
        return 0, errors.Wrapf(err, "unable to read")
      case io.EOF:
        return count, nil
      }
    }
    if len(line) == 0 {
      count++
    }
  }
}
```

打开文件的逻辑现在交给调用`count`方：

```go
file, err := os.Open(filename)
if err != nil {
  return errors.Wrapf(err, "unable to open %s", filename)
}
defer file.Close()
count, err := count(bufio.NewReader(file))
```

无论数据源如何，都可以调用`count`。并且，还将促进单元测试，因为可以从字符串创建一个`bufio.Reader`，这大大提高了效率。

```go
count, err := count(bufio.NewReader(strings.NewReader("input")))
```

## Channel 应用模式

https://colobu.com/2018/03/26/channel-patterns/
