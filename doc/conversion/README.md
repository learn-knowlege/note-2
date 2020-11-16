# Golang 断言、强制类型转换

### 断言

所有的数据类型都是 `interface{}` 接口，断言就是针对 `interface{}接口` 做类型判断。`interface.(type)`

```
// 万物皆可传入
func any (i interface{}){
	fmt.Println(i)
}
```
**使用方式**
```
n := i.(string)
```
n为断言成功后i字符串的值，如果断言失败，会抛出panic错误

**安全的方式**
```
n,ok := i.(string)
```
断言成功后，ok=true，n为断言类型的值；当断言失败时，ok=false，n为断言类型的默认值，例如**string时为空字符串**，断言**int类型时为0**。

**配合switch语句**
```
func any(i interface{}) {
    switch t := i.(type) {
    case int:
    	fmt.Printf("type int %v", t)
    case string:
    	fmt.Printf("type string %v", t)
    case bool:
    	fmt.Printf("type bool %v", t)
    default:
    	fmt.Printf("default %v", t)
    }
}
```

### 类型转换

由于 Go语言是强类型的语言，如果不满足自动转换的条件，则必须进行强制类型转换。任意两个不相干的类型如果进行强制转换，则必须符合一定的规则。

强制类型的语法格式：
```
var a T = (T) (b)
```
使用括号将类型和要转换的变量或表达式的值括起来。

非常量类型的变量 x 可以强制转化并传递给类型 T，需要满足如下任一条件：

> x 可以直接赋值给 T 类型变量。

> x 的类型和 T 具有相同的底层类型。

```
type P1 struct {
    Name string
    Map  map[string]string
}
type P2 struct {
    Name string
    Map  map[string]string
}

func main() {
    p1 := P1{
        Name: "tw",
        Map: nil,
    }
    var p2 P2 = (P2)(p1)
    fmt.Println(p2)
}
```

> x 的类型和 T 都是未命名的指针类型，并且指针指向的类型具有相同的底层类型。

```
type Person struct {
    Name    string
    Address *struct {
    	Street string
    	City   string
    }
    // 包含不可比较字段也可以
    Map map[string]string
}

var data *struct {
    Name    string `json:"name"`
    Address *struct {
    	Street string `json:"street"`
    	City   string `json:"city"`
    } `json:"address"`
    Map map[string]string
}

func main() {
    var person = (*Person)(data)  // ignoring tags, the underlying types are identical
    fmt.Println(person)
}
```

> x 的类型和 T 都是整型，或者都是浮点型。

```
func main() {
    var (
    	f float32
    	i int   
    )
    f1 := int(f)
    i1 := float32(i)
    i2 := int64(i)
}
```

> x 的类型和 T 都是复数类型。

> x 是整数值或 []byte 类型的值， T 是 string 类型。

> x 是一个字符串，T 是 []byte 或 []rune 。

```
func main() {
    // 字符串
    s1 := string("a")
    // 整数
    s2 := string(10)

    // 浮点数
    //var f float32 = 10.2
    //s3 := string(f) //cannot convert f (type float32) to type string

    s3 := string([]byte{'h', 'e', 'l', 'l', '\xc3', '\xb8'}) //hellø

    s4 := string([]rune{0x767d, 0x9d6c, 0x7fd4})  //白鵬翔

    b1 := []byte("hellø")        //[104 101 108 108 195 184]

    r1 := []rune(string("白鵬翔")) //[30333 40300 32724]
}
```

**注意**

数值类型和 string 类型之间的相互转换可能造成值部分丢失；其他的转换仅是类型的转换，不会造成值的改变。string 和数字之间的转换可使用标准库 strconv。

Go语言没有语言机制支持指针和 interger 之间的直接转换，可以使用标准库中的 unsafe 包进行处理。

```
package main

import "unsafe"
import "fmt"

func main() {
    var a int =10
    var b *int =&a
    var c *int64 = (*int64)(unsafe.Pointer(b))
    fmt.Println(*c)
}
```
