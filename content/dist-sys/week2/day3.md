+++
date = '2026-02-07T22:08:19+08:00'
draft = false
title = 'Week2 Day3'
+++
## Week2 Day 3: 逃逸分析

1. 栈分配与堆分配
    - **栈分配**：
        - **特点**：极快。分配仅仅是栈指针（Stack Pointer）的移动。内存随函数返回自动回收，无需 GC 介入。
        - **场景**：生命周期仅限于当前函数栈帧内的局部变量。
    - **堆分配**：
        - **特点**：较慢。需要通过内存分配器寻找空闲块。生命周期不由函数作用域决定，需要 GC 追踪并在不再被引用时回收。
        - **场景**：生命周期超出当前函数、或者体积过大无法在栈上分配的变量。

2. 常见的逃逸场景
    > `escapes to heap`(逃逸到堆)：这是一个动词/状态。它描述的是数据的流向。意思是：“编译器发现这个指针/值正在流出它的作用域，它无法被限制在栈上了。”
    >
    > `moved to heap` (移到了堆)：这是一个动作/结果。它描述的是变量的重分配。意思是：“编译器原本想把这个变量分配在栈上，但因为上面的‘逃逸’原因，被迫把它搬迁到了堆上。”
    - `go build -gcflags="-m"`: 用来观察 Go 编译器在编译代码时做的两件核心工作(逃逸分析 (Escape Analysis), 函数内联 (Function Inlining))
        > 可以额外加入`-l`参数来关闭内联从而只看逃逸相关的信息
        - go build: 标准编译命令
        - -gcflags: "Go Compiler Flags" 的缩写, 允许把参数传给底层的编译器
        - "-m": print optimization decisions(可以用`go tool compile -help`查看)
    ```go {title = "指针逃逸 (Returning Pointers)"}
    // 编译器判定 `x` 在函数返回后仍需被访问，因此必须将其分配在堆上
    func returnPointer() *int {
        x := 1
        return &x
    }
    ```
    ```go {title = "接口动态分发(Interface Assignment)"}
    fmt.Println(val) // 由于接口方法的动态性，编译器往往无法确定具体调用的实现，保守起见通常会将其分配到堆上(fmt.Println 接收的是 any, 把一个具体类型（如 int）传给 any 时，Go 运行时会发生 Boxing (装箱)，创建一个 runtime.eface 结构体)
    ```
    ```go {title = "闭包引用(Closure Capture)"}
	val := *returnPointer()

	/*
		Go 的编译器在处理闭包时非常聪明，它遵循 “最小代价原则”
		能 Copy 就 Copy：如果是基础类型 (int, bool) 且只读，直接拷贝值
		不能 Copy 就 Share：如果会被修改，或者是个大对象，或者是引用类型，那就必须通过指针共享，这时候往往就会导致逃逸
	*/
	go func() {
		val = val + 1
	}()
    ```
    ```go {title = "栈空间不足(Stack Overflow Potential)"}
    s := make([]int, 1024) // 尽管 Go 的栈是动态伸缩的，但大对象仍倾向于堆(或编译期无法确定大小的变量, make([]byte, n), 其中 n 是一个变量)
    ```

3. 高性能代码应尽量减少堆分配（减少 GC 压力）, 可以减少指针逃逸、使用对象复用 (`sync.Pool`) 、预分配切片容量和优化结构体内存布局