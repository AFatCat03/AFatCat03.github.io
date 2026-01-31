+++
date = '2026-01-31T09:25:45+08:00'
draft = false
title = 'Week1 Day7'
+++
## Day 7: 工程化收尾与质量内省

1. `go vet`是 Go 语言官方工具链中内置的 **静态代码分析工具**, 追求 高准确率 (High Precision), 使用启发式算法来扫描源码，寻找那些“虽然编译通过，但极大概率是写错了”的可疑结构; go test 命令默认会自动执行 go vet 的一个子集
    ```go {title = "用lab_vet.go测试go vet"}
    package main

    import "fmt"

    func main() {
        name := "Go-Cat"
        // 错误：使用 %d (整数) 打印 string
        fmt.Printf("Hello %d\n", name)
    }

    ```
    ```console {title = "go run lab_vet.go"}
    Hello %!d(string=Go-Cat)
    ```
    - `%!d(string=Go-Cat)`解释
        - `%!`: 报警信号
        - `d`: 目标动词
        - `(...)`: 实际情况
        - `string`: 实际类型
        - `=`: 类型和值的分割
        - `Go-Cat`: 实际值
    ```console {title = "go vet lab_vet.go"}
    # command-line-arguments
    # [command-line-arguments]
    ./lab_vet.go:8:20: fmt.Printf format %d has arg name of wrong type string
    ```

2. `go run -race` 启用 Go 语言内置的 **数据竞争检测器 (Data Race Detector)**, 动态分析, 只能捕获实际发生的竞争，而不是可能发生的竞争
    ```go {title = "用lab_race.go测试go run -race"}
    package main

    import (
        "fmt"
        "time"
    )

    func main() {
        c := 0
        // 启动一个 goroutine 修改 c
        go func() {
            c = 1
        }()
        // 主线程读取 c
        // 两个线程同时访问内存，且没有加锁 -> 数据竞争！
        if c == 0 {
            fmt.Println("C is 0")
        }
        time.Sleep(time.Second)
    }

    ```
    ```console {title = "go run -race lab_race.go"}
    C is 0
    ==================
    WARNING: DATA RACE
    Write at 0x00c0000140e8 by goroutine 8:
    main.main.func1()
        /home/axiolux/workspace/dist-sys-42/week1/day7/lab_race.go:12 +0x2e

    Previous read at 0x00c0000140e8 by main goroutine:
    main.main()
        /home/axiolux/workspace/dist-sys-42/week1/day7/lab_race.go:16 +0xae

    Goroutine 8 (running) created at:
    main.main()
        /home/axiolux/workspace/dist-sys-42/week1/day7/lab_race.go:11 +0xa4
    ==================
    Found 1 data race(s)
    exit status 66
    ```
    > 66 是 `race detector` 专用的退出代码

3. 生产环境发布构建 (Release Build): `go build -ldflags "-s -w" -o go-cat-linux cat.go`/`GOOS=windows GOARCH=amd64 go build -ldflags "-s -w" -o go-cat-win.exe cat.go`
    |参数|含义|解读|
    |-|-|-|
    |go build|编译指令|启动 Go 编译器流水线。|
    |-o go-cat-linux|Output (输出)|指定生成的可执行文件名叫 go-cat-linux。如果不加这个，默认会叫 main (Windows下是 main.exe)。|
    |main.go|Source (源码)|指定入口文件。|
    |"-ldflags ""..."""|Linker Flags|把参数传递给底层的 链接器 (go tool link)，而不是编译器。|
    |-s|Strip Symbol Table|去掉符号表。移除函数名、变量名与内存地址的映射关系。|
    |-w|Strip DWARF|去掉调试信息。移除 DWARF (Debugging With Attributed Record Formats) 信息，这会导致 gdb 或 dlv 无法调试它。|
    - 输出大小对比
    ```console {title = "go build -o cat-fat cat.go" }
    -rwxr-xr-x 1 axiolux axiolux 2.4M Jan 31 11:15 cat-fat
    ```
    ```console {title = "go build -ldflags \"-s -w\" -o cat-slim cat.go" }
    -rwxr-xr-x 1 axiolux axiolux 1.6M Jan 31 11:15 cat-slim
    ```
