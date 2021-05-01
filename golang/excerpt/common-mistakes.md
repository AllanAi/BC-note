http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/

## String Length

- level: beginner

Let's say you are a python developer and you have the following piece of code:

```go
data = u'♥'  
print(len(data)) #prints: 1  
```

When you convert it to a similar Go code snippet you might be surprised.

```go
package main

import "fmt"

func main() {  
    data := "♥"
    fmt.Println(len(data)) //prints: 3
}
```

The built-in `len()` function returns the number of bytes instead of the number of characters like it's done for unicode strings in Python.

To get the same results in Go use the `RuneCountInString()` function from the "unicode/utf8" package.

```go
package main

import (  
    "fmt"
    "unicode/utf8"
)

func main() {  
    data := "♥"
    fmt.Println(utf8.RuneCountInString(data)) //prints: 1
```

Technically the `RuneCountInString()` function doesn't return the number of characters because a single character may span multiple runes.

```go
package main

import (  
    "fmt"
    "unicode/utf8"
)

func main() {  
    data := "é"
    fmt.Println(len(data))                    //prints: 3
    fmt.Println(utf8.RuneCountInString(data)) //prints: 2
}
```

## Bitwise NOT Operator

- level: beginner

Many languages use `~` as the unary NOT operator (aka bitwise complement), but Go reuses the XOR operator (`^`) for that.

Fails:

```go
package main

import "fmt"

func main() {  
    fmt.Println(~2) //error
}
```

Compile Error:

> /tmp/sandbox965529189/main.go:6: the bitwise complement operator is ^

Works:

```go
package main

import "fmt"

func main() {  
    var d uint8 = 2
    fmt.Printf("%08b\n",^d)
}
```

Go still uses `^` as the XOR operator, which may be confusing for some people.

If you want you can represent a unary NOT operation (e.g, `NOT 0x02`) with a binary XOR operation (e.g., `0x02 XOR 0xff`). This could explain why `^` is reused to represent unary NOT operations.

Go also has a special 'AND NOT' bitwise operator (`&^`), which adds to the NOT operator confusion. It looks like a special feature/hack to support `A AND (NOT B)` without requiring parentheses.

```go
package main

import "fmt"

func main() {  
    var a uint8 = 0x82
    var b uint8 = 0x02
    fmt.Printf("%08b [A]\n",a)
    fmt.Printf("%08b [B]\n",b)

    fmt.Printf("%08b (NOT B)\n",^b)
    fmt.Printf("%08b ^ %08b = %08b [B XOR 0xff]\n",b,0xff,b ^ 0xff)

    fmt.Printf("%08b ^ %08b = %08b [A XOR B]\n",a,b,a ^ b)
    fmt.Printf("%08b & %08b = %08b [A AND B]\n",a,b,a & b)
    fmt.Printf("%08b &^%08b = %08b [A 'AND NOT' B]\n",a,b,a &^ b)
    fmt.Printf("%08b&(^%08b)= %08b [A AND (NOT B)]\n",a,b,a & (^b))
}
```

## Closing HTTP Response Body

- level: intermediate

When you make requests using the standard http library you get a http response variable. If you don't read the response body you still need to close it. Note that you must do it for empty responses too. It's very easy to forget especially for new Go developers.

Some new Go developers do try to close the response body, but they do it in the wrong place.

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    defer resp.Body.Close()//not ok
    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
}
```

This code works for successful requests, but if the http request fails the `resp` variable might be `nil`, which will cause a runtime panic.

The most common why to close the response body is by using a `defer` call after the http response error check.

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    if err != nil {
        fmt.Println(err)
        return
    }

    defer resp.Body.Close()//ok, most of the time :-)
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
}
```

Most of the time when your http request fails the `resp` variable will be `nil` and the `err` variable will be `non-nil`. However, when you get a redirection failure both variables will be `non-nil`. This means you can still end up with a leak.

You can fix this leak by adding a call to close `non-nil` response bodies in the http response error handling block. Another option is to use one `defer` call to close response bodies for all failed and successful requests.

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    if resp != nil {
        defer resp.Body.Close()
    }

    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
}
```

The orignal implementation for `resp.Body.Close()` also reads and discards the remaining response body data. This ensured that the http connection could be reused for another request if the keepalive http connection behavior is enabled. The latest http client behavior is different. Now it's your responsibility to read and discard the remaining response data. If you don't do it the http connection might be closed instead of being reused. This little gotcha is supposed to be documented in Go 1.5.

If reusing the http connection is important for your application you might need to add something like this at the end of your response processing logic:

```go
_, err = io.Copy(ioutil.Discard, resp.Body)  
```

It will be necessary if you don't read the entire response body right away, which might happen if you are processing json API responses with code like this:

```go
json.NewDecoder(resp.Body).Decode(&data) 
```

## JSON Encoder Adds a Newline Character

- level: intermediate

You are writing a test for your JSON encoding function when you discover that your tests fail because you are not getting the expected value. What happened? If you are using the JSON Encoder object then you'll get an extra newline character at the end of your encoded JSON object.

```go
package main

import (
  "fmt"
  "encoding/json"
  "bytes"
)

func main() {
  data := map[string]int{"key": 1}
  
  var b bytes.Buffer
  json.NewEncoder(&b).Encode(data)

  raw,_ := json.Marshal(data)
  
  if b.String() == string(raw) {
    fmt.Println("same encoded data")
  } else {
    fmt.Printf("'%s' != '%s'\n",raw,b.String())
    //prints:
    //'{"key":1}' != '{"key":1}\n'
  }
}
```

The JSON Encoder object is designed for streaming. Streaming with JSON usually means newline delimited JSON objects and this is why the Encode method adds a newline character. This is a documented behavior, but it's commonly overlooked or forgotten.

## JSON Package Escapes Special HTML Characters in Keys and String Values

- level: intermediate

This is a documented behavior, but you have to be careful reading all of the JSON package documentation to learn about it. The `SetEscapeHTML` method description talks about the default encoding behavior for the and, less than and greater than characters.

This is a very unfortunate design decision by the Go team for a number of reasons. First, you can't disable this behavior for the `json.Marshal` calls. Second, this is a badly implemented security feature because it assumes that doing HTML encoding is sufficient to protect against XSS vulnerabilities in all web applications. There are a lot of different contexts where the data can be used and each context requires its own encoding method. And finally, it's bad because it assumes that the primary use case for JSON is a web page, which breaks the configuration libraries and the REST/HTTP APIs by default.

```go
package main

import (
  "fmt"
  "encoding/json"
  "bytes"
)

func main() {
  data := "x < y"
  
  raw,_ := json.Marshal(data)
  fmt.Println(string(raw))
  //prints: "x \u003c y" <- probably not what you expected
  
  var b1 bytes.Buffer
  json.NewEncoder(&b1).Encode(data)
  fmt.Println(b1.String())
  //prints: "x \u003c y" <- probably not what you expected
  
  var b2 bytes.Buffer
  enc := json.NewEncoder(&b2)
  enc.SetEscapeHTML(false)
  enc.Encode(data)
  fmt.Println(b2.String())
  //prints: "x < y" <- looks better
}
```

## Comparing Structs, Arrays, Slices, and Maps

- level: intermediate

You can use the equality operator, `==`, to compare struct variables if each structure field can be compared with the equality operator.

```go
package main

import "fmt"

type data struct {  
    num int
    fp float32
    complex complex64
    str string
    char rune
    yes bool
    events <-chan string
    handler interface{}
    ref *byte
    raw [10]byte
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",v1 == v2) //prints: v1 == v2: true
}
```

If any of the struct fields are not comparable then using the equality operator will result in compile time errors. Note that arrays are comparable only if their data items are comparable.

```go
package main

import "fmt"

type data struct {  
    num int                //ok
    checks [10]func() bool //not comparable
    doit func() bool       //not comparable
    m map[string] string   //not comparable
    bytes []byte           //not comparable
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",v1 == v2)
}
```

Go does provide a number of helper functions to compare variables that can't be compared using the comparison operators.

The most generic solution is to use the `DeepEqual()` function in the reflect package.

```go
package main

import (  
    "fmt"
    "reflect"
)

type data struct {  
    num int                //ok
    checks [10]func() bool //not comparable
    doit func() bool       //not comparable
    m map[string] string   //not comparable
    bytes []byte           //not comparable
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",reflect.DeepEqual(v1,v2)) //prints: v1 == v2: true

    m1 := map[string]string{"one": "a","two": "b"}
    m2 := map[string]string{"two": "b", "one": "a"}
    fmt.Println("m1 == m2:",reflect.DeepEqual(m1, m2)) //prints: m1 == m2: true

    s1 := []int{1, 2, 3}
    s2 := []int{1, 2, 3}
    fmt.Println("s1 == s2:",reflect.DeepEqual(s1, s2)) //prints: s1 == s2: true
}
```

Aside from being slow (which may or may not be a deal breaker for your application), `DeepEqual()` also has its own gotchas.

```go
package main

import (  
    "fmt"
    "reflect"
)

func main() {  
    var b1 []byte = nil
    b2 := []byte{}
    fmt.Println("b1 == b2:",reflect.DeepEqual(b1, b2)) //prints: b1 == b2: false
}
```

`DeepEqual()` doesn't consider an empty slice to be equal to a "nil" slice. This behavior is different from the behavior you get using the `bytes.Equal()` function. `bytes.Equal()` considers "nil" and empty slices to be equal.

```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    var b1 []byte = nil
    b2 := []byte{}
    fmt.Println("b1 == b2:",bytes.Equal(b1, b2)) //prints: b1 == b2: true
}
```

`DeepEqual()` isn't always perfect comparing slices.

```go
package main

import (  
    "fmt"
    "reflect"
    "encoding/json"
)

func main() {  
    var str string = "one"
    var in interface{} = "one"
    fmt.Println("str == in:",str == in,reflect.DeepEqual(str, in)) 
    //prints: str == in: true true

    v1 := []string{"one","two"}
    v2 := []interface{}{"one","two"}
    fmt.Println("v1 == v2:",reflect.DeepEqual(v1, v2)) 
    //prints: v1 == v2: false (not ok)

    data := map[string]interface{}{
        "code": 200,
        "value": []string{"one","two"},
    }
    encoded, _ := json.Marshal(data)
    var decoded map[string]interface{}
    json.Unmarshal(encoded, &decoded)
    fmt.Println("data == decoded:",reflect.DeepEqual(data, decoded)) 
    //prints: data == decoded: false (not ok)
}
```

If your byte slices (or strings) contain text data you might be tempted to use `ToUpper()` or `ToLower()` from the "bytes" and "strings" packages when you need to compare values in a case insensitive manner (before using `==`,`bytes.Equal()`, or `bytes.Compare()`). It will work for English text, but it will not work for text in many other languages. `strings.EqualFold()` and `bytes.EqualFold()` should be used instead.

If your byte slices contain secrets (e.g., cryptographic hashes, tokens, etc.) that need to be validated against user-provided data, don't use `reflect.DeepEqual()`, `bytes.Equal()`, or `bytes.Compare()` because those functions will make your application vulnerable to [**timing attacks**](http://en.wikipedia.org/wiki/Timing_attack). To avoid leaking the timing information use the functions from the 'crypto/subtle' package (e.g., `subtle.ConstantTimeCompare()`).

## Updating and Referencing Item Values in Slice, Array, and Map "range" Clauses

- level: intermediate

The data values generated in the "range" clause are copies of the actual collection elements. They are not references to the original items. This means that updating the values will not change the original data. It also means that taking the address of the values will not give you pointers to the original data.

```go
package main

import "fmt"

func main() {  
    data := []int{1,2,3}
    for _,v := range data {
        v *= 10 //original item is not changed
    }

    fmt.Println("data:",data) //prints data: [1 2 3]
}
```

If you need to update the original collection record value use the index operator to access the data.

```go
package main

import "fmt"

func main() {  
    data := []int{1,2,3}
    for i,_ := range data {
        data[i] *= 10
    }

    fmt.Println("data:",data) //prints data: [10 20 30]
}
```

If your collection holds pointer values then the rules are slightly different. You still need to use the index operator if you want the original record to point to another value, but you can update the data stored at the target location using the second value in the "for range" clause.

```go
package main

import "fmt"

func main() {  
    data := []*struct{num int} {{1},{2},{3}}

    for _,v := range data {
        v.num *= 10
    }

    fmt.Println(data[0],data[1],data[2]) //prints &{10} &{20} &{30}
}
```

## Slice Data "Corruption"

- level: intermediate

Let's say you need to rewrite a path (stored in a slice). You reslice the path to reference each directory modifying the first folder name and then you combine the names to create a new path.

```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path,'/')
    dir1 := path[:sepIndex]
    dir2 := path[sepIndex+1:]
    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

    dir1 = append(dir1,"suffix"...)
    path = bytes.Join([][]byte{dir1,dir2},[]byte{'/'})

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => uffixBBBB (not ok)

    fmt.Println("new path =>",string(path))
}
```

It didn't work as you expected. Instead of "AAAAsuffix/BBBBBBBBB" you ended up with "AAAAsuffix/uffixBBBB". It happened because both directory slices referenced the same underlying array data from the original path slice. This means that the original path is also modified. Depending on your application this might be a problem too.

This problem can fixed by allocating new slices and copying the data you need. Another option is to use the full slice expression.

```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path,'/')
    dir1 := path[:sepIndex:sepIndex] //full slice expression
    dir2 := path[sepIndex+1:]
    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

    dir1 = append(dir1,"suffix"...)
    path = bytes.Join([][]byte{dir1,dir2},[]byte{'/'})

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB (ok now)

    fmt.Println("new path =>",string(path))
}
```

The extra parameter in the full slice expression controls the capacity for the new slice. Now appending to that slice will trigger a new buffer allocation instead of overwriting the data in the second slice.

## Type Declarations and Methods

- level: intermediate

When you create a type declaration by defining a new type from an existing (non-interface) type, you don't inherit the methods defined for that existing type.

Fails:

```go
package main

import "sync"

type myMutex sync.Mutex

func main() {  
    var mtx myMutex
    mtx.Lock() //error
    mtx.Unlock() //error  
}
```

Compile Errors:

> /tmp/sandbox106401185/main.go:9: mtx.Lock undefined (type myMutex has no field or method Lock) /tmp/sandbox106401185/main.go:10: mtx.Unlock undefined (type myMutex has no field or method Unlock)

If you do need the methods from the original type you can define a new struct type embedding the original type as an anonymous field.

Works:

```go
package main

import "sync"

type myLocker struct {  
    sync.Mutex
}

func main() {  
    var lock myLocker
    lock.Lock() //ok
    lock.Unlock() //ok
}
```

Interface type declarations also retain their method sets.

Works:

```go
package main

import "sync"

type myLocker sync.Locker

func main() {  
    var lock myLocker = new(sync.Mutex)
    lock.Lock() //ok
    lock.Unlock() //ok
}
```

## Iteration Variables and Closures in "for" Statements

- level: intermediate

This is the most common gotcha in Go. The iteration variables in `for` statements are reused in each iteration. This means that each closure (aka function literal) created in your `for` loop will reference the same variable (and they'll get that variable's value at the time those goroutines start executing).

Incorrect:

```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        go func() {
            fmt.Println(v)
        }()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: three, three, three
}
```

The easiest solution (that doesn't require any changes to the goroutine) is to save the current iteration variable value in a local variable inside the `for` loop block.

Works:

```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        vcopy := v //
        go func() {
            fmt.Println(vcopy)
        }()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```

Another solution is to pass the current iteration variable as a parameter to the anonymous goroutine.

Works:

```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        go func(in string) {
            fmt.Println(in)
        }(v)
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```

Here's a slightly more complicated version of the trap.

Incorrect:

```go
package main

import (  
    "fmt"
    "time"
)

type field struct {  
    name string
}

func (p *field) print() {  
    fmt.Println(p.name)
}

func main() {  
    data := []field{{"one"},{"two"},{"three"}}

    for _,v := range data {
        go v.print()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: three, three, three
}
```

Works:

```go
package main

import (  
    "fmt"
    "time"
)

type field struct {  
    name string
}

func (p *field) print() {  
    fmt.Println(p.name)
}

func main() {  
    data := []field{{"one"},{"two"},{"three"}}

    for _,v := range data {
        v := v
        go v.print()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```

## Failed Type Assertions

- level: intermediate

Failed type assertions return the "zero value" for the target type used in the assertion statement. This can lead to unexpected behavior when it's mixed with variable shadowing.

Incorrect:

```go
package main

import "fmt"

func main() {  
    var data interface{} = "great"

    if data, ok := data.(int); ok {
        fmt.Println("[is an int] value =>",data)
    } else {
        fmt.Println("[not an int] value =>",data) 
        //prints: [not an int] value => 0 (not "great")
    }
}
```

Works:

```go
package main

import "fmt"

func main() {  
    var data interface{} = "great"

    if res, ok := data.(int); ok {
        fmt.Println("[is an int] value =>",res)
    } else {
        fmt.Println("[not an int] value =>",data) 
        //prints: [not an int] value => great (as expected)
    }
}
```

## Same Address for Different Zero-sized Variables

- level: intermediate

If you have two different variables shouldn't they have different addresses? Well, it's not the case with Go :-) If you have zero-sized variables they might share the exact same address in memory.

```go
package main

import (
  "fmt"
)

type data struct {
}

func main() {
  a := &data{}
  b := &data{}
  
  if a == b {
    fmt.Printf("same address - a=%p b=%p\n",a,b)
    //prints: same address - a=0x1953e4 b=0x1953e4
  }
}
```

## The First Use of iota Doesn't Always Start with Zero

- level: intermediate

It may seem like the `iota` identifyer is like an increment operator. You start a new constant declaration and the first time you use `iota` you get zero, the second time you use it you get one and so on. It's not always the case though.

```go
package main

import (
  "fmt"
)

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

## Using Pointer Receiver Methods On Value Instances

- level: advanced

It's OK to call a pointer receiver method on a value as long as the value is addressable. In other words, you don't need to have a value receiver version of the method in some cases.

Not every variable is addressable though. Map elements are not addressable. Variables referenced through interfaces are also not addressable.

```go
package main

import "fmt"

type data struct {  
    name string
}

func (p *data) print() {  
    fmt.Println("name:",p.name)
}

type printer interface {  
    print()
}

func main() {  
    d1 := data{"one"}
    d1.print() //ok

    var in printer = data{"two"} //error
    in.print()

    m := map[string]data {"x":data{"three"}}
    m["x"].print() //error
}
```

Compile Errors:

> /tmp/sandbox017696142/main.go:21: cannot use data literal (type data) as type printer in assignment: data does not implement printer (print method has pointer receiver)
> /tmp/sandbox017696142/main.go:25: cannot call pointer method on m["x"] /tmp/sandbox017696142/main.go:25: cannot take the address of m["x"]

## "nil" Interfaces and "nil" Interfaces Values

- level: advanced

This is the second most common gotcha in Go because interfaces are not pointers even though they may look like pointers. Interface variables will be "nil" only when their type and value fields are "nil".

The interface type and value fields are populated based on the type and value of the variable used to create the corresponding interface variable. This can lead to unexpected behavior when you are trying to check if an interface variable equals to "nil".

```go
package main

import "fmt"

func main() {  
    var data *byte
    var in interface{}

    fmt.Println(data,data == nil) //prints: <nil> true
    fmt.Println(in,in == nil)     //prints: <nil> true

    in = data
    fmt.Println(in,in == nil)     //prints: <nil> false
    //'data' is 'nil', but 'in' is not 'nil'
}
```

Watch out for this trap when you have a function that returns interfaces.

Incorrect:

```go
package main

import "fmt"

func main() {  
    doit := func(arg int) interface{} {
        var result *struct{} = nil

        if(arg > 0) {
            result = &struct{}{}
        }

        return result
    }

    if res := doit(-1); res != nil {
        fmt.Println("good result:",res) //prints: good result: <nil>
        //'res' is not 'nil', but its value is 'nil'
    }
}
```

Works:

```go
package main

import "fmt"

func main() {  
    doit := func(arg int) interface{} {
        var result *struct{} = nil

        if(arg > 0) {
            result = &struct{}{}
        } else {
            return nil //return an explicit 'nil'
        }

        return result
    }

    if res := doit(-1); res != nil {
        fmt.Println("good result:",res)
    } else {
        fmt.Println("bad result (res is nil)") //here as expected
    }
}
```