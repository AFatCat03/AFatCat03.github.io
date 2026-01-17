+++
date = '2026-01-13T17:15:20+08:00'
draft = false
title = 'Week1 Day2'
+++
## Day 2: Go 语法速览与工程初探

1. 通过[A Tour of Go](https://golang.google.cn/tour/list)快速了解Go
    ### Packages, variables, and functions.
    - By convention, the package name is the same as the last element of the import path. (`"math/rand"` package comprises files that begin with the statement `package rand`)
    - In Go, a name is exported if it begins with a capital letter. When importing a package, you can refer only to its exported names. Any "unexported" names are not accessible from outside the package.
    - **type comes *after* the variable name**, [why the declaration syntax is different from the tradition established in the C family](https://golang.google.cn/blog/declaration-syntax).
    - When two or more consecutive named function parameters share a type, you can omit the type from all but the last
    - A function can return *any* number of results.
    - Go's return values may be named. If so, they are treated as variables defined at the top of the function.
    - A `return` statement without arguments returns the named return values. This is known as a "naked" return. Naked return statements should be used only in short functions. They can harm readability in longer functions.
    - `var`: declare a list of variables; as in function argument lists, the type is last. 
        - can be at package or function level. 
        - can include initializers, one per variable(`var i, j int = 1, 2`). If an initializer is present, the type can be omitted; the variable will take the type of the initializer(`var c, python, java = true, false, "no!"`)
    - `:=`: short assignment statement, can be used in place of a `var` declaration with **implicit** type.
        - can be at function level.
    - basic types
        > The `int`, `uint`, and `uintptr` types are usually 32 bits wide on 32-bit systems and 64 bits wide on 64-bit systems. When you need an integer value you should use `int` unless you have a specific reason to use a sized or unsigned integer type.
        - `bool`
        - `string`
        - `int` `int8` `int16` `int32` `int64`
        - `uint` `uint8` `uint16` `uint32` `uint64` `uintptr`
        - `byte` // alias for uint8
        - `rune` // alias for int32, represents a Unicode code point
        - `float32` `float64`
        - `complex64` `complex128`
    - Variables declared without an explicit initial value are given their zero value.
        - numeric types: `0`
        - boolean type: `false`
        - strings: `""`(the empty string)
        - pointer: `nil`
        - slice: `nil`
        - map: `nil`
    - `T(v)`: convert the value `v` to the type `T`.
        > Unlike in C, in Go assignment between items of different type requires an **explicit** conversion.
    - When declaring a variable without specifying an explicit type (either by using the `:=` syntax or `var =` expression syntax), the variable's type is inferred from the value on the right hand side.
        - When the right hand side of the declaration is typed, the new variable is of that same type
        - when the right hand side contains an untyped numeric constant, the new variable may be an `int`, `float64`, or `complex128` depending on the precision of the constant
    - Constants are declared like variables, but with the `const` keyword.
        - Constants can be character, string, boolean, or numeric values.
        - Constants cannot be declared using the `:=` syntax.
    - Numeric constants are high-precision *values*. An untyped constant takes the type needed by its context.
        > 定义一个 `const` 但不指定类型时，它不占用固定的内存空间，它存在于一个“理想的数学空间”里。
        > 这意味着，它可以存储极其巨大的数字，甚至超过 CPU 能处理的上限(`const Big = 1 << 100`, 超过了`int64`)

    ### Flow control statements: for, if, else, switch and defer
    - `for`: The basic `for` loop has three components separated by semicolons(`for i := 0; i < 10; i++`), there are no parentheses surrounding the three components of the `for` statement and the braces `{ }` are always required.
        > The init and post statements are optional.(`for ; sum < 1000;`)
        - the init statement: executed before the first iteration, the variables declared there are visible only in the scope of the for statement
        - the condition expression: evaluated before every iteration
        - the post statement: executed at the end of every iteration
        - 只有condition的`for`相当于`C`的`while`(`for sum < 1000`)
        - 空`for`相当于an infinite loop(`for`)
    - `if`: Go's `if` statements are like its `for` loops; the expression need not be surrounded by parentheses `( )` but the braces `{ }` are required.
        - can start with a short statement to execute before the condition(`if v := math.Pow(x, n); v < lim `), the variables declared by the statement are only in scope until the end of the `if`, and are also available inside any of the `else` blocks.
        - **`if ... else` 的结构必须严格遵守 One True Brace Style (1TBS)**
            > Go 并不是没有分号，而是通过严格的代码格式规范（换行规则），换来了免写分号的便利
            > 
            > **自动分号插入机制 (Automatic Semicolon Insertion, ASI): 如果一行代码以一个看起来像语句结束的符号（比如标识符、数字、return、`}` 等）结尾，就自动在这行末尾加一个分号 ;**
            - `{`必须紧跟在语句（如 `if`, `for`, `func`）的同一行末尾，不能换行
            - `else` / `catch` 必须紧跟在 `}` 的同一行后面
    `switch`: a shorter way to write a sequence of `if - else` statements. It runs the first case whose value is equal to the condition expression.
        -  In effect, the `break` statement that is needed at the end of each case in those languages(C, C++, Java...) is provided automatically in Go.
        - Another important difference is that Go's switch cases need not be constants, and the values involved need not be integers.
        - Switch cases evaluate cases from top to bottom, stopping when a case succeeds.
        - Switch without a condition is the same as `switch true`.
    - A defer statement defers the execution of a function until the surrounding function returns.
        > The deferred call's arguments are evaluated immediately, but the function call is not executed until the surrounding function returns.
        - Deferred function calls are pushed onto a stack. When a function returns, its deferred calls are executed in last-in-first-out order.
        - Defer is commonly used to simplify functions that perform various clean-up actions.
    - [Defer, Panic, and Recover](https://golang.google.cn/blog/defer-panic-and-recover)

    ### More types: structs, slices, and maps.
    - `*T`: A pointer holds the memory address of a `T` value.
        > Unlike C, Go has no pointer arithmetic.
        - The `&` operator generates a pointer to its operand.
        - The `*` operator denotes the pointer's underlying value.
    - A `struct` is a collection of fields.
        - Struct fields are accessed using a dot.
        - Struct fields can be accessed through a struct pointer.
            > To access the field `X` of a struct when we have the struct pointer `p` we could write `(*p).X`. However, that notation is cumbersome, so the language permits us instead to write just `p.X`, without the explicit dereference.
        - A struct literal denotes a newly allocated struct value by listing the values of its fields.
        - You can list just a subset of fields by using the `Name:` syntax. (And the order of named fields is irrelevant.)
    - `[n]T`: an array of `n` values of type `T`.
        - An array's length is part of its type, so arrays cannot be resized.
        - 显示声明大小: `b := [2]string{"Penn", "Teller"}`
        - 让编译器计算大小: `b := [...]string{"Penn", "Teller"}`
    - `[]T`: a slice with elements of type `T`.
        - A slice is formed by specifying two indices, a low and high bound, separated by a colon: `a[low : high]`
            > This selects a half-open range which includes the first element, but excludes the last one.
        - Slices are like references to arrays
            - A slice does not store any data, it just describes a section of an underlying array.
            - Changing the elements of a slice modifies the corresponding elements of its underlying array. Other slices that share the same underlying array will see those changes.
        - A slice literal is like an array literal without the length.
            - `[3]bool{true, true, false}`: an array literal
            - `[]bool{true, true, false}`: this creates the same array as above, then builds a slice that references it
        - When slicing, you may omit the high or low bounds to use their defaults instead.
            - low bound: zero
            - high bound: the length of the slice
        - A slice has both a *length* and a *capacity*.
            - length: `len(s)` the number of elements it contains. 
            - capacity: `cap(s)` the number of elements in the underlying array, counting from the first element in the slice.
            > 把左边界向右移（`s = s[1:]`），容量会减小！相当于前面的格子被抛弃了，且无法找回（除非你还持有原始数组的引用）
            >
            > 只要 length < capacity, 就可以把窗口向右拉大(re-slicing, `s = s[:0]`->`s = s[:5]`), 但是不能超过底层数组的物理边界。
        - A nil slice has a length and capacity of 0 and has no underlying array.
        - `make`: allocate a zeroed array and returns a slice that refers to that array
            - `a := make([]int, 5)  // len(a)=5`此时capacity = length
            - `b := make([]int, 0, 5) // len(b)=0, cap(b)=5`
        - Slices can contain any type, including other slices.
        - Go provides a built-in append function to append new elements to a slice, `func append(s []T, vs ...T) []T`
            - The first parameter `s` of append is a slice of type `T`, and the rest are `T` values to append to the slice.
            - The resulting value of `append` is a slice containing all the elements of the original slice plus the provided values.
            - If the backing array of `s` is too small to fit all the given values a bigger array will be allocated. The returned slice will point to the newly allocated array.
            - To append one slice to another, use `...` to expand the second argument to a list of arguments.
                ```go
                a := []string{"John", "Paul"}
                b := []string{"George", "Ringo", "Pete"}
                a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
                // a == []string{"John", "Paul", "George", "Ringo", "Pete"}
                ```
        - [Go Slices: usage and internals](https://golang.google.cn/blog/slices-intro)
            - Go’s arrays are values. An array variable denotes the entire array; it is not a pointer to the first array element (as would be the case in C). This means that when you assign or pass around an array value you will make a copy of its contents. (To avoid the copy you could pass a pointer to the array, but then that’s a pointer to an array, not an array.) One way to think about arrays is as a sort of struct but with indexed rather than named fields: a fixed-size composite value.
            - A slice is a descriptor of an array segment. It consists of a pointer to the array, the length of the segment, and its capacity (the maximum length of the segment).
            - Slicing does not copy the slice’s data. It creates a new slice value that points to the original array. This makes slice operations as efficient as manipulating array indices. Therefore, modifying the elements (not the slice itself) of a re-slice modifies the elements of the original slice
            - A slice cannot be grown beyond its capacity. Attempting to do so will cause a runtime panic, just as when indexing outside the bounds of a slice or array. Similarly, slices cannot be re-sliced below zero to access earlier elements in the array.
            - Since the zero value of a slice (nil) acts like a zero-length slice, you can declare a slice variable and then append to it in a loop
    - The `range` form of the `for` loop iterates over a slice or map.
        - When ranging over a slice, two values are returned for each iteration. The first is the index, and the second is a copy of the element at that index.
        - You can skip the index or value by assigning to `_`.
            ```go
            for i, _ := range pow
            for _, value := range pow
            ``` 
        - If you only want the index, you can omit the second variable.(`for i := range pow`)
        - When ranging over a map, two values are returned for each iteration. The first is the key, and the second is the value. **Note that the iteration order is random, and both returned values are copies.**
    - A map maps keys to values.
        - The zero value of a map is `nil`. A `nil` map has no keys, nor can keys be added.
        - The `make` function returns a map of the given type, initialized and ready for use. `make(map[string]int)`
        - Map literals are like struct literals, but the keys are required.
            ```go
            var m = map[string]Vertex{
                "Bell Labs": Vertex{
                    40.68433, -74.39967,
                },
                "Google": Vertex{
                    37.42202, -122.08408,
                },
            }
            ```
        - If the top-level type is just a type name, you can omit it from the elements of the literal.
            ```go
            var m = map[string]Vertex{
                "Bell Labs": {40.68433, -74.39967},
                "Google":    {37.42202, -122.08408},
            }
            ```
        - Insert or update an element in map m: `m[key] = elem`
        - Retrieve an element: `elem = m[key]`
        - Delete an element: `delete(m, key)`
        - Test that a key is present with a two-value assignment: `elem, ok = m[key]`
            > Note: If `elem` or `ok` have not yet been declared you could use a short declaration form: `elem, ok := m[key]`
            - If `key` is in `m`, `ok` is `true`. If not, `ok` is `false`.
            - If `key` is not in the map, then `elem` is the zero value for the map's element type.
    - Functions are values too. They can be passed around just like other values.
        - Function values may be used as function arguments and return values.
        - Go functions may be closures. A closure is a function value that references variables from outside its body. The function may access and assign to the referenced variables; in this sense the function is "bound" to the variables.
            > 闭包捕获的是变量的内存地址, 当闭包真正被执行的时候，它会去那个地址里读取当前最新的值

    ### Methods and interfaces
    - A method **is a function** with a special *receiver* argument. `func (v Vertex) Abs() float64`
        > - Methods with pointer receivers can modify the value to which the receiver points
        > - With a value receiver, the method operates on a copy of the original value
        - The receiver appears in its own argument list between the `func` keyword and the method name.
        - You can declare a method on non-struct types, too. But you can only declare a method with a receiver whose type is defined in the same package as the method. You cannot declare a method with a receiver whose type is defined in another package (which **includes the built-in types such as `int`**).
        - You can declare methods with pointer receivers. This means the receiver type has the literal syntax `*T` for some type `T`. (Also, `T` cannot itself be a pointer such as `*int`.)
        - Functions with a pointer argument must take a pointer, while methods with pointer receivers take either a value or a pointer as the receiver when they are called(That is, as a convenience, Go interprets the statement `v.Scale(5)` as `(&v).Scale(5)` since the `Scale` method has a pointer receiver.)
        - Functions that take a value argument must take a value of that specific type, while methods with value receivers take either a value or a pointer as the receiver when they are called(In this case, the method call `p.Abs()` is interpreted as `(*p).Abs()`)
        - There are two reasons to use a pointer receiver.
            - The first is so that the method can modify the value that its receiver points to.
            - The second is to avoid copying the value on each method call. This can be more efficient if the receiver is a large struct, for example.
        - In general, all methods on a given type should have either value or pointer receivers, but not a mixture of both.
    - An *interface type* is defined as a set of method signatures.
        - A value of interface type can hold any value that implements those methods.
        - **Interfaces are implemented implicitly**
            - A type implements an interface by implementing its methods. There is no explicit declaration of intent, no "implements" keyword.
            - Implicit interfaces decouple the definition of an interface from its implementation, which could then appear in any package without prearrangement.(可以为任何现有的类型（包括标准库类型）定义接口，只要方法签名对得上，它们就自动匹配)
        - Under the hood, interface values can be thought of as a tuple of a value and a concrete type: `(value, type)`. 
            - An interface value holds a value of a specific underlying concrete type.
            - Calling a method on an interface value executes the method of the same name on its underlying type.
        - If the concrete value inside the interface itself is nil, the method will be called with a nil receiver.
            > an interface value that holds a nil concrete value is itself non-nil. `%v`打印的是接口里面装着的那个值的状态，而不是接口变量本身的状态
        - A nil interface value holds neither value nor concrete type.
            - Calling a method on a nil interface is a run-time error because there is no type inside the interface tuple to indicate which concrete method to call.
        - The interface type that specifies zero methods is known as the *empty interface*: `interface{}`
            - An empty interface may hold values of any type. (Every type implements at least zero methods.)
            - Empty interfaces are used by code that handles values of unknown type. For example, `fmt.Print` takes any number of arguments of type `interface{}`.
    - A *type assertion* provides access to an interface value's underlying concrete value.
        - `t := i.(T)`asserts that the interface value `i` holds the concrete type `T` and assigns the underlying `T` value to the variable `t`. If `i` does not hold a `T`, the statement will trigger a panic.
        - To *test* whether an interface value holds a specific type, a type assertion can return two values: the underlying value and a boolean value that reports whether the assertion succeeded. `t, ok := i.(T)`
            - If `i` holds a `T`, then `t` will be the underlying value and `ok` will be true.
            - If not, `ok` will be false and `t` will be the zero value of type `T`, and no panic occurs.
    - A *type switch* is a construct that permits several type assertions in series.
        - A type switch is like a regular switch statement, but the cases in a type switch specify types (not values), and those values are compared against the type of the value held by the given interface value.
        ```go
        switch v := i.(type) {
        case T:
            // here v has type T
        case S:
            // here v has type S
        default:
            // no match; here v has the same type as i
        }
        ```
        - The declaration in a type switch has the same syntax as a type assertion `i.(T)`, but the specific type `T` is replaced with the **keyword `type`**.
    - One of the most ubiquitous interfaces is `Stringer` defined by the `fmt` package.
        ```go
        type Stringer interface {
            String() string
        }
        ```
        - A `Stringer` is a type that can describe itself as a string. The `fmt` package (and many others) look for this interface to print values.
    - Go programs express error state with `error` values.
        - The `error` type is a built-in interface similar to `fmt.Stringer`
        ```go
        type error interface {
            Error() string
        }
        ```
        - As with `fmt.Stringer`, the `fmt` package looks for the `error` interface when printing values.
        - Functions often return an `error` value, and calling code should handle errors by testing whether the error equals `nil`. `i, err := strconv.Atoi("42")`
            - A nil `error` denotes success
            - A non-nil `error` denotes failure.
    - 在 fmt 需要将对象“作为字符串”或“以默认格式”展示时，会触发 error > Stringer 的优先权逻辑(同时实现了 `Error()` 和 `String()`, 会无视 `String()`, 直接调用 `Error()`), 这个优先权逻辑主要适用于以下 String-like Verbs:
        - `%v` (默认格式)
        - `%s` (字符串)
        - `%q` (带引号的字符串)
        - ...
        - 以及 `Print`, `Println`, `Sprint` 等如果不指定 verb 时的默认行为。
    - The `io` package specifies the `io.Reader` interface, which represents the read end of a stream of data.
        - The `io.Reader` interface has a `Read` method: `func (T) Read(b []byte) (n int, err error)`
        - `Read` populates the given byte slice with data and returns the number of bytes populated and an error value. It returns an `io.EOF` error when the stream ends.
        - A common pattern is an [io.Reader](https://golang.google.cn/pkg/io/#Reader) that wraps another `io.Reader`, modifying the stream in some way. (Decorator Pattern)
    - [Package image](https://golang.google.cn/pkg/image/#Image) defines the Image interface
        ```go
        type Image interface {
            ColorModel() color.Model
            Bounds() Rectangle
            At(x, y int) color.Color
        }
        ```

    ### Generics
    - Go functions can be written to work on multiple types using type parameters. The type parameters of a function appear between brackets, before the function's arguments.
        - `func Index[T comparable](s []T, x T) int`: This declaration means that `s` is a slice of any type `T` that fulfills the built-in constraint `comparable`. `x` is also a value of the same type.
            > `comparable` is a useful constraint that makes it possible to use the `==` and `!=` operators on values of the type. 
    - Generic types: In addition to generic functions, Go also supports generic types. A type can be parameterized with a type parameter, which could be useful for implementing generic data structures.
        > [any](https://pkg.go.dev/builtin#any): `type any = interface{}`, any is an alias for interface{} and is equivalent to interface{} in all ways.
        ```go
        type List[T any] struct {
            next *List[T]
            val  T
        }
        ```

    ### Concurrency
    - Goroutines
        - A *goroutine* is a lightweight thread managed by the Go runtime.
        - `go f(x, y, z)` starts a new goroutine running `f(x, y, z)`. The evaluation of `f`, `x`, `y`, and `z` happens in the current goroutine and the execution of `f` happens in the new goroutine.
        - Goroutines run in the same address space, so access to shared memory must be synchronized. The `sync` package provides useful primitives.
    - Channels
        - Channels are a typed conduit through which you can send and receive values with the channel operator, `<-`.(The data flows in the direction of the arrow.)
        ```go
        ch <- v    // Send v to channel ch.
        v := <-ch  // Receive from ch, and
                // assign value to v.
        ```
        - Like maps and slices, channels must be created before use: `ch := make(chan int)`
        - By default, sends and receives block until the other side is ready. This allows goroutines to synchronize without explicit locks or condition variables.
        - Buffered Channels
            - Channels can be *buffered*. Provide the buffer length as the second argument to make to initialize a buffered channel: `ch := make(chan int, 100)`
            - Sends to a buffered channel block only when the buffer is full. Receives block when the buffer is empty.
        - A sender can `close` a channel to indicate that no more values will be sent. Receivers can test whether a channel has been closed by assigning a second parameter to the receive expression: `v, ok := <-ch`
            > 即使发送方执行了 `close(ch)`，通道里残留的数据依然有效，读取这些残留数据时，`ok` 依然是 `true`
            > 
            > 只有当通道彻底被抽干（Empty） 且 已关闭（Closed） 时，`ok` 才会变成 `false`
            - `ok` is `false` if there are no more values to receive and the channel is closed.
            - 如果通道没准备好（没数据也没关），它就不返回（阻塞）。只有当通道“有结果”（不管是数据还是关闭信号）时，它才返回
            - `ok` 为 `false` 时，值是无意义的零值。最后一个有效值是在上一次 ok 为 true 时被读走的
        - The loop `for i := range c` receives values from the channel repeatedly until it is closed.
        - **Note**: Only the sender should close a channel, never the receiver. Sending on a closed channel will cause a panic.
        - **Another note**: Channels aren't like files; you don't usually need to close them. Closing is only necessary when the receiver must be told there are no more values coming, such as to terminate a `range` loop.
    - Select: The `select` statement lets a goroutine wait on multiple communication operations. 
        - A `select` blocks until one of its cases can run, then it executes that case. It chooses one at random if multiple are ready.
        - The `default` case in a `select` is run if no other case is ready.
        - **`select` 的 case 是 及早求值 (Eager Evaluation)。所有 case 的表达式都会被算一遍，然后才决定走哪条路**
        - **`switch` 的 case 是 惰性求值 (Lazy Evaluation)。它完全遵循“短路”原则，一旦匹配成功，后面的 case 连看都不会看一眼**
    - sync.Mutex
        - What if we don't need communication? What if we just want to make sure only one goroutine can access a variable at a time to avoid conflicts?
        - This concept is called *mutual exclusion*, and the conventional name for the data structure that provides it is *mutex*.
        - Go's standard library provides mutual exclusion with `sync.Mutex` and its two methods: `Lock`, `Unlock`
        - We can define a block of code to be executed in mutual exclusion by surrounding it with a call to `Lock` and `Unlock` as shown on the `Inc` method.
            ```go
            // SafeCounter is safe to use concurrently.
            type SafeCounter struct {
                mu sync.Mutex
                v  map[string]int
            }

            // Inc increments the counter for the given key.
            func (c *SafeCounter) Inc(key string) {
                c.mu.Lock()
                // Lock so only one goroutine at a time can access the map c.v.
                c.v[key]++
                c.mu.Unlock()
            }
            ```
        - We can also use `defer` to ensure the mutex will be unlocked as in the `Value` method.
            ```go
            // Value returns the current value of the counter for the given key.
            func (c *SafeCounter) Value(key string) int {
                c.mu.Lock()
                // Lock so only one goroutine at a time can access the map c.v.
                defer c.mu.Unlock()
                return c.v[key]
            }
            ```
    
    ### When are function parameters passed by value?(https://go.dev/doc/faq#pass_by_value)
    > Note that this discussion is about the semantics of the operations. Actual implementations may apply optimizations to avoid copying as long as the optimizations do not change the semantics.
    - Go 语言只有“值传递” (Pass by Value): 在 Go 中，函数调用或赋值时，永远是对变量的直接值 (Immediate Value) 进行位拷贝 (Bitwise Copy)
    - 不同类型的“直接值”定义不同, 根据底层内存布局将 Go 的类型分为三类
        - 直接值即数据 (Direct Types)
            - 类型：int, float, bool, array (注意数组是值类型), struct (纯数据结构体, 其所有字段本身也都是 Direct Types)
            - 行为：变量的内存中存储的就是实际数据
            - 传递后果：完全深拷贝, 复制了整个数据块
        - 直接值即指针 (Pointer Types)
            - 类型：*T (指针), map, chan, func(行为上类似指针，但不应依赖其内存布局)
            - 底层揭秘
                - Pointer: 显而易见，存储的是内存地址（如 8 字节）
                - Map: 运行时类型是 *runtime.hmap。变量本质上是一个指向 hmap 结构体的指针
                - Channel: 运行时类型是 *runtime.hchan, 变量本质上是一个指向 hchan 结构体的指针
            - 传递后果：指针拷贝, 复制了一个新的指针变量（副本），但它存储的地址值与原指针相同
        - 直接值即描述符 (Header / Descriptor Types)
            - 类型：slice, interface, string
            - 底层揭秘：变量的内存中存储的是一个小的结构体（Header）。传递时，拷贝的是这个结构体本身（浅拷贝）
            - Slice (切片) 它不是一个指针，而是一个 24 字节（64位机）的结构体：
                ```go
                type SliceHeader struct {
                    Data uintptr // 指向底层数组的指针
                    Len  int     // 长度
                    Cap  int     // 容量
                }
                ```
                - 传递后果：拷贝了 {Data, Len, Cap} 这三个字段
                - 严谨结论：“浅拷贝” (Shallow Copy)。
                    - 因为拷贝了 Data 指针，所以函数内修改元素 (s[0] = 1) 会影响外部。
                    - 因为 Len 和 Cap 是值拷贝，所以函数内执行 append 导致扩容或长度变化时，不会影响外部变量的 Len/Cap。（这就是为什么说 Slice 不是纯粹的引用类型）
            - String (字符串) 它是一个 16 字节的结构体（只读）
                ```go
                type StringHeader struct {
                    Data uintptr // 指向底层字节数组
                    Len  int
                }
                ```
                - 传递后果：拷贝了 {Data, Len}。由于字符串不可变，表现上像值类型，但底层共享了内存
            - Interface (接口) 它是一个 16 字节的结构体
                ```go
                type iface struct {
                    tab  *itab          // 存储类型信息和方法表
                    data unsafe.Pointer // 指向实际数据的指针
                }
                ```
                - 传递后果：拷贝了 {tab, data}
                - Copying an interface value makes a copy of the thing stored in the interface value. If the interface value holds a struct, copying the interface value makes a copy of the struct. If the interface value holds a pointer, copying the interface value makes a copy of the pointer, but again not the data it points to.
    
2. 工程实战：CLI 版 FizzBuzz 与 Go 工具链 (run/fmt)
    - `go run`: 将源码编译成临时二进制文件，立即运行，运行完自动清理
    - `go fmt`: 自动调整代码的缩进、空格和换行
    - `os.Args`: 得到的是一个 字符串切片 (`[]string`), 包含了程序启动时传递给它的所有命令行参数
        - `os.Args[0]`: 是程序本身的名字（或路径）
        - `os.Args[1:]`: 用户真正输入的参数
    - [`strconv.Atoi`](https://golang.google.cn/pkg/strconv/#Atoi): 将字符串转为对应整数, `s, err := strconv.Atoi(v)`
    - 编写一个名为 `fizzbuzz.go` 的程序, 如果是 3 的倍数打印 "Fizz"，5 的倍数打印 "Buzz"，15 的倍数打印 "FizzBuzz", 让上限（100）通过命令行参数 `os.Args[1]` 传入
    ```go
    package main

    import (
        "fmt"
        "os"
        "strconv"
    )

    func main() {
        bound, err := strconv.Atoi(os.Args[1])
        if err != nil {
            fmt.Println(err)
            return
        }
        for i := 1; i <= bound; i++ {
            fizzbuzz(i)
        }
    }

    func fizzbuzz(num int) {
        fmt.Printf("%v:", num)
        if num%3 == 0 {
            fmt.Print("Fizz")
        }
        if num%5 == 0 {
            fmt.Print("Buzz")
        }
        fmt.Println()
    }
    ```