# Golang 比较

### 比较

Comparison operators compare two operands and yield an untyped boolean value.
> 比较运算符比较两个操作数，并产生无类型的布尔值。

```
==    equal
!=    not equal
<     less
<=    less or equal
>     greater
>=    greater or equal
```

In any comparison, the first operand must be assignable to the type of the second operand, or vice versa.
> 在任何比较中，第一个操作数必须可分配给第二个操作数的类型，反之亦然。

The equality operators == and != apply to operands that are comparable. The ordering operators <, <=, >, and >= apply to operands that are ordered. These terms and the result of the comparisons are defined as follows:
> 等号运算符 == 和 != 适用于可比较的操作数。 排序运算符 <、<=、> 和 >= 适用于已排序的操作数。 这些术语和比较结果定义如下：

* Boolean values are comparable. Two boolean values are equal if they are either both true or both false.
> 布尔可比较

* Integer values are comparable and ordered, in the usual way.
> int 可比较且有序

* Floating-point values are comparable and ordered, as defined by the IEEE-754 standard.
> 浮点数可比较且有序

* Complex values are comparable. Two complex values u and v are equal if both real(u) == real(v) and imag(u) == imag(v).
> 复数可进行 == 和 != 运算，只有虚部和实部均相等，才是相等的复数

* String values are comparable and ordered, lexically byte-wise.
> 字符串值是可比较的且按字节顺序排序

* Pointer values are comparable. Two pointer values are equal if they point to the same variable or if both have value nil. Pointers to distinct zero-size variables may or may not be equal.
> 指针值是可比较的。必须指向的是相同的数据类型或同一个结构体(底层相同不可以)。可以与 nil 比较。

* Channel values are comparable. Two channel values are equal if they were created by the same call to make or if both have value nil.
> channel 是可比较的。必须是声明的是相同的数据类型

* Interface values are comparable. Two interface values are equal if they have identical dynamic types and equal dynamic values or if both have value nil.
> 接口值是可比较的。 如果两个接口值具有相同的动态类型和相等的动态值，或者两个接口值都为nil，则它们相等。

* A value x of non-interface type X and a value t of interface type T are comparable when values of type X are comparable and X implements T. They are equal if t's dynamic type is identical to X and t's dynamic value is equal to x.
> 当类型X的值可比较且X实现T时，非接口类型X的值x和接口类型T的值t可比较。如果t的动态类型等于X并且t的动态值等于x，则它们相等。 。

* Struct values are comparable if all their fields are comparable. Two struct values are equal if their corresponding non-blank fields are equal.
> 如果结构的所有字段都是可比较的，则它们的值是可比较的，不能包含 切片，字典和函数值。

* Array values are comparable if values of the array element type are comparable. Two array values are equal if their corresponding elements are equal.
> 如果数组元素类型的值可比较，则数组值可比较。 如果两个数组的对应元素相等，则它们相等。

A comparison of two interface values with identical dynamic types causes a run-time panic if values of that type are not comparable. This behavior applies not only to direct interface value comparisons but also when comparing arrays of interface values or structs with interface-valued fields.
> 如果动态类型相同的两个接口值不具有可比性，则将它们进行比较会导致运行时恐慌。 此行为不仅适用于直接接口值比较，而且适用于将接口值或结构的数组与接口值字段进行比较。

Slice, map, and function values are not comparable. However, as a special case, a slice, map, or function value may be compared to the predeclared identifier nil. Comparison of pointer, channel, and interface values to nil is also allowed and follows from the general rules above.
> 切片，字典和函数值不可比较。 但是，在特殊情况下，可以将切片，映射或函数值与预定义的标识符nil进行比较。 还允许将指针，通道和接口值与nil进行比较，并遵循上面的一般规则。

**相同结构体不同实例比较**

```
type T1 struct {
    Name  string
    Map  map[string]string
}

func main() {
    t1 := T1{
        Name:  "yxc",
        Map:  make(map[string]string, 0),
    }
    t2 := T1{
        Name:  "yxc1",
        Map:  make(map[string]string, 0),
    }
    // 报错 实例不能比较 Invalid operation: t1 == t2 (operator == not defined on T1)
    //fmt.Println(t1 == t2)
    // 指针可以比较
    fmt.Println(&t1 == &t2) // false
}
```

相同结构体的不同实例比较时，结构体包含不可比较对象时，实例不可以比较，实例的指针可以比较。


**不同结构体不同实例比较**

```
type T1 struct {
	Name  string
	Age   int
}

type T2 struct {
	Name  string
	Age   int
}


func main() {
	t1 := T1{
		Name:  "yxc",
		Age:  10,
	}
	t2 := T2{
		Name:  "yxc",
		Age:  10,
	}
	// invalid operation: t1 == t2 (mismatched types T1 and T2)
	//fmt.Println(t1 == t2)
	// invalid operation: &t1 == &t2 (mismatched types *T1 and *T2)
	//fmt.Println(&t1 == &t2)
	
	t3 := (T1)(t2)
	fmt.Println(t1 == t3)
}
```

不同结构体的实例不可比较，可以强制转换进行比较（结构体包含不可比较成员，强制转换后也不可比较）

**扩展**

map 中的 key 值必须为可比较类型。结构体 struct 想作为 map 中的 key 值，不能包含不可比较成员。

**参考**

[可比较对象](https://golang.org/ref/spec#Comparison_operators)






