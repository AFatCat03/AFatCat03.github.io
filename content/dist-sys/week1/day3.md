+++
date = '2026-01-17T20:58:37+08:00'
draft = false
title = 'Week1 Day3'
+++
## Day 3: I/O 接口契约与内核边界探测

1. 研读[`io.Reader`](https://golang.google.cn/pkg/io/#Reader)接口定义
    ```go
    type Reader interface {
        Read(p []byte) (n int, err error)
    }
    ```
    - Reader is the interface that wraps the basic Read method.
    - Read reads up to len(p) bytes into p. It returns the number of bytes read (0 <= n <= len(p)) and any error encountered. 
    - **Even if Read returns n < len(p), it may use all of p as scratch space during the call.** 
    - **If some data is available but not len(p) bytes, Read conventionally returns what is available instead of waiting for more.**
    - When Read encounters an error or end-of-file condition after successfully reading n > 0 bytes, it returns the number of bytes read. *It may return the (non-nil) error from the same call or return the error (and n == 0) from a subsequent call.*
        - An instance of this general case is that a Reader returning a non-zero number of bytes at the end of the input stream may return either err == EOF or err == nil. The next Read should return 0, EOF.
    - Callers should always process the n > 0 bytes returned **before** considering the error err. Doing so correctly handles I/O errors that happen after reading some bytes and also both of the allowed EOF behaviors.
    - If len(p) == 0, Read should always return n == 0. It may return a non-nil error if some error condition is known, such as EOF.
    - Implementations of Read are discouraged from returning a zero byte count with a nil error, except when len(p) == 0. Callers should treat a return of 0 and nil as indicating that nothing happened; in particular it does not indicate EOF.
    - **Implementations must not retain p.**
    - 为什么 `p` 是由调用者（Caller）分配而不是被调者（Callee）
        1. 内存复用: Caller 可以在循环外面只分配一次内存, 然后在循环里复用这一块内存; Callee 分配的情况下每次调用都需要分配内存, 大大增加GC的负担
        2. 栈分配 vs 堆分配: Caller分配的p如果没有被传到其他协程或不是全局变量, 编译器很可能会把它直接分配在栈上; Callee 分配的p最终会从栈上逃逸到堆上, 因为该变量在函数结束后仍然被外部需要
        3. 由调用者分配，意味着调用者拥有决定权, 更加灵活
    可以用以下这段**专业且严谨**的术语来描述这一过程：
    - `read` 调用遵循 **即时可用性 (Immediate Availability)** 原则:
        > 有界延迟: 如果允许 `read` 贪婪地等待新数据，那么在一个高吞吐的流中，`read` 可能永远无法结束（因为总有新数据来）。现在的设计保证了 `read` 只要处理完那一瞬间的存量就会立即返回，耗时是可控的。
        > 
        > I/O 饥饿: 应用程序因为系统调用迟迟不返回，导致无法处理手头已有的数据，从而陷入逻辑停滞的状态
        >
        > 短读取: 当请求读取 $N$ 个字节时，系统只返回了 $M$ 个字节，且 $0 < M < N$，同时没有报错
    
        内核在决定开始传输数据的时刻，观测并确定当前可用数据量的上界，并据此决定本次传输的最大规模, 随后的 **内核空间到用户空间的数据复制 (Copy-to-User)** 将严格受限于数值 
        $$N = \min(\text{Buffer}_{\text{available}}, \text{Buffer}_{\text{requested}})$$
        在此复制窗口期内 (Copy Window) 到达的新数据，对于本次调用而言处于 **不可见 (Invisible)** 状态。这种设计旨在避免 **I/O 饥饿 (I/O Starvation)**，确保系统调用具有 **有界的响应延迟 (Bounded Latency)**，由此产生的现象在工程上被称为 **短读取 (Short Read)**。

2. 编写 `simple_echo.go`
    - 不使用 `fmt.Scan` 或 `fmt.Println`，而是使用底层的 [`io.Copy`](https://golang.google.cn/pkg/io/#Copy) 实现回显
    - `io.Copy`: `func Copy(dst Writer, src Reader) (written int64, err error)`
        - Copy copies from src to dst until either EOF is reached on src or an error occurs. It returns the number of bytes copied and the first error encountered while copying, if any.
        - A successful Copy returns err == nil, **not** err == EOF. Because Copy is defined to read from src until EOF, it does not treat an EOF from Read as an error to be reported.
            > EOF 只是作为“判断条件”（Signal），不会作为“数据”（Payload）被读入或写入
            > 
            > 在 Go 语言（以及 Unix 哲学）中，io.EOF 是一个 Error 类型 的值，而不是一个物理存在的字符（比如 C 语言里的 \0 或 -1）
    - [`os.Stdin`/`os.Stdout`](https://golang.google.cn/pkg/os/#pkg-variables): Stdin, Stdout, and Stderr are open Files pointing to the standard input, standard output, and standard error file descriptors.
    ```go
    package main

    import (
        "io"
        "os"
    )

    func main() {
        if _, err := io.Copy(os.Stdout, os.Stdin); err != nil {
            os.Exit(1)
        }
    }
    ```
    - 使用`go build -o echo simple_echo.go`编译
    - 用`./echo <  test.txt`和`ls -l | ./echo | grep "go"`验证符合 Unix 标准流规范
        > Unix 工具开发的黄金法则: Expect the output of every program to become the input to another, as yet unknown, program.

3. 查看[`io.Copy`](https://golang.google.cn/src/io/io.go?s=13934:13994#L377)源码
    - Go 可以把一个（或多个）接口的名字直接写在另一个接口里面。这就相当于把那个接口的所有方法都“粘贴”了进来
        > Has-a
        ```go
        type Reader interface {
            Read(p []byte) (n int, err error)
        }

        type Writer interface {
            Write(p []byte) (n int, err error)
        }

        // ReadWriter 嵌入了 Reader 和 Writer
        // 这意味着 ReadWriter 接口自动拥有了 Read() 和 Write() 两个方法
        type ReadWriter interface {
            Reader  // 嵌入
            Writer  // 嵌入
        }
        ```
        - [WriterTo](https://golang.google.cn/pkg/io/#WriterTo) is the interface that wraps the WriteTo method.
            - WriteTo writes data to w until there's no more data to write or when an error occurs. The return value n is the number of bytes written. Any error encountered during the write is also returned.
        - A [LimitedReader](https://golang.google.cn/pkg/io/#LimitedReader) reads from R but limits the amount of data returned to just N bytes. Each call to Read updates N to reflect the new amount remaining. Read returns EOF when N <= 0 or when the underlying R returns EOF.
            ```go
            type LimitedReader struct {
                R Reader // underlying reader
                N int64  // max bytes remaining
            }
            ```
        - [ReaderFrom](https://golang.google.cn/pkg/io/#ReaderFrom) is the interface that wraps the ReadFrom method.
            - ReadFrom reads data from r until EOF or error. The return value n is the number of bytes read. Any error except EOF encountered during the read is also returned.
        - [Writer](https://golang.google.cn/pkg/io/#Writer) is the interface that wraps the basic Write method.
            - Write writes len(p) bytes from p to the underlying data stream. It returns the number of bytes written from p (0 <= n <= len(p)) and any error encountered that caused the write to stop early. Write must return a non-nil error if it returns n < len(p). **Write must not modify the slice data, even temporarily**.
            - **Implementations must not retain p.**
            ```go
            type Writer interface {
                Write(p []byte) (n int, err error)
            }
            ```
        - **`io.Copy` 流程**
            > 其中1和2两步尝试使用零拷贝优化, 其核心思想是让数据在内核空间直接传输, 不要去“用户态”绕一圈, 以减少 CPU 在内存区域之间移动数据时的负担
            1. 检查 src 是否实现了 WriterTo(src 自己把自己写给 dst). The Copy function uses WriterTo if available.
            2. 检查 dst 是否实现了 ReaderFrom(dst 主动去把 src 读进来). The Copy function uses ReaderFrom if available.
            3. 在没有提供缓冲区时，自动创建一个大小(单位字节)合适的缓冲区
                1. 先将大小设置为`32 * 1024B = 32KB`旨在平衡内存占用和 I/O 系统调用的频率
                2. 如果`src`是`LimitedReader`类型且其N < 32KB, 则根据情况修改缓冲区大小
                    1. 如果N < 1, 将大小调整为1, 防外层循环依赖 EOF 来退出
                        > Read(make([]byte, 0)) 会直接返回 0, nil
                    2. 否则将buf设为N的大小
            4. 循环调用`Read`和`Write`实现copy功能(使用buf作为中转), 同时更新`written`统计已写字节数, 过程中除了`Read`和`Write`返回的err外, 可能存在的err有
                > 过程中遵循数据优先, 即使有err, 也先将读取的数据写入
                - errInvalidWrite: Writer 实现存在问题, 其返回的nw < 0或nw > nr
                - ErrShortWrite: nw < nr
            5. 最终返回`written`和`err`

4. 使用 `strace` 验证缓冲区行为与内核交互
    - 用`dd if=/dev/urandom of=bigfile bs=1M count=100`准备一个大文件
        > /dev/urandom 是一个 Linux 特有的特殊设备文件 (Character Special File), 是一个无限产生的高熵随机数据流, 非常适合用来模拟真实数据或测试磁盘/网络的最差性能
        - `dd`: Data Duplicator, 拷贝并转换文件
        - `if`: Input File, 输入文件
        - `of`: Output File, 输出文件
        - `bs`: Block Size, 块大小
        - `count`: Count, 计数
    - 用`strace -e trace=read,write ./echo < bigfile > /dev/null`验证上述内容
        > /dev/null 是一个字符特殊文件，它会丢弃写入其中的所有数据并报告写入成功
        - `strace`: 一个用户态实用程序，它利用内核的 ptrace 机制拦截并记录目标进程发出的 系统调用 (System Calls) 和接收到的 信号 (Signals)
        - `-e trace=read,write`: 表达式过滤器。默认情况下 strace 会输出所有系统调用（如内存分配 mmap、锁机制 futex 等）。该参数指示 strace 仅追踪和记录 read 与 write 这两个与 I/O 数据传输直接相关的系统调用
        - `./echo`: 待分析的 Go 编译产物, 其中通过 `io.Copy` 实现了标准输入到标准输出的数据复制
        - `< bigfile`: 以只读模式打开 `bigfile`, 并将其文件描述符替换目标进程的输入
        - `> /dev/null`: 将目标进程的输出重定向到`/dev/null`
    - 运行结果
        ```console
        read(3, "75 80 0:29 / /usr/lib/modules/6."..., 65536) = 2976
        read(3, "", 65536)                      = 0
        read(3, "0::/\n", 65536)                = 5
        read(3, "", 65536)                      = 0
        --- SIGURG {si_signo=SIGURG, si_code=SI_TKILL, si_pid=5904, si_uid=1000} ---
        read(0, "!@_n\347>\330\22\330\352\336\vO\347Q\27\6\343\352\354\326H\252\223\242\211|Z\207}\375\332"..., 32768) = 32768
        write(1, "!@_n\347>\330\22\330\352\336\vO\347Q\27\6\343\352\354\326H\252\223\242\211|Z\207}\375\332"..., 32768) = 32768
        ...
        read(0, "m\tS\271c,\266t/\33\316\251Iq\nO*\rd\215`5&\304}rr3\310T5j"..., 32768) = 32768
        write(1, "m\tS\271c,\266t/\33\316\251Iq\nO*\rd\215`5&\304}rr3\310T5j"..., 32768) = 32768
        read(0, "\36M\255\277\321A\322\276\240\266\341\367_\5\253\330\2557\t\267C)\7\310F\201\30\251{L\nD"..., 32768) = 32768
        --- SIGURG {si_signo=SIGURG, si_code=SI_TKILL, si_pid=5904, si_uid=1000} ---
        +++ exited with 0 +++
        ```
        1. 最开始是 Go 的 Runtime 在初始化。它正在读取 /proc/self/maps 或其他系统文件，了解自己的内存布局、线程限制等信息
        2. 32768=32*1024验证了因为程序没有自定义 buffer，所以 `io.Copy` 自动创建了一个 32KB 的缓冲区, 且返回值等于请求值，说明没有发生 Short Read/Write
        3. `--- SIGURG {si_signo=SIGURG, si_code=SI_TKILL, si_pid=5904, si_uid=1000} ---`是 Go 抢占式调度 (Preemptive Scheduling) 的机制
            > Go 运行时会向长时间运行的 Goroutine 发送这个信号，以此“打断”它，检查是否需要切换到别的 Goroutine，或者进行垃圾回收（GC）