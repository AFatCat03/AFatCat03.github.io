+++
date = '2026-01-19T13:08:22+08:00'
draft = false
title = 'Day4'
+++
## Day 4: 跨越内核屏障（文件与资源管理）

1. 内核句柄抽象：FD 与系统调用 (Kernel Handles: FDs & Syscalls)
    - [`os.Open`](https://golang.google.cn/pkg/os/#Open): `func Open(name string) (*File, error)`
        - Open opens the named file for reading. If successful, methods on the returned file can be used for **reading**; the associated file descriptor has mode O_RDONLY. If there is an error, it will be of type *PathError.(And *File = nil)
    - [File](https://golang.google.cn/pkg/os/#File) represents an open file descriptor. 
        - The methods of File are safe for concurrent use.
        ```go
        type File struct {
            *file // os specific
        }
        ```
        - Go 编译器会根据编译的目标平台（Linux, Windows, macOS）, 自动把 file 指向不同的实现代码(Bridge Pattern)
            > 因为Linux 和 Windows 对文件的定义完全不同
            > 
            > - Linux: 文件是一个 int 类型的 File Descriptor (FD)
            > - Windows: 文件是一个 uintptr 类型的 Handle
            - 在 Linux 下编译时，编译器会选用[file_unix.go](https://golang.google.cn/src/os/file_unix.go)里的定义
                ```go
                // file is the real representation of *File.
                // The extra level of indirection ensures that no clients of os
                // can overwrite this data, which could cause the finalizer
                // to close the wrong file descriptor.
                type file struct {
                    pfd         poll.FD
                    name        string
                    dirinfo     atomic.Pointer[dirInfo] // nil unless directory being read
                    nonblock    bool                    // whether we set nonblocking mode
                    stdoutOrErr bool                    // whether this is stdout or stderr
                    appendMode  bool                    // whether file is opened for appending
                }
                ```
    - Linux下的file中的pfd为poll.FD类型, 可以通过[fd_unix.go](https://golang.google.cn/src/internal/poll/fd_unix.go)查看其中内容
        ```go
        // FD is a file descriptor. The net and os packages use this type as a
        // field of a larger type representing a network connection or OS file.
        type FD struct {
            // Lock sysfd and serialize access to Read and Write methods.
            fdmu fdMutex

            // System file descriptor. Immutable until Close.
            Sysfd int

            // Platform dependent state of the file descriptor.
            SysFile

            // I/O poller.
            pd pollDesc

            // Semaphore signaled when file is closed.
            csema uint32

            // Non-zero if this file has been set to blocking mode.
            isBlocking uint32

            // Whether this is a streaming descriptor, as opposed to a
            // packet-based descriptor like a UDP socket. Immutable.
            IsStream bool

            // Whether a zero byte read indicates EOF. This is false for a
            // message based socket connection.
            ZeroReadIsEOF bool

            // Whether this is a file rather than a network socket.
            isFile bool
        }
        ```
        - 其中 Sysfd 就是 Linux 内核识别资源的唯一 ID 
    - 对于`os.Open`获取的File, 可以通过[`func (f *File) Fd() uintptr`](https://golang.google.cn/pkg/os/#File.Fd)方法获取fd
        - 该方法内部是调用file的[`func (f *File) fd() uintptr`](https://golang.google.cn/src/os/file_unix.go)方法
        ```go
        // fd is the Unix implementation of Fd.
        func (f *File) fd() uintptr {
            if f == nil {
                return ^(uintptr(0))
            }

            // If we put the file descriptor into nonblocking mode,
            // then set it to blocking mode before we return it,
            // because historically we have always returned a descriptor
            // opened in blocking mode. The File will continue to work,
            // but any blocking operation will tie up a thread.
            if f.nonblock {
                f.pfd.SetBlocking()
            }

            return uintptr(f.pfd.Sysfd)
        }
        ```
        - uintptr(Unsigned Integer Ptr)本质上是一个整数, 不是指针
            - 大小自适应: 在 32位 系统上是 32-bit (4字节); 在 64位 系统上，是 64-bit
                > 任何一个指针（unsafe.Pointer 或 *int）都可以被强转成 uintptr 变成一个纯数字；反之亦然
            - Fd() 返回 uintptr 是为了 跨平台兼容性(既大到足以装下 Windows 的 64位指针句柄，又能轻松装下 Linux 的 32位整数 ID)
            - **uintptr 仅仅是内存地址的数值表示，它会斩断 GC 的引用追踪链，可能导致对象在仍被使用时被错误回收或因栈移动而失效**
    - **Go 的 I/O 操作最终会委托给 internal/poll 包，该包会根据文件类型和操作系统，选择调用 syscall.Read（阻塞式）或者通过 epoll/kqueue 机制（非阻塞式）来调度**, 先前拿到的FD是进程级的文件描述符表的一个索引, 这意味着绕过了 Go 语言标准库 (os 包) 的层层封装，获得了直接与内核对话的资格, 有以下3个能力: 
        - 跨进程传递 (Inheritance & Passing)(这是 Unix 系统最强大的特性之一): FD 是可以被“继承”或“发送”的, 可以创建一个子进程，然后把这个 FD 扔给它, 即使子进程没有权限打开那个文件（比如它是低权限进程），但只要你把已经打开的 FD 给它，它就能读写(FD 代表了一种“已授权的访问能力” (Capability))
        - 执行“法外”操作 (Syscalls): Go 的 os.File 只封装了通用的 Read/Write/Close 等操作。但文件系统还有很多操作 Go 并没有完全封装。 一旦拿到了 FD，就可以利用 [syscall 包](https://golang.google.cn/pkg/syscall/)直接对它发号施令
            - syscall.Flock(fd, ...): 给文件加系统级互斥锁
            - syscall.Fsync(fd): 强迫硬盘立即把数据刷入磁道（不经过缓存）
            - syscall.Syscall(SYS_IOCTL, fd, ...): 发送设备控制指令（比如控制终端窗口的大小，或者弹出光驱）
        - 绕过 Go 的缓冲区: Go 的 bufio.Reader 或者某些 net.Conn 实现内部可能有缓存。 拿着 FD 直接去调 syscall.Read 是直接从内核缓冲区搬运数据，完全无视 Go 语言层面的任何缓冲机制。这在需要极低延迟或零拷贝的场景下非常有用
    - **FD 是有限的内核资源, 每个进程的文件描述符表的大小是有限的, 如果忘记调用`close()`, Go 运行时的对象（struct）可能被销毁了，但在操作系统内核层面，那个 Socket 依然处于 "ESTABLISHED" 或 "CLOSE_WAIT" 状态，对应的 FD 依然被占用, 最终导致再次请求在FD表中分配空闲位置时, 内核直接拒绝分配，返回错误码 EMFILE (Error: Too many open files)**

2. Linux下`os.Open`调用过程
    1. [`func Open(name string) (*File, error)`](https://golang.google.cn/src/os/file.go?s=11635:11672#L379)
    2. [`func OpenFile(name string, flag int, perm FileMode) (*File, error)`](https://golang.google.cn/src/os/file.go?s=12650:12716#L400)
    3. [`func openFileNolog(name string, flag int, perm FileMode) (*File, error)`](https://golang.google.cn/src/os/file_unix.go#L243)
    4. [`func open(path string, flag int, perm uint32) (int, poll.SysFile, error)`](https://golang.google.cn/src/os/file_open_unix.go)
    5. [`func Open(path string, mode int, perm uint32) (fd int, err error)`](https://golang.google.cn/src/syscall/syscall_linux.go#L279)
    6. [`func Openat(dirfd int, path string, flags int, mode uint32) (fd int, err error)`](https://golang.google.cn/src/syscall/syscall_linux.go#L285)
    7. [`func openat(dirfd int, path string, flags int, mode uint32) (fd int, err error)`](https://golang.google.cn/src/syscall/zsyscall_linux_amd64.go#L92)
    8. 通过[`Syscall6`](https://golang.google.cn/src/syscall/syscall_linux.go#L95)等方法调用Linux内核的`sys_openat`函数

3. 资源安全性与 Defer 栈语义 (Resource Safety & Defer Semantics)
    - 阅读[Defer, Panic, and Recover](https://go.dev/blog/defer-panic-and-recover)
        - A deferred function’s arguments are evaluated when the defer statement is evaluated.
        - Deferred function calls are executed in Last In First Out order after the surrounding function returns.
        - Deferred functions may read and assign to the returning function’s named return values.
            > return并非原子操作:
            >
            >   1. 赋值 (Assignment): 将结果写入返回值在栈帧中的内存地址（如果是具名返回值，这个地址早就分配好了）
            >
            >   2. 执行 Defer (Defer Execution): 运行该函数注册的所有 defer 链表（按 LIFO 顺序）
            >
            >   3. 返回指令 (RET Instruction): 携带栈上的返回值，跳回调用方
            > 
            > Thus, this function returns 2:
            >
            > ```go
            > func c() (i int) {
            >     defer func() { i++ }()
            >     return 1
            > }
            > ```
    - 阅读[Effective Go 的 Defer 部分](https://go.dev/doc/effective_go#defer)
        - Go's defer statement schedules a function call (the deferred function) to be run immediately before the function executing the defer returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return.
        - The arguments to the deferred function (which include the receiver if the function is a method) are evaluated when the *defer* executes, not when the *call* executes.
    - `defer` 必须放在错误检查（`if err != nil`）**之后**, 因为如果没有创建成功, 返回的*File类型变量值为nil, 执行Close()方法会导致panic(换句话说, 过程是: 申请 (Acquire) -> 检查 (Check) -> 持有 (Hold) -> 释放 (Release), 没通过检查的话就没有持有, 更不用谈释放)

4. I/O 策略：缓冲与零拷贝权衡 (I/O Strategies: Buffering vs. Streaming)
    - [`io.Copy`](https://golang.google.cn/pkg/io/#Copy): 占用固定大小的内存 (通常 32KB)
    - [`io.ReadAll`](https://golang.google.cn/pkg/io/#ReadAll): 将内容一次性全部读入内存
    - Unbuffered I/O(每一条数据的实时落地安全性): 直接交互：每一次读写操作都会直接触发一个 系统调用 (System Call)，直接陷入内核态
    - Buffered I/O(系统的整体吞吐量): 中间层：在用户空间维护了一块内存区域（Buffer）。读写操作先打给这个 Buffer，等 Buffer 满了 或者 遇到换行符（行缓冲）或者 手动 Flush 时，才发起一次大的系统调用，把数据批量发给内核
        - `io.Copy` 内部实现了一种临时的、最优粒度的 Buffered I/O 策略，用来平衡内存占用和系统调用次数
        - `io.ReadAll` 是一种极端的缓冲策略——无限缓冲

5. 工程实践：CLI 工具与错误处理规范 (Implementation: CLI & Error Handling Standards)
    - [`fmt.Fprintln`](https://golang.google.cn/pkg/fmt/#Fprintln): Fprintln formats using the default formats for its operands and writes to w. Spaces are always added between operands and a newline is appended. It returns the number of bytes written and any write error encountered. encountered. 可以明确指定输出目标
    - [`os.Exit`](https://golang.google.cn/pkg/os/#Exit): Exit causes the current program to exit with the given status code. Conventionally, code zero indicates success, non-zero an error. The program terminates immediately; deferred functions are **not** run. For portability, the status code should be in the range [0, 125].
    - `_`: 空标识符 (Blank Identifier), 是只写的 (Write-only)。任何赋值给它的数据，都会被直接丢弃，无法再被读取
    - `echo $?`: Linux/WSL 查看上一步的退出码
    > 打开指定路径的文件并将其内容流式传输到 Stdout
    ```go
    package main

    import (
        "fmt"
        "io"
        "os"
    )

    // printFile 打开指定路径的文件并将其内容流式传输到 Stdout
    func printFile(filename string) error {
        f, err := os.Open(filename)
        if err != nil {
            return err
        }
        defer f.Close()
        _, err = io.Copy(os.Stdout, f)
        return err
    }

    func main() {
        if len(os.Args) < 2 {
            fmt.Fprintln(os.Stderr, "Usage: printFile filename")
            os.Exit(1)
        }
        filename := os.Args[1]
        if err := printFile(filename); err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
        }
    }

    ```
