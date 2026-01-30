+++
date = '2026-01-24T15:51:13+08:00'
draft = false
title = 'Week1 Day5'
+++
## Day 5: 接口多态与流式复用

1. 接口多态性 (Interface Polymorphism)
    - 阅读 Language Specification 中关于 [Interface types](https://golang.google.cn/ref/spec#Interface_types) 的部分
        - An interface type defines a type set. A variable of interface type can store a value of any type that is in the type set of the interface. Such a type is said to implement the interface. The value of an uninitialized variable of interface type is nil.
        - **Basic interfaces**: Interfaces whose type sets can be defined entirely by a list of methods are called *basic interfaces*.
        - More than one type may implement an interface. Any given type may implement several distinct interfaces.
        - For convenience, the predeclared type any is an alias for the empty interface. [Go 1.18]
        - **Embedded interfaces**: In a slightly more general form an interface T may use a (possibly qualified) interface type name E as an interface element. This is called embedding interface E in T [Go 1.14]. The type set of T is the *intersection* of the type sets defined by T's explicitly declared methods and the type sets of T’s embedded interfaces. In other words, the type set of T is the set of all types that implement all the explicitly declared methods of T and also all the methods of E [Go 1.18].
            - When embedding interfaces, methods with the same names must have identical signatures.
        - **General interfaces**: In their most general form, an interface element may also be an arbitrary type term T, or a term of the form ~T specifying the underlying type T, or a union of terms t1|t2|…|tn [Go 1.18]. Together with method specifications, these elements enable the precise definition of an interface's type set as follows:
            > **By construction, an interface's type set never contains an interface type.**
            - The type set of the empty interface is the set of all non-interface types.
            - **The type set of a non-empty interface is the *intersection* of the type sets of its interface elements.**
            - The type set of a method specification is the set of all non-interface types whose method sets include that method.
            - The type set of a non-interface type term is the set consisting of just that type.
            - The type set of a term of the form ~T is the set of all types whose underlying type is T.
                > In a term of the form ~T, the underlying type of T must be itself, and T cannot be an interface.
            - The type set of a union of terms t1|t2|…|tn is the union of the type sets of the terms.
        - The quantification "the set of all non-interface types" refers not just to all (non-interface) types declared in the program at hand, but all possible types in all possible programs, and hence is infinite. Similarly, given the set of all non-interface types that implement a particular method, the intersection of the method sets of those types will contain exactly that method, even if all types in the program at hand always pair that method with another method.
        - The type T in a term of the form T or ~T cannot be a type parameter, and the type sets of all non-interface terms must be pairwise disjoint (the pairwise intersection of the type sets must be empty)
        - Implementation restriction: A union (with more than one term) cannot contain the [predeclared identifier](https://golang.google.cn/ref/spec#Predeclared_identifiers) **comparable** or interfaces that specify methods, or embed **comparable** or interfaces that specify methods.
        - Interfaces that are not basic(仅是一组方法的集合) may **only** be used as type constraints, or as elements of other interfaces used as constraints. They cannot be the types of values or variables, or components of other, non-interface types.
        - An interface type T may not embed a type element that is, contains, or embeds T, directly or indirectly.
        - **A type T implements an interface I if**
            - T is not an interface and is an element of the type set of I; or
            - T is an interface and the type set of T is a subset of the type set of I.
        - A value of type T implements an interface if T implements the interface.
    - 为什么 `io.Copy` 既能接收 `*os.File`（文件），又能接收 `os.Stdin`（键盘输入）？
        - 因为 `io.Copy` 的方法签名`func Copy(dst Writer, src Reader) (written int64, err error)`中 `src` 的类型是 `io.Reader` 接口:
            ```go
            type Reader interface {
                Read(p []byte) (n int, err error)
            }
            ```
            而 `*os.File` 实现了[`Read`](https://golang.google.cn/pkg/os/#File.Read)方法, 满足 `io.Reader` 接口
            ```go
            func (f *File) Read(b []byte) (n int, err error)
            ```
            `os.Stdin`是由[`NewFile`](https://golang.google.cn/pkg/os/#NewFile)函数得到的, 其类型是 `*File`, 因此也实现了`Read`方法
            ```go
            Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
            ```

2. [Everything is a file]({{< ref "/misc/unix-everything-is-a-file.md" >}})

3. 工程实践：`cat.go` v1, 实现基础`cat`功能
    - For utilities that use operands to represent files to be opened for either reading or writing, the '−' operand should be used to mean only standard input (or standard output when it is clear from context that an output file is being specified) or a file named −.
        > posix-1003.1-2024 Volume: 1 (Base Definitions) -> Chapter: 12 -> (Utility Conventions) -> Section: 12.2 (Utility Syntax Guidelines) -> Guideline: Guideline 13
        > 
        > XBD: X/Open **B**ase **D**efinitions (X来源于X/Open Company)
    - **在循环中，必须显式 Close**, 不然对于处理大量文件的场景, 会导致进程的文件描述符表（per-process FD table）耗尽, 最终触发 `open()` 返回 `EMFILE` 错误
    ```go
    package main

    import (
        "fmt"
        "io"
        "os"
    )

    // runCopy 是一个纯粹的逻辑函数。
    // 它不关心数据来自磁盘还是键盘，它只关心 input 满足 io.Reader 接口。
    // 这里的 'r' 既可以是 *os.File，也可以是 os.Stdin。
    func runCopy(r io.Reader) error {
        _, err := io.Copy(os.Stdout, r)
        return err
    }

    func main() {
        args := os.Args[1:]

        if len(args) == 0 {
            if err := runCopy(os.Stdin); err != nil {
                fmt.Fprint(os.Stderr, err)
                os.Exit(1)
            }
        }

        for _, filename := range args {
            if filename == "-" {
                if err := runCopy(os.Stdin); err != nil {
                    fmt.Fprint(os.Stderr, err)
                    //os.Exit(1)
                    continue
                }
            } else {

                f, err := os.Open(filename)
                if err != nil {
                    fmt.Fprintf(os.Stderr, "go-cat: %s: %v\n", filename, err)
                    continue
                }

                if err = runCopy(f); err != nil {
                    fmt.Fprint(os.Stderr, err)
                    os.Exit(1)
                }

                // 显式关闭
                f.Close()
            }
        }
    }

    ```

4. 验证功能
    ```console
    $ echo "Hello from Pipe" | go run cat.go no_exist.txt test1.txt - test2.txt
    go-cat: no_exist.txt: open no_exist.txt: no such file or directory
    hello
    Hello from Pipe
    world!
    ```

