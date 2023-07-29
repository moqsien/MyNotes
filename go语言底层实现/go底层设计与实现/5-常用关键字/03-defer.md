# 5.3 defer [#](#53-defer)

```

很多现代的编程语言中都有 `defer` 关键字，Go 语言的 `defer` 会在当前函数返回前执行传入的函数，它会经常被用于关闭文件描述符、关闭数据库连接以及解锁资源。这一节会深入 Go 语言的源代码介绍 `defer` 关键字的实现原理，相信读者读完这一节会对 `defer` 的数据结构、实现以及调用过程有着更清晰的理解。

作为一个编程语言中的关键字，`defer` 的实现一定是由编译器和运行时共同完成的，不过在深入源码分析它的实现之前我们还是需要了解 `defer` 关键字的常见使用场景以及使用时的注意事项。

使用 `defer` 的最常见场景是在函数调用结束后完成一些收尾工作，例如在 `defer` 中回滚数据库的事务：

```go
func createPost(db *gorm.DB) error {
    tx := db.Begin()
    defer tx.Rollback()
    
    if err := tx.Create(&Post{Author: "Draveness"}).Error; err != nil {
        return err
    }
    
    return tx.Commit().Error
}
```

在使用数据库事务时，我们可以使用上面的代码在创建事务后就立刻调用 `Rollback` 保证事务一定会回滚。哪怕事务真的执行成功了，那么调用 `tx.Commit()` 之后再执行 `tx.Rollback()` 也不会影响已经提交的事务。

## 5.3.1 现象 [#](#531-%e7%8e%b0%e8%b1%a1)

我们在 Go 语言中使用 `defer` 时会遇到两个常见问题，这里会介绍具体的场景并分析这两个现象背后的设计原理：

* `defer` 关键字的调用时机以及多次调用 `defer` 时执行顺序是如何确定的；
* `defer` 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；

### 作用域 [#](#%e4%bd%9c%e7%94%a8%e5%9f%9f)

向 `defer` 关键字传入的函数会在函数返回之前运行。假设我们在 `for` 循环中多次调用 `defer` 关键字：

```go
func main() {
	for i := 0; i < 5; i++ {
		defer fmt.Println(i)
	}
}

$ go run main.go
4
3
2
1
0
```

运行上述代码会倒序执行传入 `defer` 关键字的所有表达式，因为最后一次调用 `defer` 时传入了 `fmt.Println(4)`，所以这段代码会优先打印 4。我们可以通过下面这个简单例子强化对 `defer` 执行时机的理解：

```go
func main() {
    {
        defer fmt.Println("defer runs")
        fmt.Println("block ends")
    }
    
    fmt.Println("main ends")
}

$ go run main.go
block ends
main ends
defer runs
```

从上述代码的输出我们会发现，`defer` 传入的函数不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用。

### 预计算参数 [#](#%e9%a2%84%e8%ae%a1%e7%ae%97%e5%8f%82%e6%95%b0)

Go 语言中所有的函数调用都是传值的，虽然 `defer` 是关键字，但是也继承了这个特性。假设我们想要计算 `main` 函数运行的时间，可能会写出以下的代码：

```go
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))
	
	time.Sleep(time.Second)
}

$ go run main.go
0s
```

然而上述代码的运行结果并不符合我们的预期，这个现象背后的原因是什么呢？经过分析，我们会发现调用 `defer` 关键字会立刻拷贝函数中引用的外部参数，所以 `time.Since(startedAt)` 的结果不是在 `main` 函数退出之前计算的，而是在 `defer` 关键字调用时计算的，最终导致上述代码输出 0s。

想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数：

```go
func main() {
	startedAt := time.Now()
	defer func() { fmt.Println(time.Since(startedAt)) }()
	
	time.Sleep(time.Second)
}

$ go run main.go
1s
```

虽然调用 `defer` 关键字时也使用值传递，但是因为拷贝的是函数指针，所以 `time.Since(startedAt)` 会在 `main` 函数返回前调用并打印出符合预期的结果。

## 5.3.2 数据结构 [#](#532-%e6%95%b0%e6%8d%ae%e7%bb%93%e6%9e%84)

在介绍 `defer` 函数的执行过程与实现原理之前，我们首先来了解一下 `defer` 关键字在 Go 语言源代码中对应的数据结构：

```go
type _defer struct {
	siz       int32
	started   bool
	openDefer bool
	sp        uintptr
	pc        uintptr
	fn        *funcval
	_panic    *_panic
	link      *_defer
}
```

[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体是延迟调用链表上的一个元素，所有的结构体都会通过 `link` 字段串联成链表。

![golang-defer-link](https://gitlab.com/moqsien/go-design-implementation/-/raw/main/golang-defer-link.png)

**图 5-10 延迟调用链表**

我们简单介绍一下 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中的几个字段：

* `siz` 是参数和结果的内存大小；
* `sp` 和 `pc` 分别代表栈指针和调用方的程序计数器；
* `fn` 是 `defer` 关键字中传入的函数；
* `_panic` 是触发延迟调用的结构体，可能为空；
* `openDefer` 表示当前 `defer` 是否经过开放编码的优化；

除了上述的这些字段之外，[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 中还包含一些垃圾回收机制使用的字段，这里为了减少理解的成本就都省去了。

## 5.3.3 执行机制 [#](#533-%e6%89%a7%e8%a1%8c%e6%9c%ba%e5%88%b6)

中间代码生成阶段的 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 会负责处理程序中的 `defer`，该函数会根据条件的不同，使用三种不同的机制处理该关键字：

```go
func (s *state) stmt(n *Node) {
	...
	switch n.Op {
	case ODEFER:
		if s.hasOpenDefers {
			s.openDeferRecord(n.Left) // 开放编码
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack // 栈分配
			}
			s.callResult(n.Left, d)
		}
	}
}
```

堆分配、栈分配和开放编码是处理 `defer` 关键字的三种方法，早期的 Go 语言会在堆上分配 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，不过该实现的性能较差，Go 语言在 1.13 中引入栈上分配的结构体，减少了 30\% 的额外开销[1](#fn:1)，并在 1.14 中引入了基于开放编码的 `defer`，使得该关键字的额外开销可以忽略不计[2](#fn:2)，我们在一节中会分别介绍三种不同类型 `defer` 的设计与实现原理。

## 5.3.4 堆上分配 [#](#534-%e5%a0%86%e4%b8%8a%e5%88%86%e9%85%8d)

根据 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 方法对 `defer` 的处理我们可以看出，堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体是默认的兜底方案，当该方案被启用时，编译器会调用 [`cmd/compile/internal/gc.state.callResult`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.callResult) 和 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)，这表示 `defer` 在编译器看来也是函数调用。

[`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会负责为所有函数和方法调用生成中间代码，它的工作包括以下内容：

1.  获取需要执行的函数名、闭包指针、代码指针和函数调用的接收方；
2.  获取栈地址并将函数或者方法的参数写入栈中；
3.  使用 [`cmd/compile/internal/gc.state.newValue1A`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.newValue1A) 以及相关函数生成函数调用的中间代码；
4.  如果当前调用的函数是 `defer`，那么会单独生成相关的结束代码块；
5.  获取函数的返回值地址并结束当前调用；

```go
func (s *state) call(n *Node, k callKind, returnResultAddr bool) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上初始化 defer 结构体
		...
	} else {
		...
		switch {
		case k == callDefer:
			aux := ssa.StaticAuxCall(deferproc, ACArgs, ACResults)
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		...
		}
		call.AuxInt = stksize
	}
	s.vars[&memVar] = call
	...
}
```

从上述代码中我们能看到，`defer` 关键字在运行期间会调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc)，这个函数接收了参数的大小和闭包所在的地址两个参数。

编译器不仅将 `defer` 关键字都转换成 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 函数，它还会通过以下三个步骤为所有调用 `defer` 的函数末尾插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的函数调用：

1.  [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 在遇到 `ODEFER` 节点时会执行 `Curfn.Func.SetHasDefer(true)` 设置当前函数的 `hasdefer` 属性；
2.  [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 会执行 `s.hasdefer = fn.Func.HasDefer()` 更新 `state` 的 `hasdefer`；
3.  [`cmd/compile/internal/gc.state.exit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.exit) 会根据 `state` 的 `hasdefer` 在函数返回之前插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的函数调用；

```go
func (s *state) exit() *ssa.Block {
	if s.hasdefer {
		...
		s.rtcall(Deferreturn, true, nil)
	}
	...
}
```

当运行时将 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 分配到堆上时，Go 语言的编译器不仅将 `defer` 转换成了 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc)，还在所有调用 `defer` 的函数结尾插入了 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)。上述两个运行时函数是 `defer` 关键字运行时机制的入口，它们分别承担了不同的工作：

* [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 负责创建新的延迟调用；
* [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 负责在函数调用结束时执行所有的延迟调用；

我们以上述两个函数为入口介绍 `defer` 关键字在运行时的执行过程与工作原理。

### 创建延迟调用 [#](#%e5%88%9b%e5%bb%ba%e5%bb%b6%e8%bf%9f%e8%b0%83%e7%94%a8)

[`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 会为 `defer` 创建一个新的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体、设置它的函数指针 `fn`、程序计数器 `pc` 和栈指针 `sp` 并将相关的参数拷贝到相邻的内存空间中：

```go
func deferproc(siz int32, fn *funcval) {
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	return0()
}
```

最后调用的 [`runtime.return0`](https://draveness.me/golang/tree/runtime.return0) 是唯一一个不会触发延迟调用的函数，它可以避免递归 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的递归调用。

[`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 中 [`runtime.newdefer`](https://draveness.me/golang/tree/runtime.newdefer) 的作用是想尽办法获得 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，这里包含三种路径：

1.  从调度器的延迟调用缓存池 `sched.deferpool` 中取出结构体并将该结构体追加到当前 Goroutine 的缓存池中；
2.  从 Goroutine 的延迟调用缓存池 `pp.deferpool` 中取出结构体；
3.  通过 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 在堆上创建一个新的结构体；

```go
func newdefer(siz int32) *_defer {
	var d *_defer
	sc := deferclass(uintptr(siz))
	gp := getg()
	if sc < uintptr(len(p{}.deferpool)) {
		pp := gp.m.p.ptr()
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
				d := sched.deferpool[sc]
				sched.deferpool[sc] = d.link
				pp.deferpool[sc] = append(pp.deferpool[sc], d)
			}
		}
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
	}
	if d == nil {
		total := roundupsize(totaldefersize(uintptr(siz)))
		d = (*_defer)(mallocgc(total, deferType, true))
	}
	d.siz = siz
	d.link = gp._defer
	gp._defer = d
	return d
}
```

无论使用哪种方式，只要获取到 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，它都会被追加到所在 Goroutine `_defer` 链表的最前面。

![golang-new-defer](https://gitlab.com/moqsien/go-design-implementation/-/raw/main/golang-new-defer.png)

**图 5-11 追加新的延迟调用**

`defer` 关键字的插入顺序是从后向前的，而 `defer` 关键字执行是从前向后的，这也是为什么后调用的 `defer` 会优先执行。

### 执行延迟调用 [#](#%e6%89%a7%e8%a1%8c%e5%bb%b6%e8%bf%9f%e8%b0%83%e7%94%a8)

[`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会从 Goroutine 的 `_defer` 链表中取出最前面的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并调用 [`runtime.jmpdefer`](https://draveness.me/golang/tree/runtime.jmpdefer) 传入需要执行的函数和参数：

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	...

	switch d.siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	gp._defer = d.link
	freedefer(d)
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

[`runtime.jmpdefer`](https://draveness.me/golang/tree/runtime.jmpdefer) 是一个用汇编语言实现的运行时函数，它的主要工作是跳转到 `defer` 所在的代码段并在执行结束之后跳转回 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)。

```go
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-8
	MOVL	fv+0(FP), DX	// fn
	MOVL	argp+4(FP), BX	// caller sp
	LEAL	-4(BX), SP	// caller sp after CALL
#ifdef GOBUILDMODE_shared
	SUBL	$16, (SP)	// return to CALL again
#else
	SUBL	$5, (SP)	// return to CALL again
#endif
	MOVL	0(DX), BX
	JMP	BX	// but first run the deferred function
```

[`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会多次判断当前 Goroutine 的 `_defer` 链表中是否有未执行的结构体，该函数只有在所有延迟函数都执行后才会返回。

## 5.3.5 栈上分配 [#](#535-%e6%a0%88%e4%b8%8a%e5%88%86%e9%85%8d)

在默认情况下，我们可以看到 Go 语言中 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体都会在堆上分配，如果我们能够将部分结构体分配到栈上就可以节约内存分配带来的额外开销。

Go 语言团队在 1.13 中对 `defer` 关键字进行了优化，当该关键字在函数体中最多执行一次时，编译期间的 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会将结构体分配到栈上并调用 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack)：

```go
func (s *state) call(n *Node, k callKind) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上创建 _defer 结构体
		t := deferstruct(stksize)
		...

		ACArgs = append(ACArgs, ssa.Param{Type: types.Types[TUINTPTR], Offset: int32(Ctxt.FixedFrameSize())})
		aux := ssa.StaticAuxCall(deferprocStack, ACArgs, ACResults) // 调用 deferprocStack
		arg0 := s.constOffPtrSP(types.Types[TUINTPTR], Ctxt.FixedFrameSize())
		s.store(types.Types[TUINTPTR], arg0, addr)
		call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		call.AuxInt = stksize
	} else {
		...
	}
	s.vars[&memVar] = call
	...
}
```

因为在编译期间我们已经创建了 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，所以在运行期间 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack) 只需要设置一些未在编译期间初始化的字段，就可以将栈上的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 追加到函数的链表上：

```go
func deferprocStack(d *_defer) {
	gp := getg()
	d.started = false
	d.heap = false // 栈上分配的 _defer
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

除了分配位置的不同，栈上分配和堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并没有本质的不同，而该方法可以适用于绝大多数的场景，与堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 相比，该方法可以将 `defer` 关键字的额外开销降低 \~30\%。

## 5.3.5 开放编码 [#](#535-%e5%bc%80%e6%94%be%e7%bc%96%e7%a0%81)

Go 语言在 1.14 中通过开放编码（Open Coded）实现 `defer` 关键字，该设计使用代码内联优化 `defer` 关键的额外开销并引入函数数据 `funcdata` 管理 `panic` 的调用[3](#fn:3)，该优化可以将 `defer` 的调用开销从 1.13 版本的 \~35ns 降低至 \~6ns 左右：

```go
With normal (stack-allocated) defers only:         35.4  ns/op
With open-coded defers:                             5.6  ns/op
Cost of function call alone (remove defer keyword): 4.4  ns/op
```

然而开放编码作为一种优化 `defer` 关键字的方法，它不是在所有的场景下都会开启的，开放编码只会在满足以下的条件时启用：

1.  函数的 `defer` 数量少于或者等于 8 个；
2.  函数的 `defer` 关键字不能在循环中执行；
3.  函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个；

初看上述几个条件可能会觉得不明所以，但是当我们深入理解基于开放编码的优化就可以明白上述限制背后的原因，除了上述几个条件之外，也有其他的条件会限制开放编码的使用，不过这些都是不太重要的细节，我们在这里也不会深究。

### 启用优化 [#](#%e5%90%af%e7%94%a8%e4%bc%98%e5%8c%96)

Go 语言会在编译期间就确定是否启用开放编码，在编译器生成中间代码之前，我们会使用 [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 修改已经生成的抽象语法树，设置函数体上的 `OpenCodedDeferDisallowed` 属性：

```go
const maxOpenDefers = 8

func walkstmt(n *Node) *Node {
	switch n.Op {
	case ODEFER:
		Curfn.Func.SetHasDefer(true)
		Curfn.Func.numDefers++
		if Curfn.Func.numDefers > maxOpenDefers {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		if n.Esc != EscNever {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		fallthrough
	...
	}
}
```

就像我们上面提到的，如果函数中 `defer` 关键字的数量多于 8 个或者 `defer` 关键字处于 `for` 循环中，那么我们在这里都会禁用开放编码优化，使用上两节提到的方法处理 `defer`。

在 SSA 中间代码生成阶段的 [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 中，我们也能够看到启用开放编码优化的其他条件，也就是返回语句的数量与 `defer` 数量的乘积需要小于 15：

```go
func buildssa(fn *Node, worker int) *ssa.Func {
	...
	s.hasOpenDefers = s.hasdefer && !s.curfn.Func.OpenCodedDeferDisallowed()
	...
	if s.hasOpenDefers &&
		s.curfn.Func.numReturns*s.curfn.Func.numDefers > 15 {
		s.hasOpenDefers = false
	}
	...
}
```

中间代码生成的这两个步骤会决定当前函数是否应该使用开放编码优化 `defer` 关键字，一旦确定使用开放编码，就会在编译期间初始化延迟比特和延迟记录。

### 延迟记录 [#](#%e5%bb%b6%e8%bf%9f%e8%ae%b0%e5%bd%95)

延迟比特和延迟记录是使用开放编码实现 `defer` 的两个最重要结构，一旦决定使用开放编码，[`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 会在编译期间在栈上初始化大小为 8 个比特的 `deferBits` 变量：

```go
func buildssa(fn *Node, worker int) *ssa.Func {
	...
	if s.hasOpenDefers {
		deferBitsTemp := tempAt(src.NoXPos, s.curfn, types.Types[TUINT8]) // 初始化延迟比特
		s.deferBitsTemp = deferBitsTemp
		startDeferBits := s.entryNewValue0(ssa.OpConst8, types.Types[TUINT8])
		s.vars[&deferBitsVar] = startDeferBits
		s.deferBitsAddr = s.addr(deferBitsTemp)
		s.store(types.Types[TUINT8], s.deferBitsAddr, startDeferBits)
		s.vars[&memVar] = s.newValue1Apos(ssa.OpVarLive, types.TypeMem, deferBitsTemp, s.mem(), false)
	}
}
```

延迟比特中的每一个比特位都表示该位对应的 `defer` 关键字是否需要被执行，如下图所示，其中 8 个比特的倒数第二个比特在函数返回前被设置成了 1，那么该比特位对应的函数会在函数返回前执行：

![golang-defer-bits](https://gitlab.com/moqsien/go-design-implementation/-/raw/main/golang-defer-bits.png)

**图 5-12 延迟比特**

因为不是函数中所有的 `defer` 语句都会在函数返回前执行，如下所示的代码只会在 `if` 语句的条件为真时，其中的 `defer` 语句才会在结尾被执行[4](#fn:4)：

```go
deferBits := 0 // 初始化 deferBits

_f1, _a1 := f1, a1  // 保存函数以及参数
deferBits |= 1 << 0 // 将 deferBits 最后一位置位 1

if condition {
    _f2, _a2 := f2, a2  // 保存函数以及参数
    deferBits |= 1 << 1 // 将 deferBits 倒数第二位置位 1
}
exit:

if deferBits & 1 << 1 != 0 {
    deferBits &^= 1 << 1
    _f2(a2)
}

if deferBits & 1 << 0 != 0 {
    deferBits &^= 1 << 0
    _f1(a1)
}
```

延迟比特的作用就是标记哪些 `defer` 关键字在函数中被执行，这样在函数返回时可以根据对应 `deferBits` 的内容确定执行的函数，而正是因为 `deferBits` 的大小仅为 8 比特，所以该优化的启用条件为函数中的 `defer` 关键字少于 8 个。

上述伪代码展示了开放编码的实现原理，但是仍然缺少了一些细节，例如：传入 `defer` 关键字的函数和参数都会存储在如下所示的 [`cmd/compile/internal/gc.openDeferInfo`](https://draveness.me/golang/tree/cmd/compile/internal/gc.openDeferInfo) 结构体中：

```go
type openDeferInfo struct {
	n           *Node
	closure     *ssa.Value
	closureNode *Node
	rcvr        *ssa.Value
	rcvrNode    *Node
	argVals     []*ssa.Value
	argNodes    []*Node
}
```

当编译器在调用 [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 构建中间代码时会通过 [`cmd/compile/internal/gc.state.openDeferRecord`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.openDeferRecord) 方法在栈上构建结构体，该结构体的 `closure` 中存储着调用的函数，`rcvr` 中存储着方法的接收者，而最后的 `argVals` 中存储了函数的参数。

很多 `defer` 语句都可以在编译期间判断是否被执行，如果函数中的 `defer` 语句都会在编译期间确定，中间代码生成阶段就会直接调用 [`cmd/compile/internal/gc.state.openDeferExit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.openDeferExit) 在函数返回前生成判断 `deferBits` 的代码，也就是上述伪代码中的后半部分。

不过当程序遇到运行时才能判断的条件语句时，我们仍然需要由运行时的 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 决定是否执行 `defer` 关键字：

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	sp := getcallersp()
	if d.openDefer {
		runOpenDeferFrame(gp, d)
		gp._defer = d.link
		freedefer(d)
		return
	}
	...
}
```

该函数为开放编码做了特殊的优化，运行时会调用 [`runtime.runOpenDeferFrame`](https://draveness.me/golang/tree/runtime.runOpenDeferFrame) 执行活跃的开放编码延迟函数，该函数会执行以下的工作：

1.  从 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中读取 `deferBits`、函数 `defer` 数量等信息；
2.  在循环中依次读取函数的地址和参数信息并通过 `deferBits` 判断该函数是否需要被执行；
3.  调用 [`runtime.reflectcallSave`](https://draveness.me/golang/tree/runtime.reflectcallSave) 调用需要执行的 `defer` 函数；

```go
func runOpenDeferFrame(gp *g, d *_defer) bool {
	fd := d.fd

	...
	deferBitsOffset, fd := readvarintUnsafe(fd)
	nDefers, fd := readvarintUnsafe(fd)
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))

	for i := int(nDefers) - 1; i >= 0; i-- {
		var argWidth, closureOffset, nArgs uint32 // 读取函数的地址和参数信息
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
		if deferBits&(1<<i) == 0 {
			...
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure

		...

		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		...
	}
	return done
}
```

Go 语言的编译器为了支持开放编码在中间代码生成阶段做出了很多修改，我们在这里虽然省略了很多细节，但是也可以很好地展示 `defer` 关键字的实现原理。

## 5.3.7 小结 [#](#537-%e5%b0%8f%e7%bb%93)

`defer` 关键字的实现主要依靠编译器和运行时的协作，我们总结一下本节提到的三种机制：

* 堆上分配 · 1.1 \~ 1.12
  * 编译期将 `defer` 关键字转换成 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 并在调用 `defer` 关键字的函数返回之前插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)；
  * 运行时调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 会将一个新的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体追加到当前 Goroutine 的链表头；
  * 运行时调用 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会从 Goroutine 的链表中取出 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构并依次执行；
* 栈上分配 · 1.13
  * 当该关键字在函数体中最多执行一次时，编译期间的 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会将结构体分配到栈上并调用 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack)；
* 开放编码 · 1.14 \~ 现在
  * 编译期间判断 `defer` 关键字、`return` 语句的个数确定是否开启开放编码优化；
  * 通过 `deferBits` 和 [`cmd/compile/internal/gc.openDeferInfo`](https://draveness.me/golang/tree/cmd/compile/internal/gc.openDeferInfo) 存储 `defer` 关键字的相关信息；
  * 如果 `defer` 关键字的执行可以在编译期间确定，会在函数返回前直接插入相应的代码，否则会由运行时的 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 处理；

我们在本节前面提到的两个现象在这里也可以解释清楚了：

* 后调用的 `defer` 函数会先执行：
  * 后调用的 `defer` 函数会被追加到 Goroutine `_defer` 链表的最前面；
  * 运行 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 时是从前到后依次执行；
* 函数的参数会被预先计算；
  * 调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 函数创建新的延迟调用时就会立刻拷贝函数的参数，函数的参数不会等到真正执行时计算；

## 5.3.8 延伸阅读 [#](#538-%e5%bb%b6%e4%bc%b8%e9%98%85%e8%af%bb)

* [5 Gotchas of Defer in Go — Part I](https://blog.learngoprogramming.com/gotchas-of-defer-in-go-1-8d070894cb01)
* [Golang defer clarification](https://stackoverflow.com/questions/28893586/golang-defer-clarification)
* [Dive into stack and defer/panic/recover in go](http://hustcat.github.io/dive-into-stack-defer-panic-recover-in-go/)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)

[上一节](/golang/docs/part2-foundation/ch05-keyword/golang-select/) [下一节](/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/)

* * *

1.  171758: cmd/compile,runtime: allocate defer records on the stack <https://go-review.googlesource.com/c/go/+/171758/> [↩︎](#fnref:1)

2.  190098: cmd/compile, cmd/link, runtime: make defers low-cost through inline code and extra funcdata <https://go-review.googlesource.com/c/go/+/190098/6> [↩︎](#fnref:2)

3.  Dan Scales, Keith Randall, and Austin Clements. Proposal: Low-cost defers through inline code, and extra funcdata to manage the panic case. 2019-09-23. <https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md> [↩︎](#fnref:3)

4.  Three mechanisms of Go language defer statements. <https://ofstack.com/Golang/28467/three-mechanisms-of-go-language-defer-statements.html> [↩︎](#fnref:4)