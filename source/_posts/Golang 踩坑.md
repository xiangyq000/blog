## 1. 接口

```go
package main

type I interface {
	foo1()
	foo2()
}

type S struct {}

func (s S) foo1() {}

func (s S) foo2() {}

type T struct {}

func (t *T) foo1() {}

func (t *T) foo2() {}

type R struct {}

func (r R) foo1() {}

func (r *R) foo2() {}


func main() {

	var s S
	var si I = s
	si.foo1()

	si = &s
	si.foo2()

	var t T
	var ti I = &t
	ti.foo1()
	//ti = t 编译报错

	var r R
	var ri I = &r
	ri.foo1()
}
```

若 T 是一个具体类型，并且想在 T 上实现 接口 I：

- 如果将接口的方法绑定到 T 上，那么可以将 T 或者 *T 类型的变量赋值给 I 类型的变量。
- 如果将接口的方法绑定到 *T 上，那么只能将 *T 类型的变量赋值给 I 类型的变量。
- 如果将接口的一部分方法绑定到 T 上，一部分绑定到 *T 上，那么只能将 *T 类型的变量赋值给 I 类型的变量。