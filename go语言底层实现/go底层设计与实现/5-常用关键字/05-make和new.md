# 5.5 make 和 new [#](#55-make-%e5%92%8c-new)

```

当我们想要在 Go 语言中初始化一个结构时，可能会用到两个不同的关键字 — `make` 和 `new`。因为它们的功能相似，所以初学者可能会对这两个关键字的作用感到困惑[1](#fn:1)，但是它们两者能够初始化的变量却有较大的不同。

* `make` 的作用是初始化内置的数据结构，也就是我们在前面提到的切片、哈希表和 Channel[2](#fn:2)；
* `new` 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针[3](#fn:3)；

我们在代码中往往都会使用如下所示的语句初始化这三类基本类型，这三个语句分别返回了不同类型的数据结构：

```go
slice := make([]int, 0, 100)
hash := make(map[int]bool, 10)
ch := make(chan int, 5)
```

1.  `slice` 是一个包含 `data`、`cap` 和 `len` 的结构体 [`reflect.SliceHeader`](https://draveness.me/golang/tree/reflect.SliceHeader)；
2.  `hash` 是一个指向 [`runtime.hmap`](https://draveness.me/golang/tree/runtime.hmap) 结构体的指针；
3.  `ch` 是一个指向 [`runtime.hchan`](https://draveness.me/golang/tree/runtime.hchan) 结构体的指针；

相比与复杂的 `make` 关键字，`new` 的功能就简单多了，它只能接收类型作为参数然后返回一个指向该类型的指针：

```go
i := new(int)

var v int
i := &v
```

上述代码片段中的两种不同初始化方法是等价的，它们都会创建一个指向 `int` 零值的指针。

![golang-make-and-new](https://gitlab.com/moqsien/go-design-implementation/-/raw/main/golang-make-and-new.png)

**图 5-14 make 和 new 初始化的类型**

接下来我们将分别介绍 `make` 和 `new` 初始化不同数据结构的过程，我们会从编译期间和运行时两个不同阶段理解这两个关键字的原理，不过由于前面的章节已经详细地分析过 `make` 的原理，所以这里会将重点放在另一个关键字 `new` 上。

## 5.5.1 make [#](#551-make)

在前面的章节中我们已经谈到过 `make` 在创建切片、哈希表和 Channel 的具体过程，所以在这一小节，我们只是会简单提及 `make` 相关的数据结构的初始化原理。

![golang-make-typecheck](https://gitlab.com/moqsien/go-design-implementation/-/raw/main/golang-make-typecheck.png)

**图 5-15 make 关键字的类型检查**

在编译期间的类型检查阶段，Go 语言会将代表 `make` 关键字的 `OMAKE` 节点根据参数类型的不同转换成了 `OMAKESLICE`、`OMAKEMAP` 和 `OMAKECHAN` 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构。

## 5.5.2 new [#](#552-new)

编译器会在中间代码生成阶段通过以下两个函数处理该关键字：

1.  [`cmd/compile/internal/gc.callnew`](https://draveness.me/golang/tree/cmd/compile/internal/gc.callnew) 会将关键字转换成 `ONEWOBJ` 类型的节点[2](#fn:2)；
2.  [`cmd/compile/internal/gc.state.expr`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.expr) 会根据申请空间的大小分两种情况处理：
    1.  如果申请的空间为 0，就会返回一个表示空指针的 `zerobase` 变量；
    2.  在遇到其他情况时会将关键字转换成 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数：

```go
func callnew(t *types.Type) *Node {
	...
	n := nod(ONEWOBJ, typename(t), nil)
	...
	return n
}

func (s *state) expr(n *Node) *ssa.Value {
	switch n.Op {
	case ONEWOBJ:
		if n.Type.Elem().Size() == 0 {
			return s.newValue1A(ssa.OpAddr, n.Type, zerobaseSym, s.sb)
		}
		typ := s.expr(n.Left)
		vv := s.rtcall(newobject, true, []*types.Type{n.Type}, typ)
		return vv[0]
	}
}
```

需要注意的是，无论是直接使用 `new`，还是使用 `var` 初始化变量，它们在编译器看来都是 `ONEW` 和 `ODCL` 节点。如果变量会逃逸到堆上，这些节点在这一阶段都会被 [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 转换成通过 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数并在堆上申请内存：

```go
func walkstmt(n *Node) *Node {
	switch n.Op {
	case ODCL:
		v := n.Left
		if v.Class() == PAUTOHEAP {
			if prealloc[v] == nil {
				prealloc[v] = callnew(v.Type)
			}
			nn := nod(OAS, v.Name.Param.Heapaddr, prealloc[v])
			nn.SetColas(true)
			nn = typecheck(nn, ctxStmt)
			return walkstmt(nn)
		}
	case ONEW:
		if n.Esc == EscNone {
			r := temp(n.Type.Elem())
			r = nod(OAS, r, nil)
			r = typecheck(r, ctxStmt)
			init.Append(r)
			r = nod(OADDR, r.Left, nil)
			r = typecheck(r, ctxExpr)
			n = r
		} else {
			n = callnew(n.Type.Elem())
		}
	}
}
```

不过这也不是绝对的，如果通过 `var` 或者 `new` 创建的变量不需要在当前作用域外生存，例如不用作为返回值返回给调用方，那么就不需要初始化在堆上。

[`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数会获取传入类型占用空间的大小，调用 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 在堆上申请一片内存空间并返回指向这片内存空间的指针：

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

[`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 函数的实现大概有 200 多行代码，我们会在后面的章节中详细分析 Go 语言的内存管理机制。

## 5.5.3 小结 [#](#553-%e5%b0%8f%e7%bb%93)

这里我们简单总结一下 Go 语言中 `make` 和 `new` 关键字的实现原理，`make` 关键字的作用是创建切片、哈希表和 Channel 等内置的数据结构，而 `new` 的作用是为类型申请一片内存空间，并返回指向这片内存的指针。

[上一节](https://github.com/moqsien/MyNotes/blob/main/go语言底层实现/go底层设计与实现/5-常用关键字/04-panic和recover.md) [下一节](https://github.com/moqsien/MyNotes/blob/main/go语言底层实现/go底层设计与实现/6-并发编程/01-上下文Context.md)

* * *

1.  Make and new [https://groups.google.com/forum/#\!topic/golang-nuts/kWXYU95XN04/discussion\%5B1-25\%5D](https://groups.google.com/forum/#!topic/golang-nuts/kWXYU95XN04/discussion%5B1-25%5D) [↩︎](#fnref:1)

2.  Allocation with make <https://golang.org/doc/effective_go.html#allocation_make> [↩︎](#fnref:2)

3.  Allocation with new <https://golang.org/doc/effective_go.html#allocation_new> [↩︎](#fnref:3)