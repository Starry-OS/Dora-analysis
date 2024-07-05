# Dora-analysis
## 背景

运行 Dora 到 Starry 上，最终支持智能化小车

- Starry 代码：https://github.com/Starry-OS/Starry/
- Dora 代码：https://github.com/dora-rs/dora
- Dora 以 musl-libc 编译的 x86_64 执行版本：https://github.com/Starry-OS/Starry/releases/tag/dora
- Dora on Starry 项目地址：https://github.com/orgs/Starry-OS/projects/1/views/4

## 运行流程说明

- Dora 分为三部分，分别是 coordinator，daemon，cli
- coordinator 启动之后，自动监听 53290 端口，等待 daemon 连入
- daemon 启动之后，向 53290 端口发送注册信息，告知自己所在的 IP 和端口，由 coordinator 确认注册成功
- coordinator 还会监听 6012 端口，等待接受 CLI 信息
- CLI 向 coordinator 发送一个 start 数据流，其中包括了需要启动的多个 node 的信息和他们所在的可执行文件路径
- coordinator 接收到数据流信息之后，搜索 node 运行在哪个 daemon 上，将需要启动的 node 信息和所在路径传递给对应的 daemon，称为 spawn dataflow
- daemon 接收到 spawn 数据之后，找到 node 所需要的可执行文件并且启动，若启动成功则向 coordinator 回传成功信息和 node 数据
- coordinator 接收到 All finished 信息之后向 CLI 传递数据，完成一次交互



## 运行 dora

- rust-dataflow 源码：https://github.com/dora-rs/dora/tree/main/examples/rust-dataflow

- 本地运行方式：在 dora 主仓库根目录下运行：

  ```sh
  # 运行 Dora
  $ cargo build -p dora-cli
  $ ./target/release/dora -h
  
  # 运行 Dora-example
  $ cargo build --example rust-dataflow
  $ cargo build -p rust-dataflow-example-node
  $ cargo build -p rust-dataflow-example-status-node
  $ cargo build -p rust-dataflow-example-sink
  $ cd examples/rust-dataflow
  $ ../../target/debug/dora start dataflow.yml
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
    $ make A=apps/monolithic_userboot LOG=off FEATURES=img ACCEL=n run
    $ ./dora -h
    ```

## 相关文件

- `Dora up` 指令在 linux 上运行得到的`starce` 结果：[up-log](./log/dora-up.txt)

- `Dora rust-dataflow example`跟踪指令

  - 生成方式：

    ```sh
    $  ./target/release/dora up
    # 查看 dora daemon 所在进程
    $ ps -a
    # 假设 daemon 所在进程为 pid
    $ strace -p {pid} -o daemon.txt -v -f -F
    
    # 新开一个进程窗口
    $ cd examples/rust-dataflow
    $ strace -v -f -F -o ./cli.txt ./dora start ./dataflow.yml --attach
    ```

  - 原理描述：dora start 会启动一个 CLI，读取 yml 内容并且向 coordinator 发送，coordinator 会转发给 daemon，由 daemon 去真正运行 node 对应的可执行文件（规定在 dataflow.yml）中。因此我们需要同时检测 daemon 和 CLI 的系统调用内容，结果分别存储在`daemon.txt` 和 `cli.txt` 下

  - 结果：

    - daemon 分析结果：[daemon](./log/daemon.txt)
    - cli 分析结果：[cli](./log/cli.txt)
    - coordinator 分析结果：[coordinator](./log/coordinator.txt)

## Syscall 分析支持需求

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

## 任务和细节划分

预期花费**两周时间**支持 Dora example rust-dataflow

Update time: 2024-7-5

- 郑友捷：
  - 分析 rust-dataflow example，做好**后续进一步分工**
  - 修改 Starry 逻辑，支持 ramdisk 启动 dora，加快 IO 速度
  - 支持 rust-dataflow example
- 李昕昊：上述工作的 2 部分，支持 Futex 相关 flags，可以使用 ltp 等测例进行测试，**需要在此基础上完善已有测例，真正测试功能正确性**（ltp 较多的测试参数准确程度，具体的功能正确与否还要确定）。预期时间：3 天
- 柏乔森：将 ZLM 改动通过 PR 等形式合入到 Starry 中，并且上述工作的第 7 部分，支持 `fcntl` 对应的 flags。预期时间：3 天。注意：合入过程中需要时刻运行 dora 或者已有测例判断功能正确性

- 杨金全：上述工作的 3、4 部分：即 epoll file 和 socket 对文件操作的支持，这个可以参考已有的文件类型如 `file` 进行，并且可以使用 ltp 或者自己编写的测例进行验证。预计时间：3 天
- 温祖彤： 完成宏内核 tutorial, 之后尝试支持 Clone 相关的flags，预期时间：5 天

对于 example 可能还有工作，会在分析结束后继续发布。
