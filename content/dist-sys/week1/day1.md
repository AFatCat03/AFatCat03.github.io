+++
date = '2026-01-11T17:59:00+08:00'
draft = false
title = 'Week1'
+++
## Day 1: 环境与工具

1. 安装 `Ubuntu 22.04 LTS`
    ```powershell
    wsl --install -d Ubuntu-22.04
    wsl --shutdown # 停止 WSL 子系统
    wsl --export Ubuntu-22.04 D:\ubuntu-backup.tar # 导出系统镜像
    wsl --unregister Ubuntu-22.04 # 注销原系统
    wsl --import Ubuntu-22.04 D:\WSL\Ubuntu2204 D:\ubuntu-backup.tar # 导入系统到新位置
    ```
    > 导入完成后删除 `D:\ubuntu-backup.tar`

2. 打开`Ubuntu 22.04 LTS`, 修改配置文件`/etc/wsl.conf`, 将默认登录用户改回安装时设置的用户名
    ```ini { title="/etc/wsl.conf" }
    [user]
    default=你的用户名
    ```
    > 保存后重启 `WSL` 生效

3. `Ubuntu`配置国内镜像源并安装基础工具
    ```bash
    sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak # 备份旧源
    sudo tee /etc/apt/sources.list <<EOF
    deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
    EOF # 一键换源 (阿里云)
    sudo apt update #更新索引
    sudo apt install build-essential gdb cmake clang # 安装基础工具
    ```

4. 打开 `VS Code`，安装 "WSL" 插件，点击左下角的蓝色按钮，选择 "Connect to WSL"

    ![](/images/week1/day1/1.png)

    > 可以在 WSL 的命令行里直接打开当前目录的 Windows 资源管理器窗口, 从而将文件从 Windows 传入 WSL (Ubuntu)
    > ```bash
    > explorer.exe .
    > ```

5. 配置 `%UserProfile%\.wslconfig`

    ```ini { title="%UserProfile%\.wslconfig" }
    [wsl2]
    networkingMode=mirrored # 镜像网络模式, Linux 和 Windows 共享同一个 IP 地址
    autoProxy=true # 强制 WSL2 自动同步 Windows 的 HTTP/HTTPS 代理设置
    memory=6GB # 限制 WSL2 虚拟机最多只能占用 6GB 的宿主机物理内存
    swap=0 # 完全禁用虚拟内存交换, 内存一满直接触发 OOM Killer
    localhostForwarding=true # 允许通过 Windows 的 localhost 访问 WSL2 的端口
    ```

6. 使用 `dmesg` 查看内核环形缓冲区日志
    > `.wslconfig` 中的 `memory=6GB` 硬限制已准确传递给 Linux 内核, 其中系统启动时内核预留了约 `228MB (233800K reserved)` 用于代码段、页表和 BIOS 映射
    > 
    > Hyper-V 的 Balloon 驱动: 负责在宿主机和虚拟机之间动态调度物理页，确保在低负载时不浪费 Windows 内存

    ```console { title="dmesg | grep Memory" }
    [    0.075345] Memory: 4123740K/6290044K available (20480K kernel code, 3457K rwdata, 13496K rodata, 4492K init, 6208K bss, 233800K reserved, 0K cma-reserved)
    [    0.147753] x86/mm: Memory block size: 128MB
    [    0.349717] hv_balloon: Using Dynamic Memory protocol version 2.0
    ```

7. 安装 `Go`
    ```bash
    wget https://go.dev/dl/go1.25.5.linux-amd64.tar.gz
    sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.25.5.linux-amd64.tar.gz # 清理旧版本并解压新版本
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc 
    source ~/.bashrc # 配置环境变量
    go version
    ```

8. 使用 `vimtutor` 熟悉 `Vim`
    > VSCode快速切换到代码区`CTRL+1`,快速切换到终端`CTRL+\``
    - MOVING THE CURSOR 
        - `h`: left
        - `l`: right
        - `j`: down
        - `k`: up
    - START VIM: `vim FILENAME <ENTER>`
    - EXITING VIM
        - `<ESC>   :q!   <ENTER>`  to trash all changes
        - `<ESC>   :wq   <ENTER>`  to save the changes
    - TEXT EDITING - DELETION
        - `x`: delete the character under the cursor
    - TEXT EDITING - INSERTION
        - `i`: insert before the cursor
    - TEXT EDITING - APPENDING
        - `A`: append after the line
    - Pressing `<ESC>` will place you in Normal mode or will cancel an unwanted and partially completed command.
    - DELETION COMMANDS
        - `dw`: delete from the cursor up to the next word type
        - `de`: delete from the cursor up to the end of the word type. (If the cursor is already at the end, it deletes to the end of the *next* word)
        - `d$`: delete from the cursor to the end of a line type
        - `dd`: delete a whole line type
        - 当光标在词尾时,`de`会删除至下一个单词词尾(包含词尾),因为定义中的`forward`的特性
            > ```console { title="motion.txt" }
            > 385 w                       [count] words forward.  |exclusive| motion
            > ...
            > 393 e                       Forward to the end of word [count] |inclusive|.
            > 394                         Does not stop in an empty line.
            > ...
            > 416 A word consists of a sequence of letters, digits and underscores, or a
            > 417 sequence of other non-blank characters, separated with white space (spaces,
            > 418 tabs, <EOL>).  This can be changed with the 'iskeyword' option.  An empty line
            > 419 is also considered to be a word.
            > ``` 
    - To repeat a motion prepend it with a number:   `2w`
    - The format for a change command is: `operator   [number]   motion`
        - operator: is what to do, such as `d` for delete
        - [number]: is an optional count to repeat the motion
        - motion: moves over the text to operate on, such as  `w` (word),  `e` (end of word),  `$` (end of the line), etc.
    - To move to the start of the line use a zero:  `0`
    - UNDO COMMAND
        - `u`: undo previous actions
        - `U`: undo all the changes on a line
        - `CTRL-R`: undo the undo's
    - PUT COMMAND
        - `p`: put the deleted text AFTER the cursor (if a line was deleted it will go on the line below the cursor)
    - REPLACE COMMAND
        - `r`: replace the character at the cursor with the character you want to have there
    - CHANGE OPERATOR
        - `c   [number]   motion`
        - change from the cursor to where the motion takes you (delete and place you in Insert mode)
        - `ce`: change from the cursor to the end of the word
        - `c$`: change to the end of a line
        - `cc`: change the whole line
    - CURSOR LOCATION AND FILE STATUS
        - `CTRL-G`: display your location in the file and the file status
        - `G`: move to the end of the file
        - `number  G`: move to that line number
        - `gg`: move to the first line
    - SEARCH COMMAND
        - `/phrase`: search FORWARD for the phrase
        - `?phrase`: search BACKWARD for the phrase
        - After a search
            - `n`: find the next occurrence in the same direction
            - `N`: find the next occurrence in the opposite direction
    - `CTRL-O`: back to older position in jump list
    - `CTRL-I`: forward to newer position in jump list
    - MATCHING PARENTHESES SEARCH: Typing  `%`  while the cursor is on a `(`,`)`,`[`,`]`,`{`, or `}` goes to its match
    - SUBSTITUTE COMMAND
        - `:s/old/new`: substitute new for the first old in a line type
        - `:s/old/new/g`: substitute new for all 'old's on a line type
        - `:#,#s/old/new/g`: substitute phrases between two line #'s type
        - `:%s/old/new/gc`: ask for confirmation each time add 'c'(confirm)
            - `y`: yes, 替换当前这一个
            - `n`: no, 跳过当前这一个，不替换，继续找下一个
            - `a`: all, 替换当前这个，并且后续所有的都不再询问（相当于剩下的自动全选 Yes）
            - `q`: quit, 停止替换操作, 回到Normal mode
            - `l`: last, 替换当前这一个，然后立即停止
            - `^E`: `CRTL+E`, 屏幕向下滚动一行
            - `^Y`: `CRTL+Y`, 屏幕向上滚动一行
    - `:!command`: execute an external command
        - `:!ls`: show a directory listing
    - `:w FILENAME`: write the current Vim file to disk with name FILENAME
    - `v  motion  :w FILENAME`: save the Visually selected lines in file FILENAME
    - `:r FILENAME`: retrieve disk file FILENAME and put it below the cursor position
    - `:r !ls`: read the output of the dir command and put it below the cursor position
    - Visual Mode
        - `v`: 按字符选择 (Character-wise), 选中句子的一部分, 或者跨行的不规则文本段
        - `V`: 按行选择 (Line-wise), 瞬间选中光标所在的整行, 继续上下移动会整行整行地扩展选择
        - `CTRL-V`/`CTRL-Q`: 按块选择 (Block-wise), 进行纵向列选择
    - `o`: open a line BELOW the cursor and start Insert mode
    - `O`: open a line ABOVE the cursor and start Insert mode
    - `i`: insert text BEFORE the cursor
    - `a`: insert text AFTER the cursor
    - `A`: insert text after the end of the line
    - `e`: move to the end of a word
    - `y`: yank into a register
    - `d`: delete into a register
    - `c`: delete + insert
    - `p`: put AFTER (字符在后，行在下), 对应 `a`
    - `P`: put BEFORE (字符在前，行在上), 对应 `i`
    - 写入（`y`/`d`/`c`） → 持有 → 读取（`p`） → 仍然持有
    - `R`: enter Replace mode until  `<ESC>`  is pressed
    - `:set xxx`: set the option "xxx"
        - 通常对当前打开的所有窗口有效（除非使用了 `:setlocal`）
        - 有效期直到关闭 Vim 程序
        - `ic`/`ignorecase`: ignore upper/lower case when searching
        - `is`/`incsearch`: show partial matches for a search phrase
        - `hls`/`hlsearch`: highlight all matching phrases
    - Prepend "no" to switch an option off: `:set noic`
    - `:help`/`<F1>`: open a help window
    - `:help cmd`: find help on `cmd`
    - `CTRL-W CTRL-W`: jump to another window
    - `:q`: close the help window
    - Create a vimrc startup script to keep your preferred settings
    - When typing a `:` command, press `CTRL-D` to see possible completions. Press `<TAB>` to use one completion

9. 配置持久化 `~/.vimrc`
    ```vim { title="在vimrc_example.vim基础上增加" }
    set number        " 显示行号，调试必备
    set syntax=on     " 开启语法高亮
    set tabstop=4     " Tab 宽度
    set autoindent    " 自动缩进
    ```

10. 体验 Linux 数据流的传递
    - `cat`
        - `cat filename.txt`: 查看文件内容
        - `cat file1.txt file2.txt > merged.txt`: 合并文件
        - `cat > note.txt`: 快速创建/写入文件, `Ctrl+D`(EOF)保存并退出
    - `ls` 接到参数后，默认会先对参数名进行字母排序 (A-Z)，然后再开始处理
    - 文件描述符 FD (File Descriptor) 是一个非负整数（如 0, 1, 2），它作为进程私有文件描述符表的索引，用于在用户空间中引用内核中的打开对象。每个 FD 在内核中对应一个 struct file（打开文件对象），该对象记录了访问模式、当前读写偏移量以及具体的文件操作方法。struct file 进一步关联到具体的内核资源，例如 VFS inode（针对普通文件、目录和设备）或 socket、pipe 等其他内核对象，从而实现用户态对底层资源的统一访问接口。对于运行在 Linux（或 Unix/macOS）系统上的进程, 有以下 FD:
        - Stdin (FD 0): 标准输入
        - Stdout (FD 1): 标准输出
        - Stderr (FD 2): 标准错误
    - `|`: 将前一个进程的FD1连接到后一个进程的FD0
    - **重定向从左到右顺序执行, Copy by Value**
    - `>`: 重定向FD1(Stdout), 覆盖
    - `>>`: 重定向FD1(Stdout), 追加
    - `2>`: 重定向FD2(Stderr)
    - `N>`: 重定向FDN
    - `M>&N`: 将文件描述符 $M$ 重定向到 文件描述符 $N$ 当前所指向的 那个内核对象
    - `M>&-`: 关闭文件描述符 $M$