# Chromium net 模块 Socket 编程学习笔记

> 目标：理解 Chromium `net` 模块的分层架构，吃透 socket 的封装、生命周期与异步 IO 模型，提炼出可复用到自己项目的现代 C++ 网络编程范式。
>
> 说明：本项目是 **Windows 平台**，因此 Windows IOCP（Proactor）实现是学习重点，POSIX Reactor 作为对照理解。核心文件：[net/socket/tcp_socket_io_completion_port_win.cc](net/socket/tcp_socket_io_completion_port_win.cc)（Windows IOCP）、[net/socket/socket_posix.cc](net/socket/socket_posix.cc)（POSIX Reactor）、[net/server/http_server.cc](net/server/http_server.cc)（上层用法范本）。三者构成"上层用法 + 两种平台底层实现"的完整闭环。

---

## 一、学习路线（分阶段 + 进度跟踪）

| 阶段 | 主题 | 关键文件 | 状态 |
|------|------|----------|------|
| 0 | 环境与代码组织 | `net/README.md`、目录结构 | ✅ 已完成 |
| 1 | 核心抽象：错误码 / IOBuffer / 回调 | [net/base/net_errors.h](net/base/net_errors.h)、[net/base/io_buffer.h](net/base/io_buffer.h)、[net/base/completion_once_callback.h](net/base/completion_once_callback.h) | ✅ 已完成 |
| 2 | 异步 IO 三段式模式（Do/On/Handle） | [net/server/http_server.cc](net/server/http_server.cc) | ✅ 已完成 |
| 3 | 生命周期管理：WeakPtr + 延迟销毁 | [net/server/http_server.cc](net/server/http_server.cc) | ✅ 已完成 |
| 4 | 实战案例：HttpServer / HttpConnection / WebSocket | [net/server/](net/server/) | ✅ 已完成 |
| 4.5 | Socket 层内部实现：类层级 + Reactor 模式 + errno 映射 | [net/socket/socket_posix.cc](net/socket/socket_posix.cc)、[net/socket/tcp_server_socket.cc](net/socket/tcp_server_socket.cc) | ✅ 已完成 |
| 4.6 | Windows IOCP 实现（Proactor 模式）**★ 项目重点** | [net/socket/tcp_socket_io_completion_port_win.cc](net/socket/tcp_socket_io_completion_port_win.cc)、[net/socket/tcp_socket_win.h](net/socket/tcp_socket_win.h) | ✅ 已完成 |
| 5 | 落地 demo（套用模式写一个 TCP echo server） | — | ⏳ 待后续需求 |
| 6 | 集成到自己的项目 | — | ⏳ 待领导安排 |

> 进度更新约定：每完成一阶段把状态从 ⏳ 改为 ✅，并在对应章节补充笔记。

---

## 二、架构与分层概览

Chromium 的 `net` 模块是典型的分层设计，自底向上：

```
┌─────────────────────────────────────────────┐
│  应用层：net/server (HttpServer, WebSocket)   │
├─────────────────────────────────────────────┤
│  协议层：net/http, net/spdy, net/quic, net/ssl│
├─────────────────────────────────────────────┤
│  Socket 层：net/socket (StreamSocket 等)      │
├─────────────────────────────────────────────┤
│  基础抽象：net/base (IOBuffer, errors, ...)   │
└─────────────────────────────────────────────┘
```

- **net/base**：与协议无关的基础设施。错误码、IP 地址、IOBuffer、回调、网络变化通知等都在这里。是学习"设计思想"的最佳入口。
- **net/socket**：对原生 socket（POSIX / Winsock）的跨平台封装，定义了 `Socket` / `StreamSocket` / `ServerSocket` 抽象接口，`TCPClientSocket` / `TCPServerSocket` 等 TCP 实现，以及底层的 `SocketPosix`（Reactor）/ `TcpSocketIoCompletionPortWin`（Proactor）。详见 [3.5 节](#三五socket-层内部实现类层级) 与 [3.6 节](#三六-windows-iocp-实现proactor-模式重点)。
- **net/http, net/spdy, net/quic, net/ssl**：协议层实现，建立在 socket 抽象之上。
- **net/server**：用 socket 抽象搭出的 HTTP/WebSocket 服务器，是"如何用 Chromium 风格写网络服务"的范本。

**分层的关键纪律**：上层只依赖下层的抽象接口（如 `HttpServer` 只持有 `std::unique_ptr<ServerSocket>` 和 `StreamSocket*`，不依赖具体 TCP/SSL 实现），从而实现跨平台与可测试性。

---

## 三、核心抽象详解

### 3.1 错误码：一切返回值都是 `int`

[net/base/net_errors.h](net/base/net_errors.h#L16-L19):

```cpp
// Error values are negative.
enum Error {
  // No error.
  OK = 0,
  ...
};
```

[net/base/net_error_list.h](net/base/net_error_list.h#L32):

```cpp
NET_ERROR(IO_PENDING, -1)   // 异步操作未完成
NET_ERROR(FAILED, -2)        // 通用失败
NET_ERROR(ABORTED, -3)       // 被取消
...
```

**设计要点**：
- `OK = 0`，错误一律为负，正数表示字节数。**一个 `int` 同时承载"字节数 / 错误码 / pending 状态"三种语义**，签名极简。
- `ERR_IO_PENDING (-1)` 不是错误，而是"异步进行中，请等回调"的信号。这是整个异步模型的基石。
- 错误码分段：0-99 系统、100-199 连接、200-299 证书、300-399 HTTP……便于归类处理。
- `MapSystemError()` 把 OS 的 errno / WSAGetLastError() 统一映射到 `net::Error`，屏蔽平台差异。

**可复用结论**：自己设计异步网络库时，用"负数错误 + 0 成功 + 正数字节数 + 一个特殊 pending 值"的返回值约定，比抛异常或返回 `expected<T, error>` 更适合高频 IO 路径。

### 3.2 IOBuffer：引用计数的缓冲区层级

[net/base/io_buffer.h](net/base/io_buffer.h) 的类注释（L28-L83）直接阐述了设计哲学，是必读内容。核心三条：

1. **引用计数是为了异步安全，不是为了共享**。IOBuffer 继承 `RefCountedThreadSafe`，但注释明确说"不要当共享缓冲区用，不要跨线程同时用"。引用计数的真正目的是：**异步操作取消后，底层可能还在用这块内存（比如不可取消的同步 `ReadFile` 在 worker 线程跑着），引用计数保证缓冲区不被提前释放**。

2. **所有权在调用异步 IO 时隐式转移**。把 IOBuffer 传给 `Read()`/`Write()` 后，在操作完成（不是取消！）前，**不能**碰这块内存。取消也不算完成——取消后应当释放自己的引用，永不再用。

3. **按用途分层级，而非一个大而全的类**：

| 类 | 用途 | 关键方法 |
|----|------|----------|
| `IOBuffer` | 基类，不拥有内存 | `data()` / `span()` |
| `IOBufferWithSize` | 拥有固定大小内存，**读**常用 | 构造传 size |
| `StringIOBuffer` | 包装 `std::string`（只读） | — |
| `VectorIOBuffer` | 包装 `vector<uint8_t>`，**写**常用 | — |
| `DrainableIOBuffer` | 渐进式写，推进消费指针 | `DidConsume()` / `BytesRemaining()` |
| `GrowableIOBuffer` | 可扩容 + offset，**读未知大小流** | `SetCapacity()` / `set_offset()` |
| `WrappedIOBuffer` | 不拥有内存的临时包装，慎用 | — |
| `PickledIOBuffer` | 包装 Pickle，避免拷贝 | — |

[io_buffer.h#L196-L223](net/base/io_buffer.h#L196-L223) 给了 `DrainableIOBuffer` 的典型用法（循环 Write 直到写完）；[L225-L279](net/base/io_buffer.h#L225-L279) 给了 `GrowableIOBuffer` 的典型用法（读直到 EOF，容量不够就翻倍）。

**可复用结论**：异步 IO 的缓冲区设计，要把"内存所有权"和"当前可读写视图"分开。用引用计数保活 + 视图对象（offset/span）表达"读到哪/写到哪"，比裸指针 + 长度安全得多。

### 3.3 CompletionOnceCallback：异步契约

[net/base/completion_once_callback.h](net/base/completion_once_callback.h#L21):

```cpp
using CompletionOnceCallback = base::OnceCallback<void(int)>;
```

就一行。所有异步 IO 的回调都是"接收一个 `int`"——要么是字节数，要么是错误码。配合 `base::BindOnce` + `WeakPtr` 绑定成员函数，形成完整的异步调用链。

> 注释里标了 `DEPRECATED`，Chromium 后续在推 `base::expected` + `base::ByteSize` 来区分"字节数"和"错误码"，但当前代码库仍是 `OnceCallback<void(int)>` 为主。

---

## 三点五、Socket 层内部实现：类层级

> 本节是"socket 内部如何把阻塞 syscall 变成异步"。POSIX 用 Reactor（epoll/kqueue + 非阻塞 fd），Windows 用 Proactor（IOCP + overlapped IO）。两者都满足上层 `Socket` 接口的异步契约，但实现哲学不同。

### 3.5.1 类层级（POSIX 与 Windows 对照）

```
StreamSocket / ServerSocket              net/socket/{stream,server}_socket.h   ← 跨平台公开抽象接口
        ↑ inherits
TCPClientSocket / TCPServerSocket        net/socket/tcp_{client,server}_socket ← TCP 专用，处理 TCP 选项
        ↑ owns (平台二选一)
┌─────────────────────────────────────┬─────────────────────────────────────────────────────────┐
│ POSIX 分支                          │ Windows 分支                                             │
├─────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ TCPSocketPosix                      │ TCPSocketWin (基类, 抽象 Read/Write)                    │
│   ↑ owns                            │   ↑ inherits                                            │
│ SocketPosix                         │ TcpSocketIoCompletionPortWin (★ IOCP, 新)              │
│   : FdWatcher                       │   : CoreImpl : IOHandler                                │
│   ↑                                 │ TCPSocketDefaultWin (旧, ObjectWatcher, 不再推荐)       │
│ POSIX: socket/accept/connect/       │ Winsock: WSARecv/WSASend/WSAConnect (带 OVERLAPPED)    │
│         read/send (非阻塞)          │                                                         │
└─────────────────────────────────────┴─────────────────────────────────────────────────────────┘
```

**每一层职责单一、可替换**：
- `Socket` / `StreamSocket` / `ServerSocket`：纯接口，定义 `Read/Write/Connect/Accept` 的异步契约（[socket.h](net/socket/socket.h)、[stream_socket.h](net/socket/stream_socket.h)、[server_socket.h](net/socket/server_socket.h)）。上层（如 `HttpServer`）只依赖这层，不感知平台。
- `TCPClientSocket` / `TCPServerSocket`：TCP 协议专用，加 TCP 选项（`SetIPv6Only`、`SetDefaultOptionsForServer`）、把底层 TCP socket 包成 `StreamSocket` 接口。
- **POSIX 分支**：`TCPSocketPosix`（平台胶水，`SockaddrStorage` ↔ `IPEndPoint`）owns `SocketPosix`（Reactor 核心，持有 fd、watcher、callback）。
- **Windows 分支**：`TCPSocketWin`（基类，处理 socket 选项、connect）有虚 `Read/Write`，子类 `TcpSocketIoCompletionPortWin`（IOCP，[3.6 节](#三六-windows-iocp-实现proactor-模式重点)）和 `TCPSocketDefaultWin`（旧实现）。

### 3.5.2 SocketPosix 就是 Reactor（POSIX 分支）

[socket_posix.h#L27-L28](net/socket/socket_posix.h#L27-L28)：

```cpp
class NET_EXPORT_PRIVATE SocketPosix
    : public base::MessagePumpForIO::FdWatcher {
```

继承 `FdWatcher` 是关键——`SocketPosix` 注册自己为 fd 事件的观察者，`MessagePumpForIO`（底层是 epoll/kqueue/poll，按平台）在 fd 就绪时回调 `OnFileCanReadWithoutBlocking` / `OnFileCanWriteWithoutBlocking`。**这就是 Reactor 模式**：单线程事件循环 + 非阻塞 fd + 就绪回调。

### 3.5.3 非阻塞 fd + errno 映射

[socket_posix.cc#L109-L132](net/socket/socket_posix.cc#L109-L132) `Open()`：

```cpp
socket_fd_ = CreatePlatformSocket(address_family, SOCK_STREAM, IPPROTO_TCP);
if (!base::SetNonBlocking(socket_fd_)) {   // ★ 关键：fd 设为非阻塞
  int rv = MapSystemError(errno);
  Close();
  return rv;
}
```

所有 fd 一创建就 `SetNonBlocking`。之后 `read`/`send`/`accept`/`connect` 永不阻塞——要么立即成功，要么返回 `EAGAIN`/`EWOULDBLOCK`/`EINPROGRESS`。这些 errno 经三个映射函数变成 `net::Error`：

[socket_posix.cc#L50-L95](net/socket/socket_posix.cc#L50-L95)：

| POSIX errno | 映射函数 | net::Error | 含义 |
|-------------|----------|------------|------|
| `EAGAIN` / `EWOULDBLOCK` | `MapSystemError` | `ERR_IO_PENDING (-1)` | read/send 未就绪，等 fd 事件 |
| `EINPROGRESS` | `MapConnectError` | `ERR_IO_PENDING` | connect 进行中，等 fd 可写 |
| `ECONNABORTED` | `MapAcceptError` | `ERR_IO_PENDING` | accept 时对端 abort，**直接重试**而非报错 |
| `EACCES` | `MapConnectError` | `ERR_NETWORK_ACCESS_DENIED` | — |
| `ETIMEDOUT` | `MapConnectError` | `ERR_CONNECTION_TIMED_OUT` | — |
| 其他 | `MapSystemError` | 对应负值 | — |

**核心思想**：把"暂时没准备好"统一翻译成 `ERR_IO_PENDING`，让上层用同一套三段式逻辑处理，无需知道是 read 还是 connect 在等。

### 3.5.4 Read 的完整异步流程（逐行追踪）

以 `Read` 为例，看一个字节是怎么异步读上来的：

```
上层 Read(buf)
  → SocketPosix::Read()                    [socket_posix.cc#L299]
      → ReadIfReady(buf, RetryRead回调)     [L304-L306]   // 包一层 RetryRead
          → DoRead(buf)                     [L325]        // 同步尝试
              → HANDLE_EINTR(read(fd,...))  [L519]        // ★ 真正的 syscall
              → 返回 >=0 : 字节数（同步成功）
              → 返回 <0  : MapSystemError(errno)         // EAGAIN→ERR_IO_PENDING
          → 若 ERR_IO_PENDING:
              WatchFileDescriptor(WATCH_READ, &read_socket_watcher_, this)  [L329-L331]
              存 read_if_ready_callback_                                  [L336]
              return ERR_IO_PENDING
          → 若非 PENDING: 直接返回字节数/错误（同步路径）
      → 若 ERR_IO_PENDING: 存 read_buf_/read_callback_                   [L308-L310]
      → return rv

  ... fd 可读，MessagePump 回调 ...
  → OnFileCanReadWithoutBlocking(fd)       [L441]
      → ReadCompleted()                    [L540]
          → StopWatchingFileDescriptor     [L543]   // 停止监听
          → Run(read_if_ready_callback_, OK)  [L545]  // 通知"可以读了"
              → RetryRead(OK)              [L523]     // 重新发起 Read
                  → ReadIfReady(...) → DoRead → read(fd,...)  // 这次大概率有数据
                  → 若又 PENDING: 继续等（循环）
                  → 否则: Run(read_callback_, rv)  [L537]  // 最终结果给上层
```

**关键细节**：
1. `ReadIfReady` 不持有 `buf`——它只通知"可读了"，由 `RetryRead` 再次 `DoRead` 才真正读。这避免了 `buf` 在等待期间被持有（注释 [socket.h#L42-L53](net/socket/socket.h#L42-L53)）。
2. `Read`（非 IfReady）才持有 `buf`，用 `RetryRead` 把 `ReadIfReady` 的"就绪通知"转成"真正读取"。
3. `HANDLE_EINTR` 包裹所有 syscall，自动重试 `EINTR`（被信号中断）。
4. watcher 是一次性的：`ReadCompleted` 立即 `StopWatchingFileDescriptor`，要继续读需重新 `ReadIfReady` 重新注册。**这是 level-triggered 的手动重新注册，不是 edge-triggered**。

### 3.5.5 Write / Accept / Connect 的对称结构

三者结构与 Read 同构，只是监听的 fd 事件不同：

| 操作 | 同步 syscall | pending 时监听 | 就绪回调 | 完成处理 |
|------|-------------|---------------|----------|----------|
| Read | `read()` | `WATCH_READ` | `OnFileCanReadWithoutBlocking` | `ReadCompleted` |
| Write | `send(.., MSG_NOSIGNAL)` | `WATCH_WRITE` | `OnFileCanWriteWithoutBlocking` | `WriteCompleted` |
| Accept | `accept()` | `WATCH_READ` | `OnFileCanReadWithoutBlocking` | `AcceptCompleted` |
| Connect | `connect()` | `WATCH_WRITE` | `OnFileCanWriteWithoutBlocking` | `ConnectCompleted` |

注意 `OnFileCanReadWithoutBlocking` 同时服务 Accept 和 Read——靠 `accept_callback_` 是否为空区分（[socket_posix.cc#L441-L450](net/socket/socket_posix.cc#L441-L450)）；`OnFileCanWriteWithoutBlocking` 同时服务 Connect 和 Write——靠 `waiting_connect_` 标志区分（[L452-L459](net/socket/socket_posix.cc#L452-L459)）。

**Connect 的特殊点**（[L496-L513](net/socket/socket_posix.cc#L496-L513)）：fd 可写不代表连接成功，必须 `getsockopt(SO_ERROR)` 取真实错误码，再 `MapConnectError`。

**Accept 的特殊点**（[L461-L475](net/socket/socket_posix.cc#L461-L475)）：`ECONNABORTED`（对端在 accept 前 abort）映射成 `ERR_IO_PENDING` 让 `AcceptCompleted` 自动重试，而非上报错误——见 [L50-L62](net/socket/socket_posix.cc#L50-L62) 注释引用《UNIX Network Programming》5.11 节。

### 3.5.6 全双工 + 单一清理点

- **全双工**：`read_socket_watcher_` 和 `write_socket_watcher_` 是独立的两套 watcher + buffer + callback（[socket_posix.h#L136-L150](net/socket/socket_posix.h#L136-L150)），可同时挂一个读和一个写，互不干扰。注释 [socket_posix.h#L69-L70](net/socket/socket_posix.h#L69-L70) 明确："Full duplex mode (reading and writing at the same time) is supported."
- **单一清理点**：`StopWatchingAndCleanUp`（[socket_posix.cc#L582-L621](net/socket/socket_posix.cc#L582-L621)）停掉所有 watcher、按需 close fd、清空所有 callback/buffer。`Close()` 和 `ReleaseConnectedSocket()` 都走这里，保证资源不泄漏。
- **ThreadChecker**：所有方法开头 `DCHECK(thread_checker_.CalledOnValidThread())`，强制单线程使用——异步靠消息循环，不靠多线程。

### 3.5.7 TCPServerSocket::Accept 如何衔接

[tcp_server_socket.cc#L86-L113](net/socket/tcp_server_socket.cc#L86-L113) 把 `TCPSocket::Accept`（最终是 `SocketPosix::Accept`）包成 `ServerSocket::Accept` 接口，自己也是三段式的微缩版：

```cpp
int TCPServerSocket::Accept(socket, callback, peer_address) {
  // 用 base::Unretained(this) 而非 WeakPtr——注释说明 socket_ 生命周期由 this 保证
  CompletionOnceCallback accept_callback = base::BindOnce(
      &TCPServerSocket::OnAcceptCompleted, base::Unretained(this), ...);
  int result = socket_->Accept(&accepted_socket_, &accepted_address_,
                               std::move(accept_callback));
  if (result != ERR_IO_PENDING) {
    result = ConvertAcceptedSocket(result, socket, peer_address);  // 同步完成
  } else {
    pending_accept_ = true;
  }
  return result;
}
```

注意这里用的是 `base::Unretained(this)` 而非 `WeakPtr`——[注释 L96-L97](net/socket/tcp_server_socket.cc#L96-L97) 解释：`socket_` 由 `this` 拥有，`socket_` 析构前回调必不触发，故 `this` 也必存活。这是比 `WeakPtr` 更轻量的"生命周期保证"推理，值得学。

---

## 三点六、Windows IOCP 实现（Proactor 模式）★ 项目重点

> 本项目是 Windows 平台，IOCP 是核心。本节对照 POSIX Reactor 讲 Windows Proactor 的差异。
>
> 核心文件：[tcp_socket_io_completion_port_win.h](net/socket/tcp_socket_io_completion_port_win.h) / [.cc](net/socket/tcp_socket_io_completion_port_win.cc)，基类 [tcp_socket_win.h](net/socket/tcp_socket_win.h)。

### 3.6.1 Reactor vs Proactor：两种异步模型

| 维度 | POSIX Reactor (`SocketPosix`) | Windows Proactor (`TcpSocketIoCompletionPortWin`) |
|------|-------------------------------|---------------------------------------------------|
| 模型 | Reactor：就绪通知 | Proactor：完成通知 |
| 底层机制 | epoll/kqueue + 非阻塞 fd | IOCP + overlapped IO |
| syscall | `read`/`send`（非阻塞，返回 `EAGAIN`） | `WSARecv`/`WSASend`（带 OVERLAPPED，返回 `WSA_IO_PENDING`） |
| 数据就绪时机 | 内核通知"可读了"，**应用再调一次 syscall 才真读** | 内核**直接把数据读进 buffer**，完成后通知"已读完" |
| buffer 归属 | 等待期间可不持有（`ReadIfReady`） | 发起时必须交给内核（OVERLAPPED 携带 buffer 指针） |
| 触发方式 | level-triggered，手动重新注册 watcher | 一次性，完成包入队即触发 |
| 同步完成优化 | 无（同步成功就直接返回字节数） | `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS`：同步完成时**跳过** IOCP，避免 PostTask 开销 |
| Connect | `EINPROGRESS` → 监听 `WATCH_WRITE` | 不能用 IOCP，用 `WSAEvent` + `ObjectWatcher` |

**最本质的差异**：Reactor 是"告诉我什么时候能做，我自己做"；Proactor 是"你帮我做，做完通知我"。Proactor 把数据搬运交给内核，应用线程更省。

### 3.6.2 CoreImpl：IOHandler + ObjectWatcher 双重身份

[tcp_socket_io_completion_port_win.cc#L117-L120](net/socket/tcp_socket_io_completion_port_win.cc#L117-L120)：

```cpp
class TcpSocketIoCompletionPortWin::CoreImpl
    : public TCPSocketWin::Core,
      public base::win::ObjectWatcher::Delegate,      // 给 connect 用
      public base::MessagePumpForIO::IOHandler {       // 给 read/write 用
```

`CoreImpl` 同时实现两个 delegate，因为 Windows 上 read/write 走 IOCP，而 connect 不能走 IOCP（只能用 `WSAEventSelect` + 事件对象），所以 connect 走 `ObjectWatcher` 监听 `WSACreateEvent`。

注册流程在 `EnsureOverlappedIOInitialized`（[tcp_socket_io_completion_port_win.cc#L350-L394](net/socket/tcp_socket_io_completion_port_win.cc#L350-L394)）：

```cpp
// 1. 把 socket 句柄注册到 IOCP，关联 CoreImpl 作为 IOHandler
registered_as_io_handler_ = base::CurrentIOThread::Get()->RegisterIOHandler(
    reinterpret_cast<HANDLE>(socket_), &core);
// 2. 激活"同步完成跳过 IOCP"优化（需 IFS 句柄）
::SetFileCompletionNotificationModes(
    reinterpret_cast<HANDLE>(socket_),
    FILE_SKIP_COMPLETION_PORT_ON_SUCCESS);
```

### 3.6.3 Write 的完整异步流程

[tcp_socket_io_completion_port_win.cc#L269-L338](net/socket/tcp_socket_io_completion_port_win.cc#L269-L338)：

```cpp
int TcpSocketIoCompletionPortWin::Write(buf, buf_len, callback, ...) {
  EnsureOverlappedIOInitialized();
  WSABUF write_buffer{.len = buf_len, .buf = buf->data()};
  auto context = std::make_unique<CoreImpl::IOContext>(&core);  // 携带 OVERLAPPED

  int rv = ::WSASend(socket_, &write_buffer, 1, &bytes_sent, 0,
                     context->GetOverlapped(), nullptr);
  if (rv == 0) {
    // ★ 同步完成（与 POSIX EAGAIN 不同！这里数据已发出）
    if (skip_completion_port_on_success_) {
      context.reset();                    // 跳过 IOCP，直接返回
    } else {
      context.release();                  // 让 OnIOCompleted 接管（即便已处理）
    }
    return DidCompleteWrite(bytes_sent, ERROR_SUCCESS, buf, buf_len);
  }
  // rv == SOCKET_ERROR
  int wsa_error = ::WSAGetLastError();
  if (wsa_error == WSA_IO_PENDING) {
    // ★ 异步进行中：buffer 交给内核，完成后 OnIOCompleted 回调
    context->buffer = buf;                // 持有 buffer 直到完成
    context->completion_callback = std::move(callback);
    context->completion_method = &DidCompleteWrite;
    context.release();                    // 所有权转给 OnIOCompleted
    return ERR_IO_PENDING;
  }
  // 真错误
  context.reset();
  return MapSystemError(wsa_error);
}
```

**关键对比 POSIX**：
- POSIX `send` 返回 `EAGAIN` → 数据**没发**，注册 watcher 等可写。
- Windows `WSASend` 返回 `WSA_IO_PENDING` → 数据**已交给内核**正在发，buffer 不能动，完成时回调。
- POSIX 同步成功返回字节数就完事；Windows 同步成功（`rv == 0`）**默认还会触发 IOCP 完成包**，所以要 `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS` 优化掉这个多余的 PostTask。

### 3.6.4 Read 的两种模式：Read vs ReadIfReady

Windows IOCP 下 buffer 一旦发起就被内核持有，这让 `ReadIfReady`（"只通知可读，不真读"）变得棘手。Chromium 的解法在 `HandleReadRequest`（[tcp_socket_io_completion_port_win.cc#L458-L587](net/socket/tcp_socket_io_completion_port_win.cc#L458-L587)）：

**`Read`（`allow_zero_byte_overlapped_read = false`）**：
```cpp
// 直接用调用者的 buffer 发起 overlapped read
WSARecv(socket_, &read_buffer, 1, &bytes_read, &flags,
        context->GetOverlapped(), nullptr);
// WSA_IO_PENDING → 持有调用者 buffer，完成时数据已读好
```

**`ReadIfReady`（`allow_zero_byte_overlapped_read = true`）**：
```cpp
// 1. 先试非 overlapped 的 WSARecv（带调用者 buffer）
WSARecv(socket_, &read_buffer, 1, ..., /*overlapped=*/nullptr, nullptr);
//    若返回 0：数据已就绪，立即返回字节数
//    若 WSAEWOULDBLOCK：没数据，进入步骤 2
// 2. 发起"零字节 overlapped read"——buffer 长度为 0，纯粹为了等"可读"通知
read_buffer = {};   // 零字节
WSARecv(socket_, &read_buffer, 1, ..., context->GetOverlapped(), nullptr);
//    WSA_IO_PENDING → 不持有调用者 buffer，完成时只通知"可读了"
//    调用者收到 OK 后再调 ReadIfReady 真正读取
```

这个"零字节 overlapped read"是 Windows IOCP 实现 `ReadIfReady` 的经典技巧——用一次空读换取"可读"通知，避免持有调用者 buffer。

### 3.6.5 IOContext：overlapped 操作的上下文载体

[tcp_socket_io_completion_port_win.cc#L122-L149](net/socket/tcp_socket_io_completion_port_win.cc#L122-L149)：

```cpp
struct IOContext : public base::MessagePumpForIO::IOContext {
  const scoped_refptr<CoreImpl> core_keep_alive;  // ★ 自保活，直到 IO 完成
  scoped_refptr<IOBuffer> buffer;                  // 操作的 buffer
  int buffer_length = 0;
  CompletionMethod completion_method = nullptr;    // 完成时调哪个 DidCompleteXxx
  CompletionOnceCallback completion_callback;      // 上层回调
};
```

**`core_keep_alive` 是关键**：IOCP 完成包可能在 socket 对象析构后才入队，`core_keep_alive` 让 `CoreImpl` 在 IO 完成前不被销毁。`OnIOCompleted`（[L640-L659](net/socket/tcp_socket_io_completion_port_win.cc#L640-L659)）检查 `socket_` 是否还为 null（`Detach` 时置 null），若已 detach 则只释放 context 不调 completion。

### 3.6.6 完成回调路径

```
内核完成 IO → IOCP 完成包入队
  → MessagePumpForIO 取出包，调 IOHandler::OnIOCompleted
      → CoreImpl::OnIOCompleted(context, bytes, error)   [L640]
          → std::invoke(completion_method, socket_, ...)  // 调 DidCompleteRead/Write
          → Run(completion_callback, rv)                   // 触发上层回调
          → unique_ptr<IOContext> 析构                     // 释放 buffer 引用 + core_keep_alive
```

`OnIOCompleted` 用 `std::unique_ptr<IOContext> derived_context(static_cast<IOContext*>(context))` 接管 context 所有权——发起时 `context.release()` 放手，完成时 `OnIOCompleted` 接手，RAII 自动清理。这是 Windows overlapped IO 的标准所有权流转模式。

### 3.6.7 Connect 的特殊处理（不走 IOCP）

[tcp_socket_io_completion_port_win.cc#L611-L638](net/socket/tcp_socket_io_completion_port_win.cc#L611-L638)：

```cpp
HANDLE CoreImpl::GetConnectEvent() {
  connect_event_.Set(::WSACreateEvent());
  ::WSAEventSelect(socket_->socket_, connect_event_.get(), FD_CONNECT);  // 绑定 connect 事件
  return connect_event_.get();
}
void CoreImpl::WatchForConnect() {
  connect_watcher_.StartWatchingOnce(connect_event_.get(), this);  // 一次性监听
}
void CoreImpl::OnObjectSignaled(HANDLE object) {
  // connect 完成，调 TCPSocketWin::DidCompleteConnect
  socket_->DidCompleteConnect();
}
```

Windows 的 `connect` 不能像 POSIX 那样用 `EINPROGRESS` + fd 可写事件，只能用 `WSAEventSelect` + 事件对象 + `ObjectWatcher`。这是 Windows 异步 socket 的一个历史包袱。

---

## 四、异步 IO 三段式模式（核心中的核心）

这是从 [http_server.cc](net/server/http_server.cc) 提炼出的最值得复用的模式。每个 IO 操作（accept / read / write）都拆成 **三个函数**，命名与职责固定：

| 函数 | 职责 |
|------|------|
| `DoXxxLoop()` | 发起 IO；若返回 `ERR_IO_PENDING` 则退出等回调，否则立即处理结果并循环 |
| `OnXxxCompleted(int rv)` | 异步回调入口；处理完后重新进入 `DoXxxLoop` |
| `HandleXxxResult(int rv)` | 纯结果处理（成功建连接 / 失败关闭 / 解析数据），与 IO 发起解耦 |

### 4.1 模式骨架

```cpp
void Server::DoXxxLoop() {
  int rv;
  do {
    rv = socket_->Xxx(buf, size,
                      base::BindOnce(&Server::OnXxxCompleted,
                                     weak_ptr_factory_.GetWeakPtr()));
    if (rv == ERR_IO_PENDING)   // 异步未完成，等回调
      return;
    rv = HandleXxxResult(rv);    // 同步完成，立即处理
  } while (rv == OK);
}

void Server::OnXxxCompleted(int rv) {     // 异步回调
  if (HandleXxxResult(rv) == OK)
    DoXxxLoop();                          // 继续下一轮
}

int Server::HandleXxxResult(int rv) {     // 结果处理
  if (rv < 0) { Close(); return rv; }     // 错误
  // ...处理成功结果
  return OK;                              // 返回 OK 触发循环继续
}
```

### 4.2 三个实例（均在 http_server.cc）

- **Accept**: [DoAcceptLoop / OnAcceptCompleted / HandleAcceptResult](net/server/http_server.cc#L180-L211)
- **Read**: [DoReadLoop / OnReadCompleted / HandleReadResult](net/server/http_server.cc#L213-L332)
- **Write**: [DoWriteLoop / OnWriteCompleted / HandleWriteResult](net/server/http_server.cc#L334-L371)

三者结构完全同构，只是 `HandleXxxResult` 内部逻辑不同：
- `HandleAcceptResult`：成功则新建 `HttpConnection`、注册、触发 `DoReadLoop`。
- `HandleReadResult`：成功则 `read_buf->DidRead(rv)`，循环解析 HTTP 头 / WebSocket 帧；`rv <= 0` 关闭连接。
- `HandleWriteResult`：成功则 `write_buf->DidConsume(rv)`；`rv < 0` 关闭连接。

### 4.3 为什么这样设计

1. **同步与异步统一**：socket 的 `Read/Write/Accept` 可能同步完成（返回正数）也可能异步（返回 `ERR_IO_PENDING`）。三段式用同一个 `do-while` 把两种情况合并——同步时立即进 `HandleResult` 并循环，异步时退出等回调，回调里再进 `HandleResult` 并循环。**调用者无需区分同步/异步路径**。

2. **发起与处理解耦**：`HandleXxxResult` 是纯函数式的结果处理，不碰 IO，便于测试和复用。

3. **自然背压**：`do-while (rv == OK)` 在同步连续就绪时会循环，但一旦异步 pending 就退出，把控制权还给消息循环——不会阻塞线程，也不会无限递归。

4. **错误集中处理**：所有错误路径都走 `HandleXxxResult`，统一的 `Close()` 出口，避免散落的清理逻辑。

---

## 五、生命周期管理

### 5.1 WeakPtr 防悬空

所有异步回调都用 `weak_ptr_factory_.GetWeakPtr()` 绑定，例如 [http_server.cc#L65-L67](net/server/http_server.cc#L65-L67):

```cpp
base::SingleThreadTaskRunner::GetCurrentDefault()->PostTask(
    FROM_HERE, base::BindOnce(&HttpServer::DoAcceptLoop,
                              weak_ptr_factory_.GetWeakPtr()));
```

`WeakPtrFactory` 成员放在对象**最后**（[http_server.h#L147](net/server/http_server.h#L147) `base::WeakPtrFactory<HttpServer> weak_ptr_factory_{this};`），保证析构时先失效所有 WeakPtr，已 Post 的回调不会在对象销毁后触发。这是 Chromium 异步代码的标配。

### 5.2 延迟销毁：避免回调栈中的悬空指针

[http_server.cc#L144-L162](net/server/http_server.cc#L144-L162) 的 `Close()` 是经典案例：

```cpp
void HttpServer::Close(int connection_id) {
  auto it = id_to_connection_.find(connection_id);
  if (it == id_to_connection_.end()) return;
  closed_connections_.emplace_back(std::move(it->second));  // 移入"待销毁"列表
  id_to_connection_.erase(it);
  delegate_->OnClose(connection_id);
  // 不立即销毁！PostTask 到下一轮再清空
  base::SingleThreadTaskRunner::GetCurrentDefault()->PostTask(
      FROM_HERE, base::BindOnce(&HttpServer::DestroyClosedConnections,
                                weak_ptr_factory_.GetWeakPtr()));
}
```

原因：`delegate_->OnClose()` 的调用栈里，上层可能还持有 `HttpConnection*`（比如 `HandleReadResult` 里 `delegate_->OnHttpRequest(...)` 触发了 Close）。立即销毁会导致栈上的裸指针悬空。延迟到下一轮消息循环销毁，保证当前调用栈里的所有回调安全返回。

配合 `HasClosedConnection()`（[L540-L542](net/server/http_server.cc#L540-L542)）在每次 delegate 回调后检查"连接是否已被 Close"，决定是否提前退出循环。

### 5.3 IOBuffer 所有权转移

见 [3.2 节](#32-iobuffer引用计数的缓冲区层级)。要点：传给异步 IO 的 IOBuffer，在操作完成前不能动；取消后要释放引用、永不再用。

---

## 六、实战案例剖析：HttpServer 完整流程

### 6.1 启动与 Accept 循环

[构造函数](net/server/http_server.cc#L59-L68)：保存 `server_socket_`（已 listen 未 accept），**PostTask 到下一轮**才开始 `DoAcceptLoop`——避免 delegate 还没准备好就收到回调。

`DoAcceptLoop` 循环 `Accept`，每来一个连接：
1. `HandleAcceptResult` 新建 `HttpConnection(++last_id_, std::move(accepted_socket_))`，存入 `id_to_connection_` map；
2. 调 `delegate_->OnConnect(id)`；
3. 紧接着对该连接启动 `DoReadLoop`。

### 6.2 Read 循环 + 缓冲管理

[DoReadLoop](net/server/http_server.cc#L213-L231) + [HttpConnection::ReadIOBuffer](net/server/http_connection.h#L30-L71):

```cpp
HttpConnection::ReadIOBuffer* read_buf = connection->read_buf();
if (read_buf->RemainingCapacity() == 0 && !read_buf->IncreaseCapacity()) {
  Close(connection->id());          // 超过 1MB 上限，关闭
  return;
}
rv = connection->socket()->Read(read_buf, read_buf->RemainingCapacity(), ...);
```

`ReadIOBuffer` 封装 `GrowableIOBuffer`，初始 1024 字节，容量不够就翻倍（`kCapacityIncreaseFactor = 2`），上限默认 1MB。三段式 API：
- `RemainingCapacity()`：当前可写入尾部空间
- `DidRead(n)`：通知"刚读了 n 字节"，扩大可读区
- `DidConsume(n)`：消费掉 n 字节，**把未消费数据 memmove 到缓冲区头部**
- `readable_bytes()`：返回当前可读的 span（零拷贝）

[HandleReadResult](net/server/http_server.cc#L242-L332) 在 `while (!read_buf->readable_bytes().empty())` 里循环解析：先试 HTTP 头（状态机解析器在 [L373-L527](net/server/http_server.cc#L373-L527)），如果是 WebSocket 升级请求则创建 `WebSocket` 并交给它解析。解析不完（`pos == 0`）就 break 等更多数据；解析出错就 Close。

### 6.3 Write 循环 + 队列缓冲

[DoWriteLoop](net/server/http_server.cc#L334-L349) + [QueuedWriteIOBuffer](net/server/http_connection.h#L76-L117):

写缓冲用 `base::queue<std::unique_ptr<std::string>>` 维护待写分块——**指针稳定性**很重要，因为 `IOBuffer::data()` 会把分块的裸指针交给底层 socket，分块不能在写完前移动或销毁。

`SendRaw` ([L95-L105](net/server/http_server.cc#L95-L105)) 的细节值得学：先 `Append` 进队列，**只有当前没在写时才启动 `DoWriteLoop`**（`writing_in_progress = !write_buf->IsEmpty()`）。这保证了多次 `SendRaw` 不会并发发起多个 Write，而是排队串行写。

### 6.4 连接关闭的对称性

读出错 / 写出错 / 解析出错 / 主动 Close，最终都汇到 `Close(connection_id)`，走 [5.2 节](#52-延迟销毁避免回调栈中的悬空指针)的延迟销毁流程。**单一关闭出口**是避免资源泄漏的关键。

---

## 七、可复用到自己项目的设计要点清单

把上面提炼成可操作的 checklist，落地自己的网络服务时逐条对照：

1. **统一返回值约定**：IO 函数返回 `int`，`0` 成功 / 正数字节 / 负数错误 / 一个特殊值（如 `-1`）表示 pending。
2. **异步三段式**：每个 IO 操作拆 `DoXxxLoop` / `OnXxxCompleted` / `HandleXxxResult`，`do-while` 统一同步与异步路径。
3. **缓冲区引用计数 + 视图分离**：内存用 `shared_ptr`/`scoped_refptr` 保活，可读写位置用 offset/span 表达；按用途分多个 buffer 类，而非一个全能类。
4. **回调绑 WeakPtr**：异步回调一律 `BindOnce` + `GetWeakPtr()`，`WeakPtrFactory` 放成员最后。
5. **延迟销毁危险对象**：回调栈中可能被引用的对象，Close 时移入"待销毁列表"，PostTask 到下一轮再删。
6. **单一关闭出口**：所有错误路径汇到一个 `Close()` / `StopWatchingAndCleanUp()`，避免散落的资源清理。
7. **写排队串行化**：用队列缓冲 + "仅当未在写时才启动写循环"的判断，避免并发 Write。
8. **分层依赖抽象**：上层只持有 socket 的抽象接口（`unique_ptr<ServerSocket>`），不依赖具体实现。五层类层级每层职责单一可替换。
9. **背压靠 pending**：`ERR_IO_PENDING` 时退出循环还控制权给消息循环，天然不阻塞、不递归爆栈。
10. **错误码分段**：按系统/连接/证书/HTTP 等分段编号，便于归类与映射 OS 错误。
11. **非阻塞 fd 是异步的根基**：fd 一创建就 `SetNonBlocking`，所有 syscall 永不阻塞，靠 `EAGAIN`/`EINPROGRESS` → `ERR_IO_PENDING` 触发事件注册。
12. **Reactor 模式**：单线程 + 事件循环（epoll/kqueue）+ 非阻塞 fd + 就绪回调。`SocketPosix` 继承 `FdWatcher` 注册 fd 事件，`MessagePumpForIO` 就绪时回调。不要用"每连接一线程"的阻塞模型。
13. **syscall 用 `HANDLE_EINTR` 包裹**：自动重试被信号中断的调用，避免 `EINTR` 误判为错误。
14. **全双工靠独立 watcher**：读和写各有独立的 watcher + buffer + callback，可同时挂一个读一个写。
15. **生命周期用 `Unretained` 推理代替 `WeakPtr`**：当回调的触发由被 owns 的对象决定时（如 `socket_` owns fd，fd 析构前回调必不触发），用 `base::Unretained(this)` 比 `WeakPtr` 更轻量。只有回调可能跨对象析构时才用 `WeakPtr`。
16. **Windows 用 Proactor（IOCP）而非 Reactor** ★：`WSARecv`/`WSASend` 带 OVERLAPPED 发起，`WSA_IO_PENDING` 对应 `ERR_IO_PENDING`，内核完成时回调 `OnIOCompleted`（数据已就绪，无需再 syscall）。比 Reactor 更省应用线程。
17. **`FILE_SKIP_COMPLETION_PORT_ON_SUCCESS` 优化** ★：Windows overlapped IO 同步成功时默认仍会发 IOCP 完成包，造成多余 PostTask。设此标志让同步成功直接返回，跳过 IOCP。需 IFS 句柄（`SkipCompletionPortOnSuccessIsSupported` 检查）。
18. **overlapped IO 的 buffer 必须在发起时交给内核** ★：与 POSIX 不同，Windows IOCP 下 buffer 在 `WSARecv`/`WSASend` 返回 `WSA_IO_PENDING` 后就被内核持有，完成前不能动。用 `IOContext` 携带 `scoped_refptr<IOBuffer>` 保活。
19. **`ReadIfReady` 用零字节 overlapped read 实现** ★：Windows IOCP 下无法"只通知不读"，Chromium 用发起 0 字节 `WSARecv` 的技巧换取"可读"通知，避免持有调用者 buffer。
20. **overlapped context 用 `release()`/`unique_ptr` 转移所有权** ★：发起时 `context.release()` 放手让 `OnIOCompleted` 接管，完成时 `unique_ptr<IOContext>` 接手，RAII 自动清理。这是 overlapped IO 的标准所有权流转。
21. **`core_keep_alive` 自保活** ★：IOCP 完成包可能在 socket 析构后入队，`IOContext` 持有 `scoped_refptr<CoreImpl>` 让其在 IO 完成前不被销毁。`OnIOCompleted` 检查 `socket_` 是否已 detach（置 null），是则只清理不调回调。
22. **Windows connect 不能用 IOCP** ★：用 `WSAEventSelect(FD_CONNECT)` + `WSACreateEvent` + `ObjectWatcher` 监听 connect 完成。`CoreImpl` 同时实现 `IOHandler`（read/write）和 `ObjectWatcher::Delegate`（connect）。

---

## 八、待补充

- [ ] 阶段 5：套用上述模式写一个自包含的 TCP echo server demo（待后续需求，Windows IOCP 版本优先）
- [ ] 阶段 6：集成到自己的项目（待领导安排项目方向）
- [ ] 后续可深入：`TCPClientSocket` 的连接重试 / Happy Eyeballs（IPv4/IPv6 竞速）、`SSLClientSocket` 如何在 `StreamSocket` 之上分层加 TLS、`socket_pool` 的连接复用与限流。
- [ ] Windows 专项：`TCPSocketDefaultWin`（旧 ObjectWatcher 实现）与 `TcpSocketIoCompletionPortWin`（新 IOCP）的迁移历史与性能对比；`udp_socket_win.cc` 的 IOCP 用法差异。
