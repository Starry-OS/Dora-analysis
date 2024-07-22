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
   

