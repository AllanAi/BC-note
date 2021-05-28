# 一、摘录及笔记

[go tour](excerpt/go tour.md)

[gospec](excerpt/gospec.md)

[How to Write Go Code](excerpt/How to Write Go Code.md)

[Effective Go](excerpt/Effective Go.md)

[common-mistakes](excerpt/common-mistakes.md)

[golang note](excerpt/golang note.md)

[The Way to Go](https://github.com/unknwon/the-way-to-go_ZH_CN)

[golang面试题集合](https://github.com/lifei6671/interview-go)

[介绍Go语言调度器](https://mp.weixin.qq.com/s/GhC2WDw3VHP91DrrFVCnag)

[Go语言面试题笔试题](https://github.com/geektutu/interview-questions)

[Golang 性能优化实战](https://mp.weixin.qq.com/s/ogtRE_LbllN2Tla97LnFrQ)

[Learn Go with tests](https://studygolang.gitbook.io/learn-go-with-tests/)

# 二、语法

## 基本类型

```
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // uint8 的别名

rune // int32 的别名
    // 表示一个 Unicode 码点

float32 float64

complex64 complex128
```

`int`, `uint` 和 `uintptr` 在 32 位系统上通常为 32 位宽，在 64 位系统上则为 64 位宽

### 零值

`false` for booleans, `0` for numeric types, `""` for strings, and `nil` for pointers, functions, interfaces, slices, channels, and maps. This initialization is done recursively, so for instance each element of an array of structs will have its fields zeroed if no value is specified.

## import

```
Import declaration          Local name of Sin

import   "lib/math"         math.Sin
import m "lib/math"         m.Sin
import . "lib/math"         Sin
```

## 流程控制

### for

```go
for i := 0; i < 10; i++ {
	sum += i
}
// go没有while
for sum < 1000 {
	sum += sum
}
// 无限循环
for {
}
// range返回的是副本
// v is a copy of the actual collection elements 且 v 会被重用
for i, v := range x
```

在for循环里append元素会造成死循环吗？

```go
func main() {
 s := []int{1,2,3,4,5}
 for _, v:=range s {
  s =append(s, v)
  fmt.Printf("len(s)=%v\n",len(s))
 }
}
```

**不会死循环**，`for range`其实是`golang`的`语法糖`，在循环开始前会获取切片的长度 `len(切片)`，然后再执行`len(切片)`次数的循环。

### switch

```go
switch os := runtime.GOOS; os {
    case "darwin":
    fmt.Println("OS X.")
    case "linux":
    fmt.Println("Linux.")
    default:
    // freebsd, openbsd,
    // plan9, windows...
    fmt.Printf("%s.\n", os)
}
// switch true
switch {
    case t.Hour() < 12:
    	fmt.Println("Good morning!")
    case t.Hour() < 17:
    	fmt.Println("Good afternoon.")
    default:
    	fmt.Println("Good evening.")
}
switch c {
	case ' ', '?', '&', '=', '#', '+', '%':
		return true
}
return false
```



### defer

```go
// prints 3 2 1 0 before surrounding function returns
for i := 0; i <= 3; i++ {
	defer fmt.Print(i)
}

// f returns 42
// return 在 defer 之前执行
func f() (result int) {
	defer func() {
		// result is accessed after it was set to 6 by the return statement
		result *= 7
	}()
	return 6
}

// 计算方法耗时
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

## 结构体

结构体可以包含一个或多个 **匿名（或内嵌）字段**，即这些字段没有显式的名字，只有字段的类型是必须的，此时类型就是字段的名字。匿名字段本身可以是一个结构体类型，即 **结构体可以包含内嵌结构体**。

Go 语言中的继承是通过内嵌或组合来实现的。

空结构体

使用空结构体 struct{} 可以节省内存，一般作为占位符使用，表明这里并不需要一个值。

```
fmt.Println(unsafe.Sizeof(struct{}{})) // 0
```

比如使用 map 表示集合时，只关注 key，value 可以使用 struct{} 作为占位符。如果使用其他类型作为占位符，例如 int，bool，不仅浪费了内存，而且容易引起歧义。

```
type Set map[string]struct{}
```

再比如，使用信道(channel)控制并发时，我们只是需要一个信号，但并不需要传递值，这个时候，也可以使用 struct{} 代替。

```
ch := make(chan struct{})
```

再比如，声明只包含方法的结构体。

```
type Lamp struct{}
func (l Lamp) On() {
	println("On")
}
func (l Lamp) Off() {
	println("Off")
}
```



## 数组

类型 `[n]T` 表示拥有 `n` 个 `T` 类型的值的数组。数组的长度是其类型的一部分，因此数组不能改变大小。

```golang
var a [2]string
a[0] = "Hello"
a[1] = "World"

primes := [6]int{2, 3, 5, 7, 11, 13}

b := [...]string{"Penn", "Teller"} // 让编译器统计数组字面值中元素的数目
```

数组**不需要显式的初始化**；数组的零值是可以直接使用的，数组元素会自动初始化为其对应类型的零值。

Go的数组是**值语义**。一个数组变量表示整个数组，它不是指向第一个元素的指针（不像 C 语言的数组）。 当一个数组变量被赋值或者被传递的时候，实际上会复制整个数组。 （为了避免复制数组，你可以传递一个指向数组的指针，但是数组指针并不是数组。）

## 切片

**切片就像数组的引用**。切片并不存储任何数据，它只是描述了底层数组中的一段。更改切片的元素会修改其底层数组中对应的元素。

一个切片是一个数组片段的描述。它包含了指向数组的指针，片段的长度， 和容量（片段的最大长度）。

```
a[low : high] // simple
a[low : high : max] // full, len=high-low, cap=max-low
```

数组文法：

```go
[3]bool{true, true, false}
```

下面这样则会创建一个和上面相同的数组，然后构建一个引用了它的切片：

```go
[]bool{true, true, false}
```

切片 `s` 的长度和容量可通过表达式 `len(s)` 和 `cap(s)` 来获取。

切片的**零值**是 `nil`。nil 切片的长度和容量为 0 且没有底层数组。

`make` 函数会分配一个元素为零值的数组并返回一个引用了它的切片：

```go
a := make([]int, 5)  // len(a)=5
```

要指定它的容量，需向 `make` 传入第三个参数：

```go
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

// these two expressions are equivalent
make([]int, 50, 100)
new([100]int)[0:50]
```

追加元素

```go
func append(s []T, vs ...T) []T // within func the type of `vs` is equivalent to `[]T`
```

如果是要将一个切片追加到另一个切片尾部，需要使用 `...` 语法将第2个参数展开为参数列表。`...` 切片传入时**不会生成新的切片**，也就是说函数内部使用的切片与传入的切片共享相同的存储空间。说得再直白一点就是，如果函数内部修改了切片，可能会影响外部调用的函数。

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
s = append([]byte("hello"), "world"...)
```

调用append函数时，当原有长度加上新追加的长度如果超过容量则会新建一个数组，新旧切片会指向不同的数组；如果没有超过容量则在原有数组上追加元素，新旧切片会指向相同的数组，这时对其中一个切片的修改会同时影响到另一个切片。其伪代码在如下文件里，而实际上append会在编译时期被当成一个TOKEN直接编译成汇编代码，因此append并不是在运行时调用的一个函数。

https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/builtin.go

```
//	expand append(src, a [, b]* ) to
//
//   init {
//     s := src
//     const argc = len(args) - 1
//     if cap(s) - len(s) < argc {
//	    s = growslice(s, len(s)+argc)
//     }
//     n := len(s)
//     s = s[:n+argc]
//     s[n] = a
//     s[n+1] = b
//     ...
//   }
//   s
```

因此即使src没有初始化，append也可以正常执行

```go
var a []int
a = append(a, 1)
```

复制切片

```go
func copy(dst, src []T) int
func copy(dst []byte, src string) int
```

`copy` 函数支持不同长度的切片之间的复制（它只复制较短切片的长度个元素）。 此外， `copy` 函数可以正确处理源和目的切片有重叠的情况。

## map

map 的零值为 `nil` 。

```go
var m map[string]int // 声明
m = make(map[string]int) // 初始化
// 或者
m := map[string]int{}

var m = map[string]int{
	"a": 1,
	"b": 2,
}

delete(m, "a") //删除元素

// 当m为nil或不存在a这个键时，elem为值类型的零值
elem := m["a"]

elem, ok := m["a"]

for key, value := range m
```

The key can be of any type for which the equality operator is defined, such as integers, floating point and complex numbers, strings, pointers, interfaces (as long as the dynamic type supports equality), structs and arrays. **Slices cannot be used as map keys, because equality is not defined on them.** 

Like slices, maps hold references to an underlying data structure. If you pass a map to a function that changes the contents of the map, the changes will be visible in the caller.

## 方法

以指针为接收者的方法被调用时，接收者既能为值（as long as the value is addressable）又能为指针；

而以值为接收者的方法被调用时，接收者既能为值又能为指针。

```go
var v Vertex
v.Scale(5)  // (&v).Scale(5)
v.Abs()
p := &v
p.Scale(10)
p.Abs()		// (*p).Abs()
```

Not every variable is addressable though. Map elements are not addressable. Variables referenced through interfaces are also not addressable. 还有字符串中的字节、常量也是不可寻址的。

```go
d1 := data{"one"}
d1.print() //ok

var in printer = data{"two"} //error
in.print()

m := map[string]data {"x":data{"three"}}
m["x"].print() //error
```

map的value本身是不可寻址的，因为map中的值会在内存中移动，并且旧的指针地址在map改变时会变得无效。故如果需要修改map值，可以将`map`中的非指针类型`value`，修改为指针类型。

## string

```go
var b byte
var s = "hello"
b = s[0]
var s1 = s[0:2] //substring，不支持中文

for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

string 和[]byte 之间类型转换时，会发生内存拷贝。严格来说，只要是发生类型强转都会发生内存拷贝。

```go
func String(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
func Str2Bytes(s string) []byte {
    x := (*[2]uintptr)(unsafe.Pointer(&s))
    h := [3]uintptr{x[0], x[1], x[1]}
    return *(*[]byte)(unsafe.Pointer(&h))
}
```

翻转含有中文、数字、英文字母的字符串

```
func main() {
 src := "你好abc啊哈哈"
 dst := reverse([]rune(src))
 fmt.Printf("%v\n", string(dst))
}
func reverse(s []rune) []rune {
 for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
  s[i], s[j] = s[j], s[i]
 }
 return s
}
```

- `rune`关键字，是int32的别名（-2^31 ~ 2^31-1），比起byte（-128～127），**可表示更多的字符**。
- 由于rune可表示的范围更大，能处理**中文字符**。

## 接口

```go
type Abser interface {
	Abs() float64
}
```

类型通过实现一个接口的所有方法来实现该接口。无需专门显式声明，也没有“implements”关键字。

**底层值为 nil 的接口值**

即便接口内的具体值为 nil，方法仍然会被 nil 接收者调用。

在一些语言中，这会触发一个空指针异常，但在 Go 中通常会写一些方法来优雅地处理它（如本例中的 `M` 方法）。

**注意:** 保存了 nil 具体值的接口其自身并不为 nil。

```go
type I interface {
	M()
}
type T struct {
	S string
}
func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}
func main() {
	var i I
	var t *T
	i = t
	i.M()
}
```

### 空接口

指定了零个方法的接口值被称为 *空接口：*

```go
interface{}
```

空接口可保存任何类型的值。（因为每个类型都至少实现了零个方法。）

空接口被用来处理未知类型的值。

### 接口检查

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

## 类型

类型断言

```go
t := i.(T)
t, ok := i.(T) // if not ok, t is zero value of type T
```

类型选择

```go
// 类型选择 switch 中不允许使用 fallthrough
switch v := i.(type) {
case T:
    // v 的类型为 T
case S:
    // v 的类型为 S
default:
    // 没有匹配，v 与 i 的类型相同
}
```

类型声明

```go
// When you create a type declaration by defining a new type from an existing (non-interface) type, you don't inherit the methods defined for that existing type.
type myMutex sync.Mutex
func main() {  
    var mtx myMutex
    mtx.Lock() //error
    mtx.Unlock() //error  
}

type myLocker struct {  
    sync.Mutex
}
func main() {  
    var lock myLocker
    lock.Lock() //ok
    lock.Unlock() //ok
}
```



## Go 程

什么是协程泄露(Goroutine Leak)？

协程泄露是指协程创建后，长时间得不到释放，并且还在不断地创建新的协程，最终导致内存耗尽，程序崩溃。

常见的导致协程泄露的场景有以下几种：

- 缺少接收器，导致发送阻塞
- 缺少发送器，导致接收阻塞
- 死锁(dead lock)
- 无限循环(infinite loops)

## 信道

信道的零值为`nil`，`nil`信道接收和发送都会永远阻塞。

```go
ch := make(chan int)

ch <- v    // 将 v 发送至信道 ch。
v := <-ch  // 从 ch 接收值并赋予 v。

// 带缓冲信道，仅当信道的缓冲区填满后，向其发送数据时才会阻塞。当缓冲区为空时，接受方会阻塞。
ch := make(chan int, 100)

// 若没有值可以接收且信道已被关闭，那么在执行完之后 `ok` 会被设置为 `false`，v 为信道类型的零值
v, ok := <-ch
```

循环 `for i := range c` 会不断从信道接收值，直到它被关闭。

*注意：* 只有发送者才能`close`关闭信道，而接收者不能。向一个已经关闭的信道发送数据会引发panic。

`select` 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行。当 `select` 中的其它分支都没有准备好时，`default` 分支就会执行。

```go
for {
	select {
		case <- x:
		default:
	}
```

关闭`nil`信道或已经关闭的信道都会引发panic。

无缓冲的 channel 和 有缓冲的 channel 的区别？

对于无缓冲的 channel，发送方将阻塞该信道，直到接收方从该信道接收到数据为止，而接收方也将阻塞该信道，直到发送方将数据发送到该信道中为止。

对于有缓存的 channel，发送方在没有空插槽（缓冲区使用完）的情况下阻塞，而接收方在信道为空的情况下阻塞。

## 互斥锁（Mutex）

```go
mux sync.Mutex

mux.Lock()
defer mux.Unlock()
```

### Once 

`sync` 包通过 `Once` 类型为存在多个Go程的初始化提供了安全的机制。 多个线程可为特定的 `f` 执行 `once.Do(f)`，但只有一个会运行 `f()`，而其它调用会一直阻塞，直到 `f()` 返回。



# 三、高级

## iota

```go
const (
  azero = iota
  aone  = iota
)
const (
  info  = "processing"
  bzero = iota
  bone  = iota
)
func main() {
  fmt.Println(azero,aone) //prints: 0 1
  fmt.Println(bzero,bone) //prints: 1 2
}
```

The `iota` is really an index operator for the current line in the constant declaration block, so if the first use of `iota` is not the first line in the constant declaration block the initial value will not be zero.

## json

编码：

- 只支持键为`string`类型的`map`
- 不支持信道、`complex`和函数类型
- 不支持循环数据结构
- 指针会编码成其所指向的值（空指针则为null）
- 只编码导出的字段（字段名首字母大写）

自定义解析

实现 UnmarshalJSON 和 MarshalJSON 两个方法

**Generic representation**

https://eli.thegreenplace.net/2020/representing-json-structures-in-go/

```go
var m map[string]interface{}
if err := json.Unmarshal(jsonText, &m); err != nil {
  log.Fatal(err)
}

fruits := m["fruits"].([]interface{})
for _, f := range fruits {
  fruit := f.(map[string]interface{})
  fmt.Printf("%s -> %f\n", fruit["name"], fruit["sweetness"])
}
```



## 反射

```go
func TypeOf(i interface{}) Type
func ValueOf(i interface{}) Value
```

获取结构体标签

```go
type TagType struct { // tags
	field1 bool   "An important answer"
	field2 string "The name of the thing"
	field3 int    "How much there are"
}
func refTag(tt TagType, ix int) {
	ttType := reflect.TypeOf(tt)
	ixField := ttType.Field(ix)
	fmt.Printf("%v\n", ixField.Tag)
}
func main() {
	tt := TagType{true, "Barak Obama", 1}
	for i := 0; i < 3; i++ {
		refTag(tt, i)
	}
}

An important answer
The name of the thing
How much there are
```

例子：https://studygolang.gitbook.io/learn-go-with-tests/go-ji-chu/reflection

## 比较

- Struct values are comparable if all their fields are comparable. Two struct values are equal if their corresponding non-[blank](https://golang.org/ref/spec#Blank_identifier) fields are equal. A struct value **cannot be compared to nil**.
- If all fields in the struct are comparable, then you can compare with a known zero value of the struct: ` fmt.Println(value == Empty{})`
- Array values are comparable if values of the array element type are comparable. Two array values are equal if their corresponding elements are equal.

**Slice, map, and function values are not comparable**. However, as a special case, a slice, map, or function value may be compared to the predeclared identifier `nil`. Comparison of pointer, channel, and interface values to `nil` is also allowed and follows from the general rules above.

不可比较的类型可以用`reflect.DeepEqual()`进行比较。

If your byte slices contain secrets (e.g., cryptographic hashes, tokens, etc.) that need to be validated against user-provided data, don't use `reflect.DeepEqual()`, `bytes.Equal()`, or `bytes.Compare()` because those functions will make your application vulnerable to [**timing attacks**](http://en.wikipedia.org/wiki/Timing_attack). To avoid leaking the timing information use the functions from the 'crypto/subtle' package (e.g., `subtle.ConstantTimeCompare()`).

## new/make

Allocation

The built-in function `new` takes a type `T`, allocates storage for a [variable](https://golang.org/ref/spec#Variables) of that type at run time, and returns a value of type `*T` [pointing](https://golang.org/ref/spec#Pointer_types) to it. The variable is initialized as described in the section on [initial values](https://golang.org/ref/spec#The_zero_value).

```
new(T)
```

The expressions `new(File)` and `&File{}` are equivalent.

 `new([]int)` returns a pointer to a newly allocated, zeroed slice structure, that is, a pointer to a `nil` slice value.

Making slices, maps and channels

The built-in function `make` takes a type `T`, which must be a **slice, map or channel** type, optionally followed by a type-specific list of expressions. It returns a value of type `T` (not `*T`). The memory is initialized as described in the section on [initial values](https://golang.org/ref/spec#The_zero_value).

## debug

```go
fmt.Printf("%s", debug.Stack())
debug.PrintStack()

package runtime
// skip=0表示返回调用Caller的函数信息，1表示调用该函数的上一层函数，以此类推
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
//获取函数信息
func FuncForPC(pc uintptr) *Func
func (f *Func) Name() string
// 输出 file:line （两边为空格）到控制台时，可以点击直达对应源码
func (f *Func) FileLine(pc uintptr) (file string, line int)
```

## 分号

One consequence of the semicolon insertion rules is that you cannot put the opening brace of a control structure (`if`, `for`, `switch`, or `select`) on the next line. If you do, a semicolon will be inserted before the brace, which could cause unwanted effects. Write them like this

```go
if i < f() {
    g()
}
```

not like this

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```

同时，由于没有分号， `++` and `--` are statements not expressions. Thus if you want to run multiple variables in a `for` you should use parallel assignment (although that precludes `++` and `--`).

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

## Context/WaitGroup

```go
func Rpc(ctx context.Context, url string) error {
    result := make(chan int)
    err := make(chan error)

    go func() {
        // 进行RPC调用，并且返回是否成功，成功通过result传递成功信息，错误通过error传递错误信息
        isSuccess := true
        if isSuccess {
            result <- 1
        } else {
            err <- errors.New("some error happen")
        }
    }()

    select {
        case <- ctx.Done():
            // 其他RPC调用调用失败
            return ctx.Err()
        case e := <- err:
            // 本RPC调用失败，返回错误信息
            return e
        case <- result:
            // 本RPC调用成功，不返回错误信息
            return nil
    }
}


func main() {
    ctx, cancel := context.WithCancel(context.Background())
    wg := sync.WaitGroup{}
    wg.Add(1)
    go func(){
        defer wg.Done()
        err := Rpc(ctx, "http://rpc_1_url")
        if err != nil {
            cancel()
        }
    }()

    wg.Add(1)
    go func(){
        defer wg.Done()
        err := Rpc(ctx, "http://rpc_2_url")
        if err != nil {
            cancel()
        }
    }()

    wg.Add(1)
    go func(){
        defer wg.Done()
        err := Rpc(ctx, "http://rpc_3_url")
        if err != nil {
            cancel()
        }
    }()

    wg.Wait()
}
```

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err()) // prints "context deadline exceeded"
    }
}
```

## GC

通过调用 `runtime.GC()` 函数可以显式的触发 GC。

在一个对象 obj 被从内存移除前执行一些特殊操作：`runtime.SetFinalizer(obj, func(obj *typeObj))`

最常见的垃圾回收算法有标记清除(Mark-Sweep) 和引用计数(Reference Count)，Go 语言采用的是标记清除算法。并在此基础上使用了三色标记法和写屏障技术，提高了效率。

标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

- 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
- 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。

标记清除算法的一大问题是在标记期间，需要暂停程序（Stop the world，STW），标记结束之后，用户程序才可以继续执行。为了能够异步执行，减少 STW 的时间，Go 语言采用了三色标记法。

三色标记算法将程序中的对象分成白色、黑色和灰色三类。

- 白色：不确定对象。
- 灰色：存活对象，子对象待处理。
- 黑色：存活对象。

标记开始时，所有对象加入白色集合（这一步需 STW ）。首先将根对象标记为灰色，加入灰色集合，垃圾搜集器取出一个灰色对象，将其标记为黑色，并将其指向的对象标记为灰色，加入灰色集合。重复这个过程，直到灰色集合为空为止，标记阶段结束。那么白色对象即可需要清理的对象，而黑色对象均为根可达的对象，不能被清理。

三色标记法因为多了一个白色的状态来存放不确定对象，所以后续的标记阶段可以并发地执行。当然并发执行的代价是可能会造成一些遗漏，因为那些早先被标记为黑色的对象可能目前已经是不可达的了。所以三色标记法是一个 false negative（假阴性）的算法。

三色标记法并发执行仍存在一个问题，即在 GC 过程中，对象指针发生了改变。比如下面的例子：

```
A (黑) -> B (灰) -> C (白) -> D (白)
```

正常情况下，D 对象最终会被标记为黑色，不应被回收。但在标记和用户程序并发执行过程中，用户程序删除了 C 对 D 的引用，而 A 获得了 D 的引用。标记继续进行，D 就没有机会被标记为黑色了（A 已经处理过，这一轮不会再被处理）。

```
A (黑) -> B (灰) -> C (白) 
  ↓
 D (白)
```

为了解决这个问题，Go 使用了内存屏障技术，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，类似于一个钩子。垃圾收集器使用了写屏障（Write Barrier）技术，当对象新增或更新时，会将其着色为灰色。这样即使与用户程序并发执行，对象的引用发生改变时，垃圾收集器也能正确处理了。

一次完整的 GC 分为四个阶段：

- 1）标记准备(Mark Setup，需 STW)，打开写屏障(Write Barrier)
- 2）使用三色标记法标记（Marking, 并发）
- 3）标记结束(Mark Termination，需 STW)，关闭写屏障。
- 4）清理(Sweeping, 并发)

GC 触发时机：

- 到达堆阈值：默认情况下，它将在堆大小加倍时运行，可通过 GOGC 来设定更高阈值（不建议变更此配置）
- 到达时间阈值：每两分钟会强制启动一次 GC 循环

## 面向对象

Go 没有类，而是松耦合的类型、方法对接口的实现。

OO 语言最重要的三个方面分别是：封装，继承和多态，在 Go 中它们是怎样表现的呢？

- 封装（数据隐藏）：和别的 OO 语言有 4 个或更多的访问层次相比，Go 把它简化为了 2 层（参见 4.2 节的可见性规则）:

  1）包范围内的：通过标识符首字母小写，`对象` 只在它所在的包内可见

  2）可导出的：通过标识符首字母大写，`对象` 对所在包以外也可见

- 继承：用组合实现：内嵌一个（或多个）包含想要的行为（字段和方法）的类型；多重继承可以通过内嵌多个类型实现
- 多态：用接口实现：某个类型的实例可以赋给它所实现的任意接口类型的变量。类型和接口是松耦合的，并且多重继承可以通过实现多个接口实现。Go 接口不是 Java 和 C# 接口的变体，而且接口间是不相关的，并且是大规模编程和可适应的演进型设计的关键。

## GPM

goroutine是Go语言实现的用户态线程，主要用来解决操作系统线程太“重”的问题，所谓的太重，主要表现在以下两个方面：

- 创建和切换太重：操作系统线程的创建和切换都需要进入内核，而进入内核所消耗的性能代价比较高，开销较大；
- 内存使用太重：一方面，为了尽量避免极端情况下操作系统线程栈的溢出，内核在创建操作系统线程时默认会为其分配一个较大的栈内存（虚拟地址空间，内核并不会一开始就分配这么多的物理内存），然而在绝大多数情况下，系统线程远远用不了这么多内存，这导致了浪费；另一方面，栈内存空间一旦创建和初始化完成之后其大小就不能再有变化，这决定了在某些特殊场景下系统线程栈还是有溢出的风险。

而相对的，用户态的goroutine则轻量得多：

- goroutine是用户态线程，其创建和切换都在用户代码中完成而无需进入操作系统内核，所以其开销要远远小于系统线程的创建和切换；
- goroutine启动时默认栈大小只有2k，这在多数情况下已经够用了，即使不够用，goroutine的栈也会自动扩大，同时，如果栈太大了过于浪费它还能自动收缩，这样既没有栈溢出的风险，也不会造成栈内存空间的大量浪费。

CSP，全称Communicating Sequential Processes，意为通讯顺序进程，它的核心观念是将两个并发执行的实体通过通道channel连接起来，所有的消息都通过channel传输。 『 Don't communicate by sharing memory; share memory by communicating 』

GPM代表了三个角色，分别是Goroutine、Processor、Machine。

- Goroutine：它是一个待执行的任务，对应一个结构体g，结构体里保存了goroutine的堆栈信息
- Machine：表示操作系统的线程，它由操作系统的调度器调度和管理，最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行
- Processor：表示处理器，负责Machine与Goroutine的连接，它可以被看做运行在线程上的本地调度器，能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器`P`的调度，每一个内核线程都能够执行多个 `goroutine`，它能在`goroutine` 进行一些 `I/O` 操作时及时切换，提高线程的利用率。
  - 处理器的数量也是默认按照GOMAXPROCS来设置的，与线程的数量一一对应。
  - 对应结构体存储了处理器的待运行队列，队列中存储的是待执行的Goroutine列表。

此外在调度器里还有一个全局等待队列，当所有P本地的等待队列被占满后，新创建的`goroutine`会进入全局等待队列。`P`的本地队列为空后，`M`也会从全局队列中拿一批待执行的`goroutine`放到`P`本地的等待队列中。

通过以下两个机制提高线程的复用：

1. work stealing机制，当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。
2. hand off机制，当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。

Golang 调度是非抢占式多任务处理，由协程主动交出控制权。遇到如下条件时，才有可能交出控制权

- I/O,select
- channel
- 等待锁
- 函数调用（是一个切换的机会，是否会切换由调度器决定）
- runtime.Gosched()

## 逃逸

Go 语言的局部变量分配在栈上还是堆上？

由编译器决定。Go 语言编译器会自动决定把一个变量放在栈还是放在堆，编译器会做逃逸分析(escape analysis)，当发现变量的作用域没有超出函数范围，就可以在栈上，反之则必须分配在堆上。

函数返回局部变量的指针是否安全？

这在 Go 中是安全的，Go 编译器将会对每个局部变量进行逃逸分析。如果发现局部变量的作用域超出该函数，则不会将内存分配在栈上，而是分配在堆上。

可以通过 `go build -gcflags=-m` 查看逃逸的情况

**逃逸常见的情况：**

- 指针逃逸：返回局部变量的地址
- 栈空间不足
- 动态类型逃逸：如 fmt.Sprintf,json.Marshel 等接受变量为...interface{}函数的调用，会导致传入的变量逃逸。
- 闭包引用
- **发送指针或带有指针的值到 channel 中。** 在编译时，是没有办法知道哪个 goroutine 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。
- **在一个切片上存储指针或带指针的值。** 一个典型的例子就是 []*string 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
- **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
- **在 interface 类型上调用方法。** 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配。

## test

```go
func TestXxx(t *testing.T)
t.Fail()
t.Errorf
t.Run(string, func(t *testing.T){})
t.Helper()

func ExampleXxx() {
	...
	// Output: ...
}

// go test -bench=.
func BenchmarkXxx(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Xxx()
    }
}

// go test -cover
```

```go
func TestArea(t *testing.T) {

    areaTests := []struct {
        name    string
        shape   Shape
        hasArea float64
    }{
        {name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
        {name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
        {name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
    }

    for _, tt := range areaTests {
        // using tt.name from the case to use it as the `t.Run` test name
        t.Run(tt.name, func(t *testing.T) {
            got := tt.shape.Area()
            if got != tt.hasArea {
                t.Errorf("%#v got %.2f want %.2f", tt.shape, got, tt.hasArea)
            }
        })

    }

}
```

```go
const (
    ErrNotFound   = DictionaryErr("could not find the word you were looking for")
    ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
    return string(e)
}
```

`net/http/httptest`

```go
httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}))
```

