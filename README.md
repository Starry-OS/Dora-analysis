# Linux APP On Starry
## 背景

- 项目地址：https://github.com/orgs/Starry-OS/projects/1/views/4

- 运行 Dora 到 Starry 上，最终支持智能化小车

  - Starry 代码：https://github.com/Starry-OS/Starry/

  - Dora 代码：https://github.com/dora-rs/dora

  - Dora 以 musl-libc 编译的 x86_64 执行版本：https://github.com/Starry-OS/Starry/releases/tag/dora

- 支持 Tokio On Starry：

  - 支持 Tokio 应用运行在 Starry 上
  - 以 Tokio 提供的 example 作为测试标准：https://github.com/tokio-rs/tokio/tree/master/examples

## Dora 支持

### 运行流程说明

- Dora 分为三部分，分别是 coordinator，daemon，cli
- coordinator 启动之后，自动监听 53290 端口，等待 daemon 连入
- daemon 启动之后，向 53290 端口发送注册信息，告知自己所在的 IP 和端口，由 coordinator 确认注册成功
- coordinator 还会监听 6012 端口，等待接受 CLI 信息
- CLI 向 coordinator 发送一个 start 数据流，其中包括了需要启动的多个 node 的信息和他们所在的可执行文件路径
- coordinator 接收到数据流信息之后，搜索 node 运行在哪个 daemon 上，将需要启动的 node 信息和所在路径传递给对应的 daemon，称为 spawn dataflow
- daemon 接收到 spawn 数据之后，找到 node 所需要的可执行文件并且启动，若启动成功则向 coordinator 回传成功信息和 node 数据
- coordinator 接收到 All finished 信息之后向 CLI 传递数据，完成一次交互

### 运行 dora

- rust-dataflow 源码：https://github.com/dora-rs/dora/tree/main/examples/rust-dataflow

- 本地运行方式：在 dora 主仓库根目录下运行：

  ```sh
  # 运行 Dora
  $ cargo build -p dora-cli
  $ ./target/release/dora -h
  
  # 运行 Dora-benchmark
  $ cd {workspace} # 这里的 workspace 是自选的工作目录
  $ git clone https://github.com/dora-rs/dora-benchmark.git
  $ cd dora-benchmark
  $ cd dora-rs/rs-latency
  $ cargo build -p benchmark-example-node --release
  $ cargo build -p benchmark-example-sink --release
  $ cp {dora-pwd} ./ # 将第一步编译得到的 dora 的可执行文件放在 rs-latency 文件夹下
  $ ./dora up
  $ ./dora start dataflow.yml
  ```
  
- 如何将 dora 放到 Starry 上运行

  - 在 Starry-OS 主仓运行如下指令：

    ```sh
    $ git submodule init
    $ git submodule up
    ```

  - 本地得到 testcases 文件夹之后，在`testcases/x86_64_dora`文件夹下创建 `x86_64_dora` 文件夹，将 https://github.com/Starry-OS/Starry/releases/download/dora/x86_64_dora.tar.gz 下载并且解压在这个文件夹下

  - 运行如下指令

    ```sh
    $ ./build_img.sh -a x86_64 -file x86_64_dora 
    $ make A=apps/monolithic_userboot LOG=off FEATURES=img,sched_rr ACCEL=n run
    $ ./dora -h
    $ ./dora up
    ```

### 相关文件

- `Dora up` 指令在 linux 上运行得到的`starce` 结果：[up-log](./log/dora-up.txt)

- `Dora-benchmark-rs-latency`跟踪指令

  - 生成方式：

    ```sh
    # 在 rs-latency 目录下
    $  ./dora up
    # 查看 dora daemon 所在进程
    $ ps -a
    # 假设 daemon 所在进程为 daemon-pid
    $ strace -p {daemon-pid} -o daemon.txt -v -f -F
    # 新开一个进程窗口
    # 假设 coordinator 所在进程为 coordinator-pid
    $ strace -p {coordinator-pid} -o daemon.txt -v -f -F
    # 新开一个进程窗口
    $ strace -v -f -F -o ./cli.txt ./dora start ./dataflow.yml --attach
    ```

  - 原理描述：dora start 会启动一个 CLI，读取 yml 内容并且向 coordinator 发送，coordinator 会转发给 daemon，由 daemon 去真正运行 node 对应的可执行文件（规定在 dataflow.yml）中。因此我们需要同时检测 daemon 和 CLI 的系统调用内容，结果分别存储在`daemon.txt` 和 `cli.txt` 下

  - 结果：
  
    - daemon 分析结果：[daemon](./log/daemon.txt)
    - cli 分析结果：[cli](./log/cli.txt)
    - coordinator 分析结果：[coordinator](./log/coordinator.txt)

### Dora Syscall 分析支持需求

以下 syscall 的 flags 可以查看 https://filippo.io/linux-syscall-table/ 得到具体的 syscall 语义说明，文档解释的可能不够完整和正确。

1. `syscall_clone` 支持`CLONE_FS`和`CLONE_SIGHAND`

   1. `CLONE_FS`需要做到父子进程之间工作目录共享，还需要后续对于 `chdir`，`umask` 的修改是共享的
   2. `CLONE_SIGHAND`：继承信号处理函数，但是不继承掩码和等待信号集

2. `futex` 支持 `FUTEX_WAIT_PRIVATE`，`FUTEX_WAKE_PRIVATE`，`FUTEX_WAIT_BITSET_PRIVATE`，`FUTEX_WAKE_BITSET_PRIVATE`

   1. `FUTEX_WAIT_PRIVATE` 需要保证进程间不共享，现有做法可能导致 page fault
   2. `FUTEX_WAIT_BITSET_PRIVATE`：本意是为了筛选部分任务进行唤醒，但是常见情况掩码为 `FUTEX_BITSET_MATCH_ANY` ，此时和`FUTEX_WAIT`无区别

   已有工作可看：https://github.com/Josen-B/linux_syscall_api/commits/main/。

3. `epoll_create1` 支持`EPOLL_CLOEXEC`：将 `epoll`文件作为一个通用文件类型来实现其泛型

4. `epoll_ctl` 支持`EPOLLRDHUP` 和`EPOLLET`

   1. `EPOLLRDHUP`：用于监视 socket 文件描述符的关闭事件。具体来说，当对方关闭连接（半关闭）或关闭一半的读操作时，会触发 `EPOLLRDHUP` 事件。
      在实际应用中，`EPOLLRDHUP` 主要用于检测对端关闭连接的情况，这在长连接（如 HTTP 长连接或 WebSocket 连接）中尤为重要。使用 `EPOLLRDHUP` 可以有效地检测到对端关闭连接，避免资源浪费或处理无效连接。
	    如果不好实现，可以先考虑绕过。

   2. `EPOLLET`：触发方式改为边缘触发。在这种模式下，当文件描述符的状态发生变化时，只会通知一次，直到下一次状态变化。如果实现较为复杂可以考虑绕过

5. `socketpair`支持`SOCK_STREAM`，`SOCK_CLOEXEC`，`SOCK_NONBLOCK`：将 socketpair 作为一对`socket`即可，部分属性已经在`socket`实现了

6. `sendto`对于`MSG_NOSIGNAL`支持：当对端关闭时不需要发送`SIG_PIPE`信号，可以考虑绕过不支持

7. `fcntl` 支持 `F_SETFL` 与 `F_GETFL`：支持修改 file 的 status 和 mode 

8. `rseq` 支持： 应该可以是绕过的

9. `fsync`：支持文件同步，由于 Starry 直接调用修改文件，所以应该可以不考虑支持

10. `prlimit` 需要支持设置栈大小：柏乔森已经实现，考虑提 PR 合入主线

11. `dev/shm` 支持：`dev/shm` 是 unix 系统下用内存页替代文件的一个实现，在里面创建的文件以及读写操作均只会在内存页上进行，当页被释放则操作则全部失效。对应在 ArceOS 上即是将 ArceOS 原有的 `ramfs` 挂载到`dev/shm`目录下即可，不过需要对文件读写等进行判断，查看是否是读写`dev/shm`下文件。


### Dora 任务和细节划分

预期花费**两周时间**支持 Dora example rust-dataflow

Update time: 2024-7-5

- 郑友捷：
  - 分析 rust-dataflow example，做好**后续进一步分工**
  - 修改 Starry 逻辑，支持 ramdisk 启动 dora，加快 IO 速度
  - 支持 rust-dataflow example
  
- 李昕昊：上述工作的 2 部分，支持 Futex 相关 flags，可以使用 ltp 等测例进行测试，**需要在此基础上完善已有测例，真正测试功能正确性**（ltp 较多的测试参数准确程度，具体的功能正确与否还要确定）。预期时间：3 天

- 柏乔森：将 ZLM 改动通过 PR 等形式合入到 Starry 中，并且上述工作的第 7 部分，支持 `fcntl` 对应的 flags。预期时间：3 天。注意：合入过程中需要时刻运行 dora 或者已有测例判断功能正确性

- 杨金全：上述工作的 3、4 部分：即 epoll file 和 socket 对文件操作的支持，这个可以参考已有的文件类型如 `file` 进行，并且可以使用 ltp 或者自己编写的测例进行验证。预计时间：3 天

  

## Tokio 支持

### 相关文件说明

相关源码来自于https://github.com/tokio-rs/tokio/tree/master/examples，编译成的 x86_64 的 musl libc 文件来自于https://github.com/Starry-OS/Starry/releases/tag/tokio。

其中：

- https://github.com/Starry-OS/Starry/releases/download/tokio/x86_64_tokio_musl_debug.tar.gz 为测例集的 debug 版本，由于文件较大不能完整打包到 Starry 中，建议每次打包本次运行所需文件作为镜像进行加载。
- https://github.com/Starry-OS/Starry/releases/download/tokio/x86_64_tokio_musl_release.tar.gz 为测例集的 release 版本，可以整体打包。

`batch`文件是另外编写的、方便同时运行两个文件并且进行输入交互的可执行文件，其源码如下：

```rust
//! Run the server and the client at the same time.

fn main() {
    // Receive an argument from the command line as the server name
    let server_name = std::env::args().nth(1).unwrap();
    // Run the excutable file of the server
    let mut server = std::process::Command::new(server_name)
        .spawn()
        .expect("failed to start server");

    // Receive the remaing arguments from the command line as the client name
    let client_args: Vec<String> = std::env::args().skip(2).collect();
    // Run the excutable file of the client
    let client_name = "./connect";
    let mut client = std::process::Command::new(client_name)
        .args(client_args)
        .spawn()
        .expect("failed to start client");

    // Wait for the server to finish
    server.wait().expect("server failed");
    // Wait for the client to finish
    client.wait().expect("client failed");
}
```

运行格式如下：

```sh
$ ./batch [server_name] [client_args]
```

其中第一个参数为执行监听服务的可执行文件的名称，后续的参数均为执行客户端功能的可执行文件所需的参数。

运行示例为：

```sh
$ ./batch ./echo 127.0.0.1:8080
# 等价于运行
# 在一个终端中运行： ./echo
# 在另一个终端中运行：./connect 127.0.0.1:8080
```

### 运行方式

在 `Starry/apps/monolithic_userboot/src/batch.rs:66`的`OTHER_TESTCASES`修改即将运行的程序与参数，可供选择的参数包括但不限于：

```sh
$ ./batch ./echo 127.0.0.1:8080
$ ./batch ./tinydb 127.0.0.1:8080
$ ./batch ./print_each_packet 127.0.0.1:8080
```

另外还可通过本地主机和 QEMU 中分别运行服务端和客户端的方式进行测试，对应测试为：

```sh
# 主机上运行：
$ netcat -l -p 6142
# QEMU 上选定 OTHER_TESTCASES="./hello_world"
$ make A=apps/monolithic_userboot LOG=info FEATURES=img,sched_rr ACCEL=n APP_FEATURES=batch run NET_DUMP=y
```

### 需求分析

以下的工作可以对照 Linux 主机使用 strace 系统调用分析其不同之处。

1. TCP 支持 `SO_REUSEADDR`选项。

   `SO_REUSEADDR` 选项有以下几个作用：

   1. **允许端口重用**：当一个 TCP 连接关闭后，连接的端口会进入 `TIME_WAIT` 状态，此时端口虽然没有被使用，但仍然不能立即重新绑定。启用 `SO_REUSEADDR` 选项后，可以在该端口仍处于 `TIME_WAIT` 状态时，重新绑定该端口。
   2. **允许多个套接字绑定到同一个端口**：在某些多播协议的应用中，多个套接字可以绑定到同一个端口，使用 `SO_REUSEADDR` 可以实现这一点。
   3. **解决服务器重启绑定端口的问题**：在服务器程序重启时，如果上一次运行的服务器占用了某个端口且该端口尚未完全释放，启用 `SO_REUSEADDR` 选项可以让新启动的服务器成功绑定到同一个端口。

   之前在 UDP 上支持了 `SO_REUSEADDR`，见该 PR：https://github.com/Arceos-monolithic/Starry/pull/37/files，可以用作 TCP 支持 `SO_REUSEADDR` 参考。

   测试方式：可以根据第一和二个点设计测例，将多个套接字绑定到同一个端口，或者重用一个端口。

2. `Futex`返回参数判断：该错误在 RELEASE 版本的可执行文件不会出现，但是对于 DEBUG 版本的文件，会做更多的检查。此时 Futex 原先的调用检查不甚完善会导致报错。

   可以通过 LTP 测例来进行检查。测试方式：`echo`的 `debug`版本不会在进入 socket 之前报错。

3. `STDIN`输入多个字符、READ 系统调用规范化：原先的标准输入是为了支持 Busybox 存在，默认每一次读入只会返回一个字符。但是对于 Tokio 的 `tinydb`而言，每次读入返回需要判断是否为一个有效的请求，并且需要按照系统调用的格式返回有效的数据，否则 `tinydb` 会卡死在 READ 系统调用。

   测试方式：`./batch ./tinydb 127.0.0.1:8080`可以正确完成输入并且发送请求

4. 终端支持正常回显：对于`echo`测例，进行输入之后会将输入发送给对方，并且 recv 同一个字符输出到标准输出，但是Starry 目前是逐个输入立即返回，所以没有办法区分换行，体现为输入之后立即被输出覆盖。并且回车也没有正常换行。

   测试方式：` ./batch ./echo 127.0.0.1:8080`，期望在打入一行输入按下回车之后可以在新的一行打出输出。

5. 支持和主机通信：Starry 已经支持和同局域网下的其他主机通信，但是目前不支持和所在的主机（IP 为 10.0.2.2）通信。

   `hello_world`测例运行方式：

   - 在主机上运行：

     ```sh
     $ netcat -l -p 6142
     ```
   
   
      - 在 QEMU 中将 Starry 执行指令修改为：`./hello_world`，并且运行如下指令：    
   
        ```sh
            $ make A=apps/monolithic_userboot LOG=info FEATURES=img,sched_rr ACCEL=n APP_FEATURES=batch run NET_DUMP=y
        ```
   
        这里的`hello_world`是修改了源码，将目标地址从`127.0.0.1:6142`改为了`10.0.2.2:6142`，从而能够让 QEMU 访问主机上某一个端口。（原理是 QEMU 自己建立了一个子网，10.0.2.15是它的地址，10.0.2.2 是网关（本机）的地址）

        在当前的 Starry，他会在第一次 connect 返回 EINPROGRESS 之后进入 Futex，但是始终不会解锁。分析 Linux 上该程序的表现发现虽然也是返回 EINPROGRESS，但是 Futex 不久之后就会解锁，从而完成链接。这里关键是解决 Futex 为何不解锁的问题。

        另外还有一个测例：`echo_outside`，运行方式为：在主机上运行：

        ```sh
        $ ./connect 127.0.0.1:8080
        ```

        在 QEMU 中将 Starry 执行指令修改为：`./echo_outside`，并且运行如下指令：

        ```sh
        $ make A=apps/monolithic_userboot LOG=info FEATURES=img,sched_rr ACCEL=n APP_FEATURES=batch run NET_DUMP=y
        ```

        其中`echo_outside`是将 `echo.rs` 的源码的目标地址从 127.0.0.1 修改为 10.0.2.2 之后编译成的可执行文件。在主机上进行输入，期望能够在按下回车之后将输入内容重新输出。但 Starry 目前无法完成链接。
   

### 目前 example 支持情况

由于 Starry 当前仅支持单个网络设备，因此交替使用本地回环和以太网设备会导致丢包问题。单独使用本地回环或者以太网均不会导致问题的出现。

以下的测试均通过本地回环测试进行，如果没有特殊说明均在 Starry 中终端运行

- [ ] chat

- 没法起多个终端, 不方便, 一个终端服务器跑在后台, 一个服务器跑在前台，不过应该是可以运行的

```sh
$ ./chat 127.0.0.1:8080
$ telnet localhost 8080
```

- [ ] hello_world

- 目前由于网桥不支持，所以暂时无法通过

```sh
# 主机上运行
$ netcat -l -p 6142

# Starry 中终端
$ ./hello_world
```

- [x] tiny-http

```sh
$ ./tinyhttp &
$ busybox wget 127.0.0.1:8080/plaintext
Connecting to 127.0.0.1:8080 (127.0.0.1:8080)
saving to 'plaintext'
plaintext            100% |********************************|    13  0:00:00 ETA
'plaintext' saved
$ busybox cat ./plaintext
hello_world
$ busybox wget 127.0.0.1:8080/json
$ busybox cat ./json
```

- [x] ~~named-pipe-ready只支持windows~~

- [x] print_each_packet

```sh
$ ./print_each_packet &

$ ./connect 127.0.0.1:8080
```

- [ ] echo-udp和udp-client
- 需要使用EOF terminate

```
echo-udp 127.0.0.1:8080
./udp-client
```

- [x] ~~dump：不支持 linux 相关系统~~

- [x] echo

- [x] proxy

  ```sh
  $ ./proxy &
  $ ./echo &
  $ ./connect 127.0.0.1:8081
  ```

- [x] tinydb

  ```sh
  $ ./tinydb &
  $ ./connect 127.0.0.1:8080
  ```

- [x] udp-codec

支持运行 `./MediaServer -d &` 命令以及 `ffmpeg`。这个命令可以将 ZLMediaKit 作为守护进程在后台启动，并连接 `ffmpeg` 真正实现推流功能。

## ZLM 支持

ZLM 的分析仓库：https://github.com/Arceos-monolithic/ZLMediaKit-analysis/

如何启动 ZLM：https://github.com/Arceos-monolithic/ZLMediaKit-analysis/blob/main/ZLM%20%E8%BF%90%E8%A1%8C%E6%8C%87%E5%BC%95%EF%BC%88%E4%B8%89%E6%9C%88%EF%BC%89.md

### 使用的测试命令

以下指令用于分析 syscall 功能和最终测试。在并行实现这些功能的过程中不必每次都按这个流程走。

1. 在 [ZLMediaKit 提供的文档](https://docs.zlmediakit.com/zh/guide/install/start.html) 下载并编译项目，获取可执行文件 `MediaServer`
2. 将下面文件放到文件系统镜像中
   1. `MediaServer` 及同目录下其他文件、所需的动态库
   2. `ffmpeg` （需另外安装，可以本机 apt/pacman 安装后直接拖可执行文件）和所需的动态库
   3. 一个视频文件 `01.mp4`，内容随意。放到和 `ffmpeg` 同目录
3. 启动后执行 `./MediaServer -d &` 命令
4. 然后执行 `ffmpeg -re -i "01.mp4" -vcodec h264 -acodec aac -f rtsp -rtsp_transport tcp rtsp://127.0.0.1/live/test` 命令
5. 等待播放完成，检查 `MediaServer` 文件同目录下的 `www` 这个目录，里面套了几层，有视频传输留下的 `.ts` 格式流文件
6. 关机后，将 `.ts` 文件拷出文件系统镜像，（可以合并但不是必须）确认能正常播放，且与 `01.mp4` 内容相同，则说明 Starry 成功运行了 `ZLMediaKit`

### 待分配的任务

#### 从启动到等待连接

1. 在文件镜像中加入 `ffmpeg`，并支持简单启动。<font color='orange'>（已完成）</font>

   - MediaServer 启动后会执行 `execve("/bin/sh", ["sh", "-c", "which ffmpeg"], 0x7ffc153105d0`，也即询问内核 "ffmpeg 在哪"。**如果 Starry 上 busybox 的 which 指令有处理不对的地方，也应尽量完善**

     [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/ffmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0starry.pdf)， [代码](https://github.com/Arceos-monolithic/Starry/commit/8c92debedb383858f2d963289328c6e831330cd7)

2. 支持 `sys_rseq`：<font color='orange'>（暂放，未实现）</font>

   - rseq 是一个比较新的 syscall，扩展了 Linux 的锁机制。大体上来说，它可以让用户注册一段“临界区代码”，如果某个线程在其中执行时被调度了，下次再回到这个线程时就要从头开始。中文文档比较少，[这里](https://zhuanlan.zhihu.com/p/103960889)有一个，代码建议直接参考 [Linux 源码的这一段](https://github.com/torvalds/linux/blob/master/kernel/rseq.c)。
   - 这个功能对大家来说比较难，如果后续评估认为它太复杂可以绕过

3. 完善 `sys_dup2` 和 `sys_dup3`：<font color='orange'>（已完成）</font>

   - 目前 `ulib/axstarry/syscall_fs/src` 中有 `syscall_dup3`，它实际上实现的内容是 `dup2`。我们需要把它改名成 `dup2` （记得看 syscall 编号），然后添加一个真正的 `dup3`。这个不难，只是多一个参数

     [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/syscall_dup.pdf)， [代码](https://github.com/Arceos-monolithic/Starry/commit/0dfe1b3caa344514226132be1373904b9959dcab)

4. 完善 `futex`：<font color='orange'>（部分实现，FUTEX_CLOCK_REALTIME还未实现）</font>

   - 在第一阶段任务中，已经添加了一个 `FUTEX_WAKE_PRIVATE` 参数。但实际运行 ZLMediaKit 时需要更多的参数，例如 `FUTEX_WAIT_BITSET_PRIVATE` `FUTEX_CLOCK_REALTIME` `FUTEX_BITSET_MATCH_ANY` 等，因此需要在 Starry 中做出一个相对完善的 `futex` 实现。

     [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/%E6%94%AF%E6%8C%81futex.pdf)， [代码](https://github.com/Josen-B/linux_syscall_api/commit/87c36678d6ae26816ffbb5855f71bc7952fda4f4)

5. 文件镜像中添加其他必需文件<font color='orange'>（已完成）</font>

   - 第一阶段中已经添加了一堆动态库，例如 `/lib/x86_64-linux-gnu/libssl.so.3`，但实际运行时我们还需要更多，比如：
     - `/usr/lib/ssl/openssl.cnf`
     - ~~`/usr/lib/ssl/cert.pem`~~ 这个不一定需要。如果找不到这个文件，ZLMediaServer 会去找同目录下的 `default.pem`，也就是在本地编译后的 `ZLMediaKit/release/linux/Debug/default.pem` 这个位置。

6. `prctl` 需要支持设置线程名：<font color='orange'>（已完成）</font>

   - 如 `prctl(PR_SET_NAME, "stamp thread"`，相当于每个线程存一个字符串

     [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/prctl%E8%AE%BE%E7%BD%AE%E8%BF%9B%E7%A8%8B%E5%90%8D%E7%A7%B0.pdf)， [代码](https://github.com/Josen-B/linux_syscall_api/commit/05b919795abdd48d674a28b842bec2e08c6dc9fa)

7. `ioctl` 支持修改 `FIONBIO`：<font color='orange'>（已完成）</font>

   - 如 `ioctl(4, FIONBIO, [1])` 表示修改 4 这个 fd 为非阻塞读写，`ioctl(5, FIONBIO, [0])` 表示修改 5 这个 fd 为阻塞读写。它等价于 open 时的 `O_NONBLOCK` 或者 fcntl 的 `F_SETFL`，可以参考这俩去实现。“设置为非阻塞读写”意味着当用户用 read 去读这个文件（实际上是个设备 `/sys/devices/system/cpu/online`）却没读到的时候，应该立即返回 EAGAIN，而不是卡在这里去切换其他进程。

     [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/ioctl%E6%94%AF%E6%8C%81FIONBIO.pdf)， [代码](https://github.com/Starry-OS/axfs/pull/4/commits/bc9d61dda6eb8a94bd85b8715a3036a980757bf7)

8. `SCHED_FIFO` 调度以及相关 syscall：

   <font color='orange'>（新版本starry中已实现sched_setscheduler；  sched_get_priority_min 和 sched_get_priority_max已写完代码，还未提交）</font>

   - 什么是 `SCHED_FIFO`？
     给没上过 OS 课的同学说一下，在这种调度框架下，每个进程有一个优先级，而每个优先级都对应一个队列。在同队列内，每个进程轮流运行；在不同队列之间，优先级高的必定抢占优先级低的，除非高优先级的进程退出或者主动让出。
     以 Linux 为例，优先级最高是 99 最低是 1。假设四个进程 $a,b,c,d$ 的优先级分别是 $99,99,5,1$。那么 a 和 b 轮流运行。除非有一天 a 和 b 都调用 `sys_sleep`，那此时轮到 c 运行，一个时钟周期后内核又回到 a 和 b。假设此时 a,b 都没醒，那么还是只有 c 可以运行，a,b,d 不能运行。假设此时 a 醒了 b 没醒，那么此时只有 a 可以运行，b,c,d 不能运行。

   - MediaServer 进程执行了什么操作？

     ```c
     sched_get_priority_min(SCHED_FIFO) = 1
     sched_get_priority_max(SCHED_FIFO) = 99
     sched_setscheduler(7560, SCHED_FIFO, [99]) = 0
     ```

     它首先通过 `sched_get_priority_min` 和 `sched_get_priority_max` 获取了内核允许的最大和最小优先级（一般就设置为和 Linux 一致，从 1 到 99），然后通过 `sched_setscheduler` 设置了自己的优先级。我们需要实现这三个 syscall。

9. 线程绑定到核： `sched_setaffinity` 和 `sched_getaffinity`：<font color='orange'>（新版本starry中已实现）</font>

   - 这个 syscall 可以指定将某个线程绑定到某些具体 cpu 执行。如 `sched_setaffinity(7560, 128, [0])` 表示将 7560 线程绑定到 0 号 cpu。注意第二个和第三个参数本质上是 bitset，有空可以根据[其他资料](https://blog.popkx.com/linux-multi-cpu-programming-specifying-cpu-sched_setaffinity-and-sched_getaffinity-for-threads/)写个基于 musl-libc 的小测例实验一下。
   - 注意事项：
     - fork 出的进程也遵循父进程的设定。
     - ZLMediaKit 的行为是创建与核数相等的线程，然后每个线程各自绑定一个核。所以尽管 syscall 运行每个线程绑定到一个“集合”，但你**可以实现为线程要么不绑定，要么绑定到某一个唯一确定的核**。
     - `sched_getaffinity` 也要实现。在我这的例子是 `sched_getaffinity(0, 4096, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]) = 32`，不过最终测试的实机上核心数可能要少一些。

10. `socket` 的 IPV6 模式：<font color='orange'>（已完成fake调用流程）</font>

    - ZLMediaKit 默认使用 IPv6：`socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 64`。在内核里参照目前的网络栈加个 ipv6 支持难度应该不大？实在不行再去改 ZLMediaKit 配置看能否改 ipv4。

    - 还有与之相关的一系列 syscall。具体流程是：

      ```c
      socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 64
      setsockopt(64, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
      ioctl(64, FIONBIO, [1])           = 0
      fcntl(64, F_GETFD) = 0
      fcntl(64, F_SETFD, FD_CLOEXEC)    = 0
      setsockopt(64, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
      bind(64, {sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::", &sin6_addr), sin6_scope_id=0}, 28) = 0
      listen(64, 1024)                  = 0
      getsockname(64,{sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
      getpeername(64, 0x55fa52f82770, [128]) = -1 ENOTCONN (Transport endpoint is not connected)
      ```

      之后就是把它扔进 epoll 里不断等待连接了。这里涉及 ipv6 的有 `setsockopt` `bind` 和 `getsockname`。可以找找 libc-test 有没有相关测例可以用。

      [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/%E6%94%AF%E6%8C%81socket%E7%9A%84ipv6%E6%A8%A1%E5%BC%8F.pdf)， [代码1](https://github.com/Starry-OS/linux_syscall_api/commit/ae662d33814613bb39315fe3e7a98f12e7e73a1d)，[代码2](https://github.com/Starry-OS/axnet/commit/133141f47e25202796cdfaf0ac78b92bfc5f5358)

11. OOM 模式：<font color='orange'>（已完成）</font>
    应用会试图打开 `/proc/sys/vm/overcommit_memory` 文件并读取其中的信息。这个文件包含了“内核如何处理内存溢出情况”的信息。它比较复杂且不必要，我们不用实现，读取时直接默认返回一个 `"0"` 即可（注意不是返回空，是一个 ASCII 字符 `0`）

    ```c
    openat(AT_FDCWD, "/proc/sys/vm/overcommit_memory", O_RDONLY|O_CLOEXEC)             = 11
    read(110, "0", 1)                 = 1
    ```

​	  [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/OMM%E6%A8%A1%E5%BC%8F.pdf)， [代码](https://github.com/Starry-OS/axstarry/commit/fd2a166ff2c71c5304f65615d3b2854e6715a039)

#### FFMPEG 支持

> ffmpeg 所需要的库太多导致占用内存很大。是否有可能把 ffmpeg 启动在外部，然后去连 Starry 里面的 ZLMediaKit？如果可以实现，我们就不需要在 Starry 里支持 ffmpeg 了。

以下是启动 ffmpeg 所需的支持。

上述流程只是启动了 Server 端，并没有真正让它“发挥作用”。我们需要从 ffmpeg 建立连接向 Server 推流。

在这一阶段主要涉及以下功能

12. ffmpeg 的动态库支持。<font color='orange'>（已完成）</font>

    - ffmpeg 需要的动态库实在太多，去掉没打开成功的 19 个还有 204 个。我把列表放在本项目的 `ffmpeg_dylib.txt` 中了。

13. prlimit 需要支持设置栈大小 <font color='orange'>（已完成）</font>

    - 目前的 `syscall_prlimit64` 仅支持修改最大文件数，用户栈大小固定是 256K，但 ffmpeg 的调用是 `prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0` 直接要了 8M 的栈大小。这可能需要动一下 TCB 的结构，把常量改成变量。可以参考 `curr_process.fd_manager.set_limit(new_limit)` 设置文件数，写一个类似 `curr_process.set_stack_limit` 的东西。

      [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/prlimit%E6%94%AF%E6%8C%81%E8%AE%BE%E7%BD%AE%E6%A0%88%E5%A4%A7%E5%B0%8F.pdf)， [代码1](https://github.com/Starry-OS/linux_syscall_api/commit/8c394a0cb590a73ea42c352ffe43d935e2d00397)，[代码2](https://github.com/Starry-OS/axprocess/commit/336c510afe95f736fac582f445ac4d76a9df875b)

14. 简单应付一下 prctl 输出 <font color='orange'>（排查ffmpeg日志中没有以下输出）</font>

    - 目前 prctl 是直接返回 0。需要改成对以下参数返回 -1 或者 1：

      ```c
      prctl(PR_CAPBSET_READ, CAP_MAC_OVERRIDE) = 1
      prctl(PR_CAPBSET_READ, 0x30 /* CAP_??? */) = -1 EINVAL (Invalid argument)
      prctl(PR_CAPBSET_READ, CAP_CHECKPOINT_RESTORE) = 1
      prctl(PR_CAPBSET_READ, 0x2c /* CAP_??? */) = -1 EINVAL (Invalid argument)
      prctl(PR_CAPBSET_READ, 0x2a /* CAP_??? */) = -1 EINVAL (Invalid argument)
      prctl(PR_CAPBSET_READ, 0x29 /* CAP_??? */) = -1 EINVAL (Invalid argument)
      ```

      不建议去实现对应功能，直接用 if-else 特判吧。

15. 新增特殊文件：<font color='orange'>（已经合并）</font>

    - ffmpeg 会使用新的特殊文件 `/proc/filesystems` `/proc/self/status` 以及目录 `/sys/devices/system/cpu` 对应如下输出

      ```c
      openat(AT_FDCWD, "/proc/filesystems", O_RDONLY|O_CLOEXEC) = 3
      newfstatat(3, "", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_EMPTY_PATH) = 0
      read(3, "nodev\tsysfs\nnodev\ttmpfs\nnodev\tbd"..., 1024) = 471
      openat(AT_FDCWD, "/proc/self/status", O_RDONLY) = 3
      close(3)                                = 0  
      newfstatat(3, "", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_EMPTY_PATH) = 0
      read(3, "Name:\tffmpeg\nUmask:\t0022\nState:\t"..., 1024) = 1024
      read(3, "s_allowed:\t1\nMems_allowed_list:\t"..., 1024) = 94
      close(3)                                = 0
      sched_getaffinity(0, 512, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]) = 32
      openat(AT_FDCWD, "/sys/devices/system/cpu", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
      newfstatat(3, "", {st_mode=S_IFDIR|0755, st_size=0, ...}, AT_EMPTY_PATH) = 0
      getdents64(3, 0x5636c4118240 /* 36 entries */, 32768) = 1064
      getdents64(3, 0x5636c4118240 /* 0 entries */, 32768) = 0
      close(3)                                = 0
      ```

      你可以在本机上检查自己的这几个文件和目录的格式，然后在 Starry 中实现时查询 axmem / axprocess 等等模块，填入同样的信息。不需要完全支持，

    - `/proc/filesystems` 就写 fat32，或者考虑到以后我们有 ext4 也可以再加一个

    - `/proc/self/status` 支持 name / pid / fdsize 应该足够

    - `/sys/devices/system/cpu` 只会被访问目录项和之前就支持的 `/sys/devices/system/cpu/online` 文件。ffmpeg 只是“看看”这个目录，不会实际打开其他文件读写。所以只需要在里面放 `cpu0`, `cpu1`... 这些空目录，再加上上述的 `online` 文件就行。

      [文档](https://github.com/Josen-B/Docs_for_zlm/blob/main/ZLMediaKit/%E6%B7%BB%E5%8A%A0%E7%89%B9%E6%AE%8A%E6%96%87%E4%BB%B6.pdf)， 代码暂存本地

16. 其他略过的 syscall <font color='orange'>（待启动）</font>

    - `mlock` 返回 0，`getuid` `geteuid` 返回 1000 即可。后两个在目前的 Starry 里是返回 0 的，需要过一遍其他测例检查改成 1000 有没有其他问题。
    - `ioctl(2, TCGETS, {B38400 opost isig icanon echo ...}) = 0` 这里 ioctl 通过 TCGETS 获取串口信息，随后在屏幕上输出。这里还没有进一步考察是否可以直接忽略并返回 0。

17. 新增 mremap <font color='orange'>（新版本starry已实现）</font>

    - 和名字一样，sys_mremap 的意思是重新映射一段内存，对应这一条调用 `mremap(0x7fad7240c000, 163840, 344064, MREMAP_MAYMOVE) = 0x7fad723b8000`。
    - 意思是以 `0x7fad7240c000` 开头的内存区间本来是 `163840` 这么长，现在想要向后拓展为 `344064` 这么长。如果后面没有足够的空间，本来这个调用是要失败的，但它用 `MREMAP_MAYMOVE` 指定了区间可以移动。也就是说，内核可以找一段 `344064` 长的新区间，把已有区间搬过去放在前半截，然后删除掉旧的区间。
    - 包含“扩展区间”和“复制并移动现有区间”两个功能，应该不算太难

之后就是建立连接的工作了。顺便一提，ffmpeg 使用 poll 而非 epoll 接收：

```c
sendto(4, "OPTIONS rtsp://127.0.0.1:554/liv"..., 87, MSG_NOSIGNAL, NULL, 0) = 87
poll([{fd=4, events=POLLIN}], 1, 100)   = 1 ([{fd=4, revents=POLLIN}])
recvfrom(4, "R", 1, 0, NULL, NULL)      = 1
poll([{fd=4, events=POLLIN}], 1, 100)   = 1 ([{fd=4, revents=POLLIN}])
recvfrom(4, "T", 1, 0, NULL, NULL)      = 1
poll([{fd=4, events=POLLIN}], 1, 100)   = 1 ([{fd=4, revents=POLLIN}])
recvfrom(4, "S", 1, 0, NULL, NULL)      = 1
poll([{fd=4, events=POLLIN}], 1, 100)   = 1 ([{fd=4, revents=POLLIN}])
recvfrom(4, "P", 1, 0, NULL, NULL)      = 1
......
```

不过这个应该 Starry 本身有支持，不需要额外新增功能。

#### 建立连接<font color='orange'>（待启动）</font>

以下是 `ffmpeg` 和 `ZLMediaKit` 两边建立连接发生的 syscall 调用以及需要的支持。想要在机器上看到这一部分的功能，至少需要先完成 `ffmpeg` 的启动和推流。

接收并建立连接，主要涉及 `accept` 和 `recvfrom` `sendmsg` 中的 ipv6 问题：

- 首先是建立连接，设置 fd 相关的参数

  ```c
  accept(64, {sa_family=AF_INET6, sin6_port=htons(57660), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 109
  ioctl(109, FIONBIO, [1])          = 0
  setsockopt(109, SOL_TCP, TCP_NODELAY, [1], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_SNDBUF, [262144], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_RCVBUF, [262144], 4) = 0
  setsockopt(109, SOL_SOCKET, SO_LINGER, {l_onoff=0, l_linger=0}, 8) = 0
  fcntl(109, F_GETFD)               = 0
  fcntl(109, F_SETFD, FD_CLOEXEC)   = 0
  getsockname(109, {sa_family=AF_INET6, sin6_port=htons(554), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
  getpeername(109, {sa_family=AF_INET6, sin6_port=htons(57660), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
  ```

- 然后扔进 epoll

  ```c
  epoll_ctl(6, EPOLL_CTL_ADD, 109, {events=EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP|EPOLLEXCLUSIVE|EPOLLET, data={u32=109, u64=109}}) = 0
  epoll_wait(6, [{events=EPOLLIN|EPOLLOUT, data={u32=109, u64=109}}], 1024, 5) = 1
  ```

- 这时我们手动在另一边启动 ffmpeg，开始尝试连接 554 端口：

  ```c
  socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_TCP) = 4
   fcntl(4, F_GETFL)                       = 0x2 (flags O_RDWR)
   fcntl(4, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
   connect(4, {sa_family=AF_INET, sin_port=htons(554), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
   poll([{fd=4, events=POLLOUT}], 1, 100)  = 1 ([{fd=4, revents=POLLOUT}])
   getsockopt(4, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
   getpeername(4, {sa_family=AF_INET, sin_port=htons(554), sin_addr=inet_addr("127.0.0.1")}, [128 => 16]) = 0
   poll([{fd=4, events=POLLOUT}], 1, 100)  = 1 ([{fd=4, revents=POLLOUT}])
   sendto(4, "OPTIONS rtsp://127.0.0.1:554/liv"..., 87, MSG_NOSIGNAL, NULL, 0) = 87
  ```

- `ZLMediaKit` 响应后，开始接收推流

  ```c
  recvfrom(109, "OPTIONS rtsp://127.0.0.1:554/liv"..., 262144, 0, 0x7fd1677faa30, [128 => 0]) = 87
  sendmsg(109, {msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="RTSP/1.0 200 OK\r\nCSeq: 1\r\nDate: "..., iov_len=279}], msg_iovlen=1, msg_controllen=0, msg_flags=MSG_DONTWAIT|MSG_NOSIGNAL}, MSG_DONTWAIT|MSG_NOSIGNAL)            = 279
  recvfrom(109, "ANNOUNCE rtsp://127.0.0.1:554/li"..., 262144, 0, 0x7fd1677faa30, [128 => 0]) = 626
  ```

   一部分 recvfrom 和 accept 的输出是 `EAGAIN`，表示资源不可用。这是正常的，单纯只是因为其他没建立连接的线程在抢同个文件。顺便一提，前几次 recvfrom 正常收到的信息内容分别是 `OPTIONS` `ANNOUNCE` `SETUP` `RECORD`，随后才开始正常传输，不过这属于 rtsp 协议的内容了，内核不需要关心。

- 最后传输完成后 `shutdown`，这个目前 Starry 应该已经有了

#### 检查输出<font color='orange'>（待启动）</font>

我们需要一项机制去检查内核是否实现了 ZLMidiaKit 需要的功能，且使其正常工作。显然，`return 0` 不能算是一个程序正常运转的
依据。

ZLMediaKit 会将直播流存成 .ts 文件，保存在自己的一个文件中。例如 `openat(AT_FDCWD, "/home/ZLMediaKit/release/linux/Debug/www/live/test/2024-01-19/16/03-56_0.ts", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 110`。每个 ts 文件长度比较短，而且内容不全，需要修改 ZLMediaKit 让他保留完整文件。

> 在文件夹外还有一个 `openat(AT_FDCWD, "/home/ZLMediaKit/release/linux/Debug/www/live/test/hls.m3u8", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 110`，不过合并 ts 文件的时候不一定需要这个 m3u8

之后，我们还需要把文件从 Starry 上拉到本机上，正常播放，以证明 ZLMediaKit 输出了正确的文件。目前立即想到有两种思路：

- 比较优雅的方式：在 Starry 内部支持某种类似 scp 的指令，在运行时把文件拖出来
- 更简单直接的方式：在播放完成后关机，此时文件会保存在文件系统镜像上。写一个脚本把它挂载到本机。就可以把文件复制出来了

之后可以用 ffmpeg 合并 ts 文件并演示。或者其实直接播放 .ts 也是可以的。





## 提交规范

- 通过本地 fork 模块或者在模块上开分支的形式进行开发，project 中给大家分配的任务都在 [Starry](https://github.com/Starry-OS/Starry/issues) 仓库上有对应的 issue。
- 如果有问题，可以在 issue 评论区讨论。
- 提交 commit 时，在 commit comment 前写下对应的 issue，格式为 Starry-OS/Starry#IssueId，如 Starry-OS/Starry#23。
- 提起 PR 时将所有涉及修改的模块提起 PR，PR 中引用该次修改解决的问题的 ISSUE 地址（即 project 中给大家分配的任务对应的 issue，都在 Starry 仓库下）。
- 当一个 PR 合并之后，在对应的 issue 中将本次修改涉及的所有模块的 PR 链接以一个 comment 的形式提交，并且 close issue。之后如果出现问题方便进行回滚追溯。
