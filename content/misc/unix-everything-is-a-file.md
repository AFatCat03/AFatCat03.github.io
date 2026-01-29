+++
date = '2026-01-25T15:37:46+08:00'
draft = false
title = 'Unix Everything Is a File'
tags = ["Unix", "OS Architecture", "Design Principles"]
+++
1. 硬件即文件
    1. 输入 `tty`(输出当前 shell 会话所绑定的终端设备路径)，查看当前终端的设备文件路径(/dev/pts/7)
    2. 打开另一个终端窗口，输入 `echo "Hello Hackers" > /dev/pts/7`
    3. 字符串直接出现在了第一个终端窗口里(对显示器的输出，本质上就是往一个设备文件里写字节)

2. 进程即文件
    1. 运行 `ps` 拿到当前 shell 的 PID(13644)
    2. 执行 ls -l /proc/1234/fd(枚举指定进程已经打开的所有文件描述符，以及它们各自指向的内核对象)
    3. 可以看到 `0`(输入), `1`(输出), `2`(错误) 这三个文件指向当前终端 `/dev/pts/7`

3. Rule of Composition: Design programs to be connected with other programs.(The Art of Unix Programming)
    - Unix tradition strongly encourages writing programs that read and write simple, textual, stream-oriented, device-independent formats. It’s because if you don’t write programs that accept and emit simple text streams, it’s much more difficult to hook the programs together.
    - To make programs composable, make them independent. A program on one end of a text stream should care as little as possible about the program on the other end.

4. File I/O (Advanced Programming in the UNIX® Environment)
    - To the kernel, all open files are referred to by file descriptors. A file descriptor is a non-negative integer. When we open an existing file or create a new file, the kernel returns a file descriptor to the process.
    - By convention, UNIX System shells associate file descriptor 0 with the standard input of a process, file descriptor 1 with the standard output, and file descriptor 2 with the standard error.
    - The file descriptor returned by open and openat is guaranteed to be the lowest numbered unused descriptor.
    - The basic idea behind  time-of-check-to-time-of-use(TOCTTOU) errors is that a program is vulnerable if it makes two file-based function calls where the second call depends on the results of the first call.
    - Every open file has an associated ‘‘current file offset,’’ normally a non-negative integer that measures the number of bytes from the beginning of the file. Read and write operations normally start at the current file offset and cause the offset to be incremented by the number of bytes read or written. By default, this offset is initialized to 0 when a file is opened, unless the O_APPEND option is specified. 
    - An open file’s offset can be set explicitly by calling lseek(`os.Seek` in Go). Because a successful call to lseek returns the new file offset, we can seek zero bytes from the current position to determine the current offset. This technique can also be used to determine if a file is capable of seeking. If the file descriptor refers to a pipe, FIFO, or socket, lseek sets errno to ESPIPE and returns −1.
        - 稀疏文件 (Sparse File): 比如虚拟机镜像（VM Disk Images）往往分配 100GB 空间，但实际只占用几百 MB 磁盘
    - The file’s offset can be greater than the file’s current size, in which case the next write to the file will extend the file. This is referred to as creating a hole in a file and is allowed. Any bytes in a file that have not been written are read back as 0. A hole in a file isn’t required to have storage backing it on disk. Depending on the file system implementation, when you write after seeking past the end of a file, new disk blocks might be allocated to store the data, but there is no need to allocate disk blocks for the data between the old end of file and the location where you start writing. (P68)
    - The read operation starts at the file’s current offset. Before a successful return, the offset is incremented by the number of bytes actually read.
    - For a regular file, the write operation starts at the file’s current offset. If the O_APPEND option was specified when the file was opened, the file’s offset is set to the current end of file before each write operation. After a successful write, the file’s offset is incremented by the number of bytes actually written.
    - The kernel uses three data structures to represent an open file, and the relationships among them determine the effect one process has on another with regard to file sharing. (P74)
        ![Kernel data structures for open files](/images/misc/unix_everything_is_a_file/1.png)
        1. Every process has an entry in the process table. Within each process table entry is a table of open file descriptors, which we can think of as a vector, with one entry per descriptor. Associated with each file descriptor are
            1. The file descriptor flags (close-on-exec; refer to Figure 3.7 and Section 3.14)
            2. A pointer to a file table entry
        2. The kernel maintains a file table for all open files. Each file table entry contains
            1. The file status flags for the file, such as read, write, append, sync, and nonblocking; more on these in Section 3.14
            2. The current file offset
            3. A pointer to the v-node table entry for the file
        3. Each open file (or device) has a v-node structure that contains information about the type of file and pointers to functions that operate on the file. For most files, the v-node also contains the i-node for the file. This information is read from disk when the file is opened, so that all the pertinent information about the file is readily available. For example, the i-node contains the owner of the file, the size of the file, pointers to where the actual data blocks for the file are located on disk, and so on.
            - V-node 是 VFS (Virtual File System) 层的抽象，它的存在是为了屏蔽不同文件系统（ext4, zfs, nfs, procfs）的差异
            - I-node 是具体文件系统（如 ext4）在磁盘上的元数据表示
            - 正因为有了 V-node 这一层统一抽象，才真正实现了 "Everything is a file" —— 无论是网络套接字（Socket）、管道（Pipe）还是真实磁盘文件，在 VFS 层看来都是一个 v-node，操作起来才有一致的 API (read/write)
        - Each process that opens the file gets its own file table entry, but only a single v-node table entry is required for a given file. One reason each process gets its own file table entry is so that each process has its own current offset for the file.
    - In general, the term *atomic operation* refers to an operation that might be composed of multiple steps. If the operation is performed atomically, either all the steps are performed (on success) or none are performed (on failure). It must not be possible for only a subset of the steps to be performed.
    - `dup`/`dup2`: The new file descriptor that is returned as the value of the functions shares the same file table entry as the fd argument.(P80)
        ![Kernel data structures after dup(1)](/images/misc/unix_everything_is_a_file/2.png)
    - `fcntl`: The fcntl function can change the properties of a file that is already open.
    - `ioctl`: The ioctl function has always been the catchall for I/O operations. Anything that couldn’t be expressed using one of the other functions in this chapter usually ended up being specified with an ioctl.(P87)
