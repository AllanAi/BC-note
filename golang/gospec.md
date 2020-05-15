https://golang.org/ref/spec

## Types

### Slice types

A new, initialized slice value for a given element type `T` is made using the built-in function [`make`](https://golang.org/ref/spec#Making_slices_maps_and_channels), which takes a slice type and parameters specifying the length and optionally the capacity. A slice created with `make` always allocates a new, hidden array to which the returned slice value refers. That is, executing

```go
make([]T, length, capacity)
```

produces the same slice as allocating an array and [slicing](https://golang.org/ref/spec#Slice_expressions) it, so these two expressions are equivalent:

```go
make([]int, 50, 100)
new([100]int)[0:50]
```

### Type identity

- Two array types are identical if they have identical element types and the same array length.
- Two slice types are identical if they have identical element types.
- Two struct types are identical if they have the same sequence of fields, and if corresponding fields have the same names, and identical types, and identical tags. [Non-exported](https://golang.org/ref/spec#Exported_identifiers) field names from different packages are always different.
- Two pointer types are identical if they have identical base types.
- Two function types are identical if they have the same number of parameters and result values, corresponding parameter and result types are identical, and either both functions are variadic or neither is. Parameter and result names are not required to match.
- Two interface types are identical if they have the same set of methods with the same names and identical function types. [Non-exported](https://golang.org/ref/spec#Exported_identifiers) method names from different packages are always different. The order of the methods is irrelevant.
- Two map types are identical if they have identical key and element types.
- Two channel types are identical if they have identical element types and the same direction.

### Type declarations

#### Alias declarations

An alias declaration binds an identifier to the given type.

```
AliasDecl = identifier "=" Type .
```

Within the [scope](https://golang.org/ref/spec#Declarations_and_scope) of the identifier, it serves as an *alias* for the type.

```go
type (
	nodeList = []*Node  // nodeList and []*Node are identical types
	Polar    = polar    // Polar and polar denote identical types
)
```

#### Type definitions

The new type is called a *defined type*. It is [different](https://golang.org/ref/spec#Type_identity) from any other type, including the type it is created from.

```go
type (
	Point struct{ x, y float64 }  // Point and struct{ x, y float64 } are different types
	polar Point                   // polar and Point denote different types
)
```

## Expressions

### Index expressions

For `a` of [pointer](https://golang.org/ref/spec#Pointer_types) to array type:

- `a[x]` is shorthand for `(*a)[x]`

For `a` of [string type](https://golang.org/ref/spec#String_types):

- `a[x]` is the non-constant byte value at index `x` and the type of `a[x]` is `byte`
- `a[x]` may not be assigned to

For `a` of [map type](https://golang.org/ref/spec#Map_types) `M`:

- if the map is `nil` or does not contain such an entry, `a[x]` is the [zero value](https://golang.org/ref/spec#The_zero_value) for the element type of `M`

### Slice expressions

#### Full slice expressions

For an array, pointer to array, or slice `a` (but not a string), the primary expression

```
a[low : high : max]
```

constructs a slice of the same type, and with the same length and elements as the simple slice expression `a[low : high]`. Additionally, it controls the resulting slice's capacity by setting it to `max - low`. Only the first index may be omitted; it defaults to 0. After slicing the array `a`

```go
a := [5]int{1, 2, 3, 4, 5}
t := a[1:3:5]
```

the slice `t` has type `[]int`, length 2, capacity 4, and elements

```
t[0] == 2
t[1] == 3
```

### Passing arguments to `...` parameters

If `f` is [variadic](https://golang.org/ref/spec#Function_types) with a final parameter `p` of type `...T`, then within `f` the type of `p` is equivalent to type `[]T`. If `f` is invoked with no actual arguments for `p`, the value passed to `p` is `nil`. Otherwise, the value passed is a new slice of type `[]T` with a new underlying array whose successive elements are the actual arguments, which all must be [assignable](https://golang.org/ref/spec#Assignability) to `T`. 

If the final argument is assignable to a slice type `[]T`, it is passed unchanged as the value for a `...T` parameter if the argument is followed by `...`. In this case no new slice is created.

Given the slice `s` and call

```go
s := []string{"James", "Jasmine"}
Greeting("goodbye:", s...)
```

within `Greeting`, `who` will have the same value as `s` with the same underlying array.

### Comparison operators

- Struct values are comparable if all their fields are comparable. Two struct values are equal if their corresponding non-[blank](https://golang.org/ref/spec#Blank_identifier) fields are equal.
- Array values are comparable if values of the array element type are comparable. Two array values are equal if their corresponding elements are equal.

Slice, map, and function values are not comparable. However, as a special case, a slice, map, or function value may be compared to the predeclared identifier `nil`. Comparison of pointer, channel, and interface values to `nil` is also allowed and follows from the general rules above.

### Receive operator

Receiving from a `nil` channel blocks forever. A receive operation on a [closed](https://golang.org/ref/spec#Close) channel can always proceed immediately, yielding the element type's [zero value](https://golang.org/ref/spec#The_zero_value) after any previously sent values have been received.

### Conversions

```
Conversion = Type "(" Expression [ "," ] ")" .
```

If the type starts with the operator `*` or `<-`, or if the type starts with the keyword `func` and has no result list, it must be parenthesized when necessary to avoid ambiguity:

```go
*Point(p)        // same as *(Point(p))
(*Point)(p)      // p is converted to *Point
<-chan int(c)    // same as <-(chan int(c))
(<-chan int)(c)  // c is converted to <-chan int
func()(x)        // function signature func() x
(func())(x)      // x is converted to func()
(func() int)(x)  // x is converted to func() int
func() int(x)    // x is converted to func() int (unambiguous)
```

A [constant](https://golang.org/ref/spec#Constants) value `x` can be converted to type `T` if `x` is [representable](https://golang.org/ref/spec#Representability) by a value of `T`. As a special case, an integer constant `x` can be explicitly converted to a [string type](https://golang.org/ref/spec#String_types) using the [same rule](https://golang.org/ref/spec#Conversions_to_and_from_a_string_type) as for non-constant `x`.

```go
string([]byte{'a'})      // not a constant: []byte{'a'} is not a constant
(*int)(nil)              // not a constant: nil is not a constant, *int is not a boolean, numeric, or string type
int(1.2)                 // illegal: 1.2 cannot be represented as an int
string(65.0)             // illegal: 65.0 is not an integer constant
```

A non-constant value `x` can be converted to type `T` in any of these cases:

- `x` is [assignable](https://golang.org/ref/spec#Assignability) to `T`.
- ignoring struct tags (see below), `x`'s type and `T` have [identical](https://golang.org/ref/spec#Type_identity) [underlying types](https://golang.org/ref/spec#Types).
- ignoring struct tags (see below), `x`'s type and `T` are pointer types that are not [defined types](https://golang.org/ref/spec#Type_definitions), and their pointer base types have identical underlying types.
- `x`'s type and `T` are both integer or floating point types.
- `x`'s type and `T` are both complex types.
- `x` is an integer or a slice of bytes or runes and `T` is a string type.
- `x` is a string and `T` is a slice of bytes or runes.

#### Conversions to and from a string type

 Converting a signed or unsigned integer value to a string type yields a string containing the UTF-8 representation of the integer. Values outside the range of valid Unicode code points are converted to `"\uFFFD"`.

```go
string('a')       // "a"
string(-1)        // "\ufffd" == "\xef\xbf\xbd"
string(0xf8)      // "\u00f8" == "ø" == "\xc3\xb8"
type MyString string
MyString(0x65e5)  // "\u65e5" == "日" == "\xe6\x97\xa5"
```

### Constant expressions

Constant expressions are always evaluated exactly; intermediate values and the constants themselves may require precision significantly larger than supported by any predeclared type in the language. The following are legal declarations:

```go
const Huge = 1 << 100         // Huge == 1267650600228229401496703205376  (untyped integer constant)
const Four int8 = Huge >> 98  // Four == 4                                (type int8)
```

The values of *typed* constants must always be accurately [representable](https://golang.org/ref/spec#Representability) by values of the constant type. The following constant expressions are illegal:

```go
uint(-1)     // -1 cannot be represented as a uint
int(3.14)    // 3.14 cannot be represented as an int
int64(Huge)  // 1267650600228229401496703205376 cannot be represented as an int64
Four * 300   // operand 300 cannot be represented as an int8 (type of Four)
Four * 100   // product 400 cannot be represented as an int8 (type of Four)
```

The mask used by the unary bitwise complement operator `^` matches the rule for non-constants: the mask is all 1s for unsigned constants and -1 for signed and untyped constants.

```go
^1         // untyped integer constant, equal to -2
uint8(^1)  // illegal: same as uint8(-2), -2 cannot be represented as a uint8
^uint8(1)  // typed uint8 constant, same as 0xFF ^ uint8(1) = uint8(0xFE)
int8(^1)   // same as int8(-2)
^int8(1)   // same as -1 ^ int8(1) = -2
```

### Order of evaluation

For example, in the (function-local) assignment

```go
y[f()], ok = g(h(), i()+x[j()], <-c), k()
```

the function calls and communication happen in the order `f()`, `h()`, `i()`, `j()`, `<-c`, `g()`, and `k()`. However, the order of those events compared to the evaluation and indexing of `x` and the evaluation of `y` is not specified.

```go
a := 1
f := func() int { a++; return a }
x := []int{a, f()}            // x may be [1, 2] or [2, 2]: evaluation order between a and f() is not specified
m := map[int]int{a: 1, a: 2}  // m may be {2: 1} or {2: 2}: evaluation order between the two map assignments is not specified
n := map[int]int{a: f()}      // n may be {2: 3} or {3: 3}: evaluation order between the key and the value is not specified
```

At package level, initialization dependencies override the left-to-right rule for individual initialization expressions, but not for operands within each expression:

```go
var a, b, c = f() + v(), g(), sqr(u()) + v()

func f() int        { return c }
func g() int        { return a }
func sqr(x int) int { return x*x }

// functions u and v are independent of all other variables and functions
```

The function calls happen in the order `u()`, `sqr()`, `v()`, `f()`, `v()`, and `g()`.

## Statements

### Send statements

A send on a closed channel proceeds by causing a [run-time panic](https://golang.org/ref/spec#Run_time_panics). A send on a `nil` channel blocks forever.

### Assignments

- If an untyped constant is assigned to a variable of interface type or the blank identifier, the constant is first implicitly [converted](https://golang.org/ref/spec#Conversions) to its [default type](https://golang.org/ref/spec#Constants).

### Switch statements

#### Expression switches

The switch expression may be preceded by a simple statement, which executes before the expression is evaluated.

```go
switch tag {
default: s3()
case 0, 1, 2, 3: s1()
case 4, 5, 6, 7: s2()
}

switch x := f(); {  // missing switch expression means "true"
case x < 0: return -x
default: return x
}
```

#### Type switches

The "fallthrough" statement is not permitted in a type switch.

### For statements

#### For statements with `range` clause

1. For a `nil` slice, the number of iterations is 0.
2. For a string value, the "range" clause iterates over the Unicode code points in the string starting at byte index 0. On successive iterations, the index value will be the index of the first byte of successive UTF-8-encoded code points in the string, and the second value, of type `rune`, will be the value of the corresponding code point. If the iteration encounters an invalid UTF-8 sequence, the second value will be `0xFFFD`, the Unicode replacement character, and the next iteration will advance a single byte in the string.
3. If the map is `nil`, the number of iterations is 0.
4. For channels, the iteration values produced are the successive values sent on the channel until the channel is [closed](https://golang.org/ref/spec#Close). If the channel is `nil`, the range expression blocks forever.

### Select statements

- If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection. Otherwise, if there is a default case, that case is chosen. If there is no default case, the "select" statement blocks until at least one of the communications can proceed.

### Return statements

```go
func (devnull) Write(p []byte) (n int, _ error) {
	n = len(p)
	return
}
```

Regardless of how they are declared, all the result values are initialized to the [zero values](https://golang.org/ref/spec#The_zero_value) for their type upon entry to the function. A "return" statement that specifies results sets the result parameters before any deferred functions are executed.

Implementation restriction: A compiler may disallow an empty expression list in a "return" statement if a different entity (constant, type, or variable) with the same name as a result parameter is in [scope](https://golang.org/ref/spec#Declarations_and_scope) at the place of the return.

```go
func f(n int) (res int, err error) {
	if _, err := f(n-1); err != nil {
		return  // invalid return statement: err is shadowed
	}
	return
}
```

### Break statements

A "break" statement terminates execution of the innermost ["for"](https://golang.org/ref/spec#For_statements), ["switch"](https://golang.org/ref/spec#Switch_statements), or ["select"](https://golang.org/ref/spec#Select_statements) statement within the same function.

### Continue statements

A "continue" statement begins the next iteration of the innermost ["for" loop](https://golang.org/ref/spec#For_statements) at its post statement. 

### Defer statements

For instance, if the deferred function is a [function literal](https://golang.org/ref/spec#Function_literals) and the surrounding function has [named result parameters](https://golang.org/ref/spec#Function_types) that are in scope within the literal, the deferred function may access and modify the result parameters before they are returned. If the deferred function has any return values, they are discarded when the function completes. (See also the section on [handling panics](https://golang.org/ref/spec#Handling_panics).)

```go
// prints 3 2 1 0 before surrounding function returns
for i := 0; i <= 3; i++ {
	defer fmt.Print(i)
}

// f returns 42
func f() (result int) {
	defer func() {
		// result is accessed after it was set to 6 by the return statement
		result *= 7
	}()
	return 6
}
```

## Built-in functions

### Close

For a channel `c`, the built-in function `close(c)` records that no more values will be sent on the channel. It is an error if `c` is a receive-only channel. Sending to or closing a closed channel causes a [run-time panic](https://golang.org/ref/spec#Run_time_panics). Closing the nil channel also causes a [run-time panic](https://golang.org/ref/spec#Run_time_panics). After calling `close`, and after any previously sent values have been received, receive operations will return the zero value for the channel's type without blocking. The multi-valued [receive operation](https://golang.org/ref/spec#Receive_operator) returns a received value along with an indication of whether the channel is closed.

### Allocation

The built-in function `new` takes a type `T`, allocates storage for a [variable](https://golang.org/ref/spec#Variables) of that type at run time, and returns a value of type `*T` [pointing](https://golang.org/ref/spec#Pointer_types) to it. The variable is initialized as described in the section on [initial values](https://golang.org/ref/spec#The_zero_value).

```
new(T)
```

### Making slices, maps and channels

The built-in function `make` takes a type `T`, which must be a slice, map or channel type, optionally followed by a type-specific list of expressions. It returns a value of type `T` (not `*T`). The memory is initialized as described in the section on [initial values](https://golang.org/ref/spec#The_zero_value).

### Appending to and copying slices

As a special case, `append` also accepts a first argument assignable to type `[]byte` with a second argument of string type followed by `...`. This form appends the bytes of the string.

```go
s0 := []int{0, 0}
s1 := append(s0, 2)                // append a single element     s1 == []int{0, 0, 2}
s2 := append(s1, 3, 5, 7)          // append multiple elements    s2 == []int{0, 0, 2, 3, 5, 7}
s3 := append(s2, s0...)            // append a slice              s3 == []int{0, 0, 2, 3, 5, 7, 0, 0}
s4 := append(s3[3:6], s3[2:]...)   // append overlapping slice    s4 == []int{3, 5, 7, 2, 3, 5, 7, 0, 0}

var b []byte
b = append(b, "bar"...)            // append string contents      b == []byte{'b', 'a', 'r' }
```

As a special case, `copy` also accepts a destination argument assignable to type `[]byte` with a source argument of a string type. This form copies the bytes from the string into the byte slice.

```
copy(dst, src []T) int
copy(dst []byte, src string) int
```

Examples:

```go
var a = [...]int{0, 1, 2, 3, 4, 5, 6, 7}
var s = make([]int, 6)
var b = make([]byte, 5)
n1 := copy(s, a[0:])            // n1 == 6, s == []int{0, 1, 2, 3, 4, 5}
n2 := copy(s, s[2:])            // n2 == 4, s == []int{2, 3, 4, 5, 4, 5}
n3 := copy(b, "Hello, World!")  // n3 == 5, b == []byte("Hello")
```

### Handling panics

The return value of `recover` is `nil` if any of the following conditions holds:

- `panic`'s argument was `nil`;
- the goroutine is not panicking;
- `recover` was not called directly by a deferred function.

## Packages

### Import declarations

```
Import declaration          Local name of Sin

import   "lib/math"         math.Sin
import m "lib/math"         m.Sin
import . "lib/math"         Sin
```

## Program initialization and execution

### The zero value

Each element of such a variable or value is set to the *zero value* for its type: `false` for booleans, `0` for numeric types, `""` for strings, and `nil` for pointers, functions, interfaces, slices, channels, and maps. This initialization is done recursively, so for instance each element of an array of structs will have its fields zeroed if no value is specified.

### Package initialization

For example, given the declarations

```go
var (
	a = c + b  // == 9
	b = f()    // == 4
	c = f()    // == 5
	d = 3      // == 5 after initialization has finished
)

func f() int {
	d++
	return d
}
```

the initialization order is `d`, `b`, `c`, `a`. Note that the order of subexpressions in initialization expressions is irrelevant: `a = c + b` and `a = b + c` result in the same initialization order in this example.

