# Chromium net 模块 Socket 编程学习笔记

> 目标：理解 Chromium `net` 模块的分层架构，吃透 socket 的封装、生命周期与异步 IO 模型，提炼出可复用到自己项目的现代 C++ 网络编程范式。
>
> 说明：本项目是 Windows 平台，Windows IOCP 是学习重点，POSIX Reactor 作为对照。核心文件：[tcp_socket_io_completion_port_win.cc](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc)（Windows IOCP）、[socket_posix.cc](file:///workspace/net/socket/socket_posix.cc)（POSIX）、[http_server.cc](file:///workspace/net/server/http_server.cc)（上层用法范本）。

---

## 一、学习路线（分阶段 + 进度跟踪）

| 阶段 | 主题 | 关键文件 | 状态 |
|------|------|----------|------|
| 0 | 环境与代码组织 | `net/README.md`、目录结构 | ✅ 已完成 |
| 1 | 核心抽象：错误码 / IOBuffer / 回调 | [net_errors.h](file:///workspace/net/base/net_errors.h)、[io_buffer.h](file:///workspace/net/base/io_buffer.h)、[completion_once_callback.h](file:///workspace/net/base/completion_once_callback.h) | ✅ 已完成 |
| 2 | 异步 IO 三段式模式（Do/On/Handle） | [http_server.cc](file:///workspace/net/server/http_server.cc) | ✅ 已完成 |
| 3 | 生命周期管理：WeakPtr + 延迟销毁 | [http_server.cc](file:///workspace/net/server/http_server.cc) | ✅ 已完成 |
| 4 | 实战案例：HttpServer / HttpConnection / WebSocket | [net/server/](file:///workspace/net/server/) | ✅ 已完成 |
| 4.5 | Socket 层内部实现：类层级 + Reactor 模式 + errno 映射 | [socket_posix.cc](file:///workspace/net/socket/socket_posix.cc)、[tcp_server_socket.cc](file:///workspace/net/socket/tcp_server_socket.cc) | ✅ 已完成 |
| 4.6 | Windows IOCP 实现（Proactor 模式）★ 项目重点 | [tcp_socket_io_completion_port_win.cc](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc)、[tcp_socket_win.h](file:///workspace/net/socket/tcp_socket_win.h) | ✅ 已完成 |
| 4.7 | 调用链路：上层如何调用使用 socket（池化/ConnectJob/Handle） | [client_socket_pool_manager.cc](file:///workspace/net/socket/client_socket_pool_manager.cc)、[http_stream_parser.cc](file:///workspace/net/http/http_stream_parser.cc) | ✅ 已完成 |
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

- **协议层**：`net/http`（HTTP/1.1 解析与流工厂）、`net/spdy`（SPDY/HTTP2 帧）、`net/quic`（QUIC 协议）、`net/ssl`（TLS 握手与加密），把字节流抽象成带语义的协议流。
- **net/base**：与协议无关的基础设施。错误码、IP 地址、IOBuffer、回调、网络变化通知等都在这里。是学习"设计思想"的最佳入口。
- **net/socket**：对原生 socket（POSIX / Winsock）的跨平台封装，定义了 `StreamSocket`、`ServerSocket`、`TCPServerSocket`、`SSLClientSocket` 等抽象。详见 3.5 节与 3.6 节。
- **net/server**：用 socket 抽象搭出的 HTTP/WebSocket 服务器，是"如何用 Chromium 风格写网络服务"的范本。

**分层的关键纪律**：上层只依赖下层的抽象接口（如 `HttpServer` 只持有 `std::unique_ptr<ServerSocket>` 和 `StreamSocket*`，不依赖具体 TCP/SSL 实现），从而实现跨平台与可测试性。

---

## 三、核心抽象详解

### 3.1 错误码：一切返回值都是 `int`

[net_errors.h](file:///workspace/net/base/net_errors.h#L16-L19):

```cpp
// Error values are negative.
enum Error {
  // No error.
  OK = 0,
  ...
};
```

[net_error_list.h](file:///workspace/net/base/net_error_list.h#L32):

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

[io_buffer.h](file:///workspace/net/base/io_buffer.h) 的类注释（L28-L83）直接阐述了设计哲学，是必读内容。核心三条：

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

[io_buffer.h#L196-L223](file:///workspace/net/base/io_buffer.h#L196-L223) 给了 `DrainableIOBuffer` 的典型用法（循环 Write 直到写完）；[L225-L279](file:///workspace/net/base/io_buffer.h#L225-L279) 给了 `GrowableIOBuffer` 的典型用法（读直到 EOF，容量不够就翻倍）。

**可复用结论**：异步 IO 的缓冲区设计，要把"内存所有权"和"当前可读写视图"分开。用引用计数保活 + 视图对象（offset/span）表达"读到哪/写到哪"，比裸指针 + 长度安全得多。

### 3.3 CompletionOnceCallback：异步契约

[completion_once_callback.h](file:///workspace/net/base/completion_once_callback.h#L21):

```cpp
using CompletionOnceCallback = base::OnceCallback<void(int)>;
```

就一行。所有异步 IO 的回调都是"接收一个 `int`"——要么是字节数，要么是错误码。配合 `base::BindOnce` + `WeakPtr` 绑定成员函数，形成完整的异步调用链。

> 注释里标了 `DEPRECATED`，Chromium 后续在推 `base::expected` + `base::ByteSize` 来区分"字节数"和"错误码"，但当前代码库仍是 `OnceCallback<void(int)>` 为主。

### 3.5 Socket 层内部实现（类层级）

`net/socket` 把"跨平台 socket 操作"拆成抽象接口层 + 平台实现层。本节聚焦 POSIX 分支的 Reactor 模式，3.6 节聚焦 Windows IOCP 的 Proactor 模式。

#### 3.5.1 类层级图（POSIX / Windows 对照）

自顶向下的继承与持有关系：

```
应用/协议层（持抽象指针）
  │  StreamSocket（抽象接口，net/socket/stream_socket.h）
  │  ServerSocket（抽象接口，net/socket/server_socket.h）
  ▼
TCP 封装层
  │  TCPClientSocket   ── 持有 std::unique_ptr<TCPSocket>
  │  TCPServerSocket   ── 持有 std::unique_ptr<TCPSocket>
  ▼
平台 TCP 层（TCPSocket 内部委托）
  │  POSIX:  TCPSocketPosix  ── 持有 SocketPosix
  │  Win:    TCPSocketWin    ── 持有 Core（RefCounted）
  │            ├── TCPSocketDefaultWin（旧实现，ObjectWatcher）
  │            └── TcpSocketIoCompletionPortWin（新实现，IOCP ★）
  ▼
原生 socket 封装层
  │  POSIX:  SocketPosix          ── 直接调 accept/connect/read/send
  │  Win:    CoreImpl（嵌套类）   ── 调 WSASend/WSARecv + IOCP
  ▼
OS syscall
     POSIX:  accept / connect / read / send
     Win:    WSASend / WSARecv / WSAEventSelect / SetFileCompletionNotificationModes
```

关键文件：
- 抽象接口：[stream_socket.h](file:///workspace/net/socket/stream_socket.h)、[server_socket.h](file:///workspace/net/socket/server_socket.h)
- POSIX 实现：[socket_posix.h](file:///workspace/net/socket/socket_posix.h) + [socket_posix.cc](file:///workspace/net/socket/socket_posix.cc)
- Windows 抽象基类：[tcp_socket_win.h](file:///workspace/net/socket/tcp_socket_win.h)（`Core` 抽象基类在 L176-L198）
- Windows IOCP 实现：[tcp_socket_io_completion_port_win.h](file:///workspace/net/socket/tcp_socket_io_completion_port_win.h) + [tcp_socket_io_completion_port_win.cc](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc)

#### 3.5.2 SocketPosix 是 Reactor（POSIX 分支）

`SocketPosix` 同时继承 `base::MessagePumpForIO::FdWatcher`，把自己注册为 fd 事件的观察者。fd 一创建就设为非阻塞：

[socket_posix.cc#L109-L132](file:///workspace/net/socket/socket_posix.cc#L109-L132) 的 `Open()`：

```cpp
socket_fd_ = CreatePlatformSocket(address_family, SOCK_STREAM, ...);
if (!base::SetNonBlocking(socket_fd_)) {   // fd 创建即非阻塞
  int rv = MapSystemError(errno);
  Close();
  return rv;
}
```

`AdoptUnconnectedSocket`（[L144-L157](file:///workspace/net/socket/socket_posix.cc#L144-L157)）同样在 adopt 后立即 `SetNonBlocking`。非阻塞是 Reactor 模式的前提——所有 syscall 都不会卡线程，"没数据"靠 errno 表达。

**errno 映射表**（把 OS 错误翻译成 `net::Error`）：

| OS errno | 映射结果 | 出处 | 含义 |
|----------|----------|------|------|
| `EAGAIN` / `EWOULDBLOCK` | `ERR_IO_PENDING` | `MapSystemError()`，在 [DoRead L519-L520](file:///workspace/net/socket/socket_posix.cc#L519-L520) / [DoWrite L562-L563](file:///workspace/net/socket/socket_posix.cc#L562-L563) 调用 | 数据未就绪，注册 watcher 等回调 |
| `EINPROGRESS` | `ERR_IO_PENDING` | [MapConnectError L82-L83](file:///workspace/net/socket/socket_posix.cc#L82-L83) | 非阻塞 connect 进行中，注册 WRITE watcher |
| `ECONNABORTED` | `ERR_IO_PENDING` | [MapAcceptError L57-L58](file:///workspace/net/socket/socket_posix.cc#L57-L58) | 客户端在 accept 前断开，自动重试 accept |
| `EACCES` | `ERR_NETWORK_ACCESS_DENIED` | [L84-L85](file:///workspace/net/socket/socket_posix.cc#L84-L85) | 权限拒绝 |
| `ETIMEDOUT` | `ERR_CONNECTION_TIMED_OUT` | [L86-L87](file:///workspace/net/socket/socket_posix.cc#L86-L87) | 连接超时 |

注意 `ECONNABORTED → ERR_IO_PENDING` 的妙处：accept 时客户端突然断开，不是报错而是返回 pending，让上层自动重试 accept，对调用者完全透明。

#### 3.5.3 Read 完整异步流程

`Read` 不是直接 `read()`，而是先 `ReadIfReady` 再包一层 `RetryRead`。完整链路（[socket_posix.cc](file:///workspace/net/socket/socket_posix.cc)）：

1. **Read**（[L299-L313](file:///workspace/net/socket/socket_posix.cc#L299-L313)）：调 `ReadIfReady`，回调绑 `RetryRead`（用 `Unretained`，见 3.5.7 推理）。若返回 `ERR_IO_PENDING`，缓存 `read_buf_` / `read_callback_`。
2. **ReadIfReady**（[L315-L338](file:///workspace/net/socket/socket_posix.cc#L315-L338)）：先 `DoRead` 试一次同步 read。
3. **DoRead**（[L515-L521](file:///workspace/net/socket/socket_posix.cc#L515-L521)）：`HANDLE_EINTR(read(...))`，返回值 `< 0` 则 `MapSystemError(errno)`——`EAGAIN` 被映射成 `ERR_IO_PENDING`。
4. 回到 `ReadIfReady`：`rv == ERR_IO_PENDING` 时调 `WatchFileDescriptor(WATCH_READ)`（[L329-L331](file:///workspace/net/socket/socket_posix.cc#L329-L331)）注册读 watcher，缓存 `read_if_ready_callback_`，返回 `ERR_IO_PENDING`。
5. 数据就绪 → MessagePump 回调 **OnFileCanReadWithoutBlocking**（[L441-L450](file:///workspace/net/socket/socket_posix.cc#L441-L450)）。
6. **ReadCompleted**（[L540-L546](file:///workspace/net/socket/socket_posix.cc#L540-L546)）：停 watcher，调 `read_if_ready_callback_.Run(OK)`——只通知"可读了"，不传数据。
7. **RetryRead**（[L523-L538](file:///workspace/net/socket/socket_posix.cc#L523-L538)）：再次 `ReadIfReady`，这次 `DoRead` 能读到数据（同步返回正数），最终 `read_callback_.Run(rv)` 把字节数交给上层。

> **关键设计**：`ReadIfReady` 只通知"可读"，不负责读数据；`RetryRead` 负责真正读。这种分离让"可读通知"可以被取消（`CancelReadIfReady`），而数据读取在回调时才发生。

#### 3.5.4 四操作对称结构

`Read` / `Write` / `Accept` / `Connect` 结构完全同构，都是"先试同步 → 失败注册 watcher → 回调重试"三段：

| 操作 | 同步尝试 | 注册 watcher | 回调入口 | 完成重试 |
|------|----------|-------------|----------|----------|
| Read | `DoRead` (L515) | `WATCH_READ` (L329) | `OnFileCanReadWithoutBlocking` (L441) | `RetryRead` (L523) |
| Write | `DoWrite` (L548) | `WATCH_WRITE` (L379) | `OnFileCanWriteWithoutBlocking` (L452) | `WriteCompleted` (L570) |
| Accept | `DoAccept` (L461) | `WATCH_READ` (L208) | `OnFileCanReadWithoutBlocking` (L441) | `AcceptCompleted` (L477) |
| Connect | `DoConnect` (L489) | `WATCH_WRITE` (L233) | `OnFileCanWriteWithoutBlocking` (L452) | `ConnectCompleted` (L496) |

**如何区分 Accept 和 Read**：两者都注册 `WATCH_READ`、都回调 `OnFileCanReadWithoutBlocking`。区分靠 `accept_callback_` 是否为空（[L444-L449](file:///workspace/net/socket/socket_posix.cc#L444-L449)）：

```cpp
void SocketPosix::OnFileCanReadWithoutBlocking(int fd) {
  if (!accept_callback_.is_null()) {
    AcceptCompleted();    // 正在 accept
  } else {
    DCHECK(!read_if_ready_callback_.is_null());
    ReadCompleted();      // 正在 read
  }
}
```

**如何区分 Connect 和 Write**：两者都注册 `WATCH_WRITE`、都回调 `OnFileCanWriteWithoutBlocking`。区分靠 `waiting_connect_` 标志（[L452-L459](file:///workspace/net/socket/socket_posix.cc#L452-L459)）。

#### 3.5.5 HANDLE_EINTR 处理被信号中断的 syscall

POSIX 下 `read` / `write` / `accept` / `connect` 可能被信号中断返回 `EINTR`。`HANDLE_EINTR` 宏自动重试：

```cpp
// DoAccept (L463-L464)
int new_socket = HANDLE_EINTR(accept(socket_fd_, ...));

// DoConnect (L490-L491)
int rv = HANDLE_EINTR(connect(socket_fd_, ...));

// DoRead (L519)
int rv = HANDLE_EINTR(read(socket_fd_, buf->data(), buf_len));

// DoWrite (L558-L559)
ssize_t send_rv = HANDLE_EINTR(send(socket_fd_, buf->data(), buf_len, MSG_NOSIGNAL));
```

`EINTR` 不映射成错误也不映射成 pending，而是静默重试，对上层完全透明。Windows 下没有 `EINTR` 概念，所以 IOCP 实现不需要这层处理。

#### 3.5.6 全双工独立 watcher

构造函数（[L99-L103](file:///workspace/net/socket/socket_posix.cc#L99-L103)）初始化三个独立 watcher：

```cpp
SocketPosix::SocketPosix()
    : socket_fd_(kInvalidSocket),
      accept_socket_watcher_(FROM_HERE),
      read_socket_watcher_(FROM_HERE),
      write_socket_watcher_(FROM_HERE) {}
```

- `read_socket_watcher_`：只管 read 就绪（`WATCH_READ`）
- `write_socket_watcher_`：只管 write/connect 就绪（`WATCH_WRITE`）
- `accept_socket_watcher_`：只管 accept 就绪（`WATCH_READ`，与 read 互斥）

**全双工**：read 和 write 用不同 watcher，可以同时 pending 互不干扰（[tcp_socket_win.h#L84-L85](file:///workspace/net/socket/tcp_socket_win.h#L84-L85) 的注释也明确："Full duplex mode (reading and writing at the same time) is supported"）。`StopWatchingAndCleanUp`（[L582-L621](file:///workspace/net/socket/socket_posix.cc#L582-L621)）关闭时三个 watcher 都要停。

#### 3.5.7 TCPServerSocket::Accept 用 Unretained 而非 WeakPtr 的生命周期推理

[tcp_server_socket.cc#L96-L100](file:///workspace/net/socket/tcp_server_socket.cc#L96-L100)：

```cpp
// It is safe to use base::Unretained(this). |socket_| is owned by this class,
// and the callback won't be run after |socket_| is destroyed.
CompletionOnceCallback accept_callback = base::BindOnce(
    &TCPServerSocket::OnAcceptCompleted, base::Unretained(this), socket,
    peer_address, std::move(callback));
```

同理 [socket_posix.cc#L302-L306](file:///workspace/net/socket/socket_posix.cc#L302-L306) 的 `RetryRead` 也用 `Unretained`：

```cpp
// Use base::Unretained() is safe here because OnFileCanReadWithoutBlocking()
// won't be called if |this| is gone.
int rv = ReadIfReady(buf, buf_len,
    base::BindOnce(&SocketPosix::RetryRead, base::Unretained(this)));
```

**为什么这里敢用 `Unretained` 而不用 `WeakPtr`**：

1. **回调的触发方是 `socket_` 自己**。`socket_`（`TCPSocket` / `SocketPosix`）是 `this` 的成员，由 `this` 独占持有。
2. **回调链经过 `socket_->Accept` / `socket_->Read`**，最终由 MessagePump 的 fd watcher 触发。fd watcher 注册在 `socket_` 内部的 `watcher_` 对象上。
3. **`this` 析构时必然先析构 `socket_`**（成员析构顺序），`socket_` 析构时 `StopWatchingAndCleanUp` 停掉所有 watcher，fd 不再被监听。
4. **watcher 已停 → MessagePump 不会再回调 → `this` 已析构也不悬空**。

简言之：**回调的"发源体"是 `this` 的成员，成员的销毁先于 `this`，销毁时已切断回调路径**。这比 `WeakPtr` 更轻量（无 `WeakPtrFactory` 开销），但前提是生命周期严格是"成员属于 this"的包含关系。`HttpServer` 用 `WeakPtr` 是因为它把回调 Post 到全局 TaskRunner，回调发源体不属于 `this`。

### 3.6 Windows IOCP 实现（Proactor 模式）★ 项目重点

Windows 平台不用 `select`/`poll`/`epoll`（Reactor），而用 IOCP（I/O Completion Port，Proactor）。本仓库的 [tcp_socket_io_completion_port_win.cc](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc) 是 Chromium 最新的 IOCP 实现，取代了旧的 `TCPSocketDefaultWin`。

#### 3.6.1 Reactor vs Proactor 对比表

| 维度 | POSIX Reactor | Windows IOCP Proactor |
|------|---------------|----------------------|
| **模型** | Reactor（反应器）：通知"可读/可写了，你自己来读/写" | Proactor（前摄器）：通知"读/写已完成，数据已就位" |
| **底层机制** | `epoll`/`kqueue`/`poll` 监听 fd 就绪事件 | IOCP 监听 overlapped I/O 完成包 |
| **syscall** | `read`/`send`（就绪后调用，非阻塞） | `WSARecv`/`WSASend`（发起时调用，带 overlapped） |
| **数据就绪时机** | 通知时数据**还没**读进 buffer，需要调用者再 `read` | 通知时数据**已经**在 buffer 里（内核拷贝完了） |
| **buffer 归属** | 通知阶段不持有 buffer，`read` 时才传 | 发起 IO 时就把 buffer 交给内核，完成前不能动 |
| **触发方式** | 边沿/水平触发，通知"fd 就绪" | 完成通知，通知"IO 完成" |
| **同步完成优化** | 无（`EAGAIN` 就是 pending） | `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS`：同步完成时不投完成包 |
| **Connect** | `connect` 返回 `EINPROGRESS`，注册 WRITE watcher | 不能用 IOCP，用 `WSAEventSelect` + 事件对象 + `ObjectWatcher` |

#### 3.6.2 CoreImpl 双重身份

[tcp_socket_io_completion_port_win.cc#L117-L120](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L117-L120) 定义 `CoreImpl`，同时继承两个 Delegate：

```cpp
class TcpSocketIoCompletionPortWin::CoreImpl
    : public TCPSocketWin::Core,
      public base::win::ObjectWatcher::Delegate,       // 用于 connect
      public base::MessagePumpForIO::IOHandler {        // 用于 read/write
```

- **`ObjectWatcher::Delegate`**：监听 connect 完成事件（`OnObjectSignaled`），因为 connect 不能走 IOCP。
- **`MessagePumpForIO::IOHandler`**：监听 read/write 完成包（`OnIOCompleted`），这是 IOCP 的核心。

`EnsureOverlappedIOInitialized`（[L350-L394](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L350-L394)）做两件事：

1. **注册 IOHandler**（[L358-L359](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L358-L359)）：把 socket 句柄和 `CoreImpl` 绑定到 IOCP。
2. **激活同步完成优化**（[L370-L373](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L370-L373)）：

```cpp
BOOL result = ::SetFileCompletionNotificationModes(
    reinterpret_cast<HANDLE>(socket_),
    FILE_SKIP_COMPLETION_PORT_ON_SUCCESS);
skip_completion_port_on_success_ = (result != 0);
```

`FILE_SKIP_COMPLETION_PORT_ON_SUCCESS` 的作用：当 `WSASend`/`WSARecv` 同步完成（返回 0）时，**不**往完成端口投完成包，省去一次 `PostTask` 开销。这是 IOCP 相比旧 `ObjectWatcher` 实现的关键性能优化（见 [tcp_socket_io_completion_port_win.h#L21-L24](file:///workspace/net/socket/tcp_socket_io_completion_port_win.h#L21-L24) 的类注释）。前提是 socket 必须返回 IFS 句柄（[L49-L84](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L49-L84) 的 `SkipCompletionPortOnSuccessIsSupported` 检查）。

#### 3.6.3 Write 完整异步流程

[Write](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L269-L338)（L269-L338）：

```cpp
const int rv = ::WSASend(socket_, &write_buffer, 1, &bytes_sent, 0,
                         context->GetOverlapped(), nullptr);
```

三种返回路径：

1. **`rv == 0`：同步完成**（[L296-L311](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L296-L311)）。与 POSIX 的 `EAGAIN` 不同——**数据已经发出去了**！如果开了 `skip_completion_port_on_success_`，直接 `context.reset()` 释放上下文（不需要等完成包），立即调 `DidCompleteWrite` 返回字节数。否则 `context.release()`（让 `OnIOCompleted` 接管）但立即处理。

2. **`WSA_IO_PENDING`：异步进行中**（[L315-L328](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L315-L328)）。**buffer 所有权交给内核**——`context->buffer = buf` 保活，`context->completion_method = &DidCompleteWrite`，`context.release()` 放弃 `unique_ptr` 所有权让 `OnIOCompleted` 接管。返回 `ERR_IO_PENDING`。

3. **其他错误**（[L330-L337](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L330-L337)）：`context.reset()` 清理，`MapSystemError` 返回错误码。

**关键对比 POSIX**：POSIX 的 `write` 返回 `EAGAIN` 时数据**没**发出，只是"buffer 可写了"；Windows 的 `WSASend` 返回 0 时数据**已经**发出。Proactor 在发起时就交出 buffer，Reactor 在通知时才传 buffer。

#### 3.6.4 Read 两种模式

[HandleReadRequest](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L458-L587)（L458-L587）被 `Read` 和 `ReadIfReady` 共用，靠 `allow_zero_byte_overlapped_read` 参数区分：

**模式一：Read（`allow_zero_byte_overlapped_read = false`）**——直接用调用者的 buffer 做 overlapped read（[L498-L501](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L498-L501)）：

```cpp
auto rv = ::WSARecv(socket_, &read_buffer, 1, &bytes_read, &flags,
                    context->GetOverlapped(), nullptr);
```

- `rv == 0`：同步完成，数据已在 buffer，立即 `DidCompleteRead`。
- `WSA_IO_PENDING`：异步进行中，buffer 交给内核，完成时 `OnIOCompleted` → `DidCompleteRead`。

**模式二：ReadIfReady（`allow_zero_byte_overlapped_read = true`）**——用零字节 overlapped read 技巧（[L515-L554](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L515-L554)）：

1. 先不带 overlapped 试 `WSARecv`（[L498-L501](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L498-L501) 传 `nullptr`）。
2. 如果返回 `WSAEWOULDBLOCK`（无数据），发起一个**零字节** overlapped `WSARecv`（[L522-L524](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L522-L524)）：buffer 长度为 0，纯为等"数据可读"通知。
3. 完成时 `OnIOCompleted` → `DidCompleteRead` 返回 `OK`（[L419-L420](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L419-L420)），上层再调一次 `ReadIfReady` 真正读数据。

这个技巧的目的是：**ReadIfReady 不持有调用者的 buffer**，所以可以被随时取消（`CancelReadIfReady` 只需清空 `completion_callback`，[L253-L267](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L253-L267)），而真正的数据读取在取消后仍可安全发生（因为 buffer 是零字节的）。

#### 3.6.5 IOContext 结构

[IOContext](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L122-L149)（L122-L149）是每个 overlapped 操作的上下文，继承自 `base::MessagePumpForIO::IOContext`：

```cpp
struct IOContext : public base::MessagePumpForIO::IOContext {
  const scoped_refptr<CoreImpl> core_keep_alive;   // 自保活
  scoped_refptr<IOBuffer> buffer;                  // 操作用的 buffer
  int buffer_length = 0;
  CompletionMethod completion_method = nullptr;    // 完成时调哪个方法
  CompletionOnceCallback completion_callback;      // 外部回调
};
```

**四个字段的职责**：

| 字段 | 职责 |
|------|------|
| `core_keep_alive` | 持有 `CoreImpl` 的引用计数，保证 socket 析构后 IO 完成时 `CoreImpl` 仍活着（[L132-L134](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L132-L134) 注释） |
| `buffer` | 发起 IO 时保活调用者的 IOBuffer，完成时 `std::move` 交出 |
| `completion_method` | 指向 `DidCompleteRead` 或 `DidCompleteWrite`，完成时调哪个 |
| `completion_callback` | 外部回调（上层传进来的），完成时 `Run(rv)` |

`core_keep_alive` 是关键：socket 可能在 IO 完成前析构，但 `CoreImpl` 继承 `RefCounted`（[tcp_socket_win.h#L176](file:///workspace/net/socket/tcp_socket_win.h#L176)），`IOContext` 持有引用就能保活。析构时 `CoreImpl::Detach`（[L602-L609](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L602-L609)）清空 `socket_` 指针，`OnIOCompleted` 检查 `socket_` 为空就不调 completion method。

#### 3.6.6 完成回调路径

[OnIOCompleted](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L640-L659)（L640-L659）：

```cpp
void CoreImpl::OnIOCompleted(IOContext* context, DWORD bytes_transferred, DWORD error) {
  // Take ownership of `context`, which was released in `Read` or `Write`.
  std::unique_ptr<IOContext> derived_context(static_cast<IOContext*>(context));

  if (socket_ && derived_context->completion_method) {
    const int rv = std::invoke(
        derived_context->completion_method, socket_, bytes_transferred, error,
        std::move(derived_context->buffer), derived_context->buffer_length);
    if (derived_context->completion_callback) {
      std::move(derived_context->completion_callback).Run(rv);
    }
  }
}
```

**RAII 自动清理**：发起 IO 时 `context.release()` 放弃所有权（[L325](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L325)、[L573](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L573)），完成时 `unique_ptr<IOContext>` 接管所有权（[L647](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L647)）。函数结束时 `derived_context` 析构，自动释放 `buffer`、`core_keep_alive` 等所有引用。**不需要手动 delete，不会泄漏**。

**所有权流转链**：`make_unique`（发起）→ `release()`（交给内核/IOCP）→ `unique_ptr` 接管（完成）→ 析构（清理）。这是零开销的所有权转移，比 `shared_ptr` 更精确。

#### 3.6.7 Connect 特殊处理

Connect 不能用 IOCP（Windows 的 `connect` 不支持 overlapped 完成），所以走 **WSAEventSelect + WSACreateEvent + ObjectWatcher** 路径：

[GetConnectEvent](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L611-L618)（L611-L618）：

```cpp
HANDLE CoreImpl::GetConnectEvent() {
  if (!connect_event_.is_valid()) {
    connect_event_.Set(::WSACreateEvent());
    ::WSAEventSelect(socket_->socket_, connect_event_.get(), FD_CONNECT);
  }
  return connect_event_.get();
}
```

[WatchForConnect](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L620-L623)（L620-L623）：

```cpp
void CoreImpl::WatchForConnect() {
  CHECK(connect_event_.is_valid());
  connect_watcher_.StartWatchingOnce(connect_event_.get(), this);
}
```

[OnObjectSignaled](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L629-L638)（L629-L638）：

```cpp
void CoreImpl::OnObjectSignaled(HANDLE object) {
  CHECK_EQ(object, connect_event_.get());
  StopWatchingAndCloseConnectEvent();
  socket_->DidCompleteConnect();
}
```

流程：`WSACreateEvent` 创建事件 → `WSAEventSelect` 绑定 `FD_CONNECT` 事件 → `ObjectWatcher` 监听事件句柄 → connect 完成时事件被 signal → `OnObjectSignaled` → `DidCompleteConnect`。这是 Reactor 风格（等事件通知），但只用于 connect，read/write 仍走 IOCP Proactor。

---

## 四、异步 IO 三段式模式（核心中的核心）

这是从 [http_server.cc](file:///workspace/net/server/http_server.cc) 提炼出的最值得复用的模式。每个 IO 操作（accept / read / write）都拆成 **三个函数**，命名与职责固定：

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

- **Accept**: [DoAcceptLoop / OnAcceptCompleted / HandleAcceptResult](file:///workspace/net/server/http_server.cc#L180-L211)
- **Read**: [DoReadLoop / OnReadCompleted / HandleReadResult](file:///workspace/net/server/http_server.cc#L213-L332)
- **Write**: [DoWriteLoop / OnWriteCompleted / HandleWriteResult](file:///workspace/net/server/http_server.cc#L334-L371)

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

所有异步回调都用 `weak_ptr_factory_.GetWeakPtr()` 绑定，例如 [http_server.cc#L65-L67](file:///workspace/net/server/http_server.cc#L65-L67):

```cpp
base::SingleThreadTaskRunner::GetCurrentDefault()->PostTask(
    FROM_HERE, base::BindOnce(&HttpServer::DoAcceptLoop,
                              weak_ptr_factory_.GetWeakPtr()));
```

`WeakPtrFactory` 成员放在对象**最后**（[http_server.h#L147](file:///workspace/net/server/http_server.h#L147) `base::WeakPtrFactory<HttpServer> weak_ptr_factory_{this};`），保证析构时先失效所有 WeakPtr，已 Post 的回调不会在对象销毁后触发。这是 Chromium 异步代码的标配。

### 5.2 延迟销毁：避免回调栈中的悬空指针

[http_server.cc#L144-L162](file:///workspace/net/server/http_server.cc#L144-L162) 的 `Close()` 是经典案例：

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

配合 `HasClosedConnection()`（[L540-L542](file:///workspace/net/server/http_server.cc#L540-L542)）在每次 delegate 回调后检查"连接是否已被 Close"，决定是否提前退出循环。

### 5.3 IOBuffer 所有权转移

见 [3.2 节](#32-iobuffer引用计数的缓冲区层级)。要点：传给异步 IO 的 IOBuffer，在操作完成前不能动；取消后要释放引用、永不再用。

---

## 六、实战案例剖析：HttpServer 完整流程

### 6.1 启动与 Accept 循环

[构造函数](file:///workspace/net/server/http_server.cc#L59-L68)：保存 `server_socket_`（已 listen 未 accept），**PostTask 到下一轮**才开始 `DoAcceptLoop`——避免 delegate 还没准备好就收到回调。

`DoAcceptLoop` 循环 `Accept`，每来一个连接：
1. `HandleAcceptResult` 新建 `HttpConnection(++last_id_, std::move(accepted_socket_))`，存入 `id_to_connection_` map；
2. 调 `delegate_->OnConnect(id)`；
3. 紧接着对该连接启动 `DoReadLoop`。

### 6.2 Read 循环 + 缓冲管理

[DoReadLoop](file:///workspace/net/server/http_server.cc#L213-L231) + [HttpConnection::ReadIOBuffer](file:///workspace/net/server/http_connection.h#L30-L71):

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

[HandleReadResult](file:///workspace/net/server/http_server.cc#L242-L332) 在 `while (!read_buf->readable_bytes().empty())` 里循环解析：先试 HTTP 头（状态机解析器在 [L373-L527](file:///workspace/net/server/http_server.cc#L373-L527)），如果是 WebSocket 升级请求则创建 `WebSocket` 并交给它解析。解析不完（`pos == 0`）就 break 等更多数据；解析出错就 Close。

### 6.3 Write 循环 + 队列缓冲

[DoWriteLoop](file:///workspace/net/server/http_server.cc#L334-L349) + [QueuedWriteIOBuffer](file:///workspace/net/server/http_connection.h#L76-L117):

写缓冲用 `base::queue<std::unique_ptr<std::string>>` 维护待写分块——**指针稳定性**很重要，因为 `IOBuffer::data()` 会把分块的裸指针交给底层 socket，分块不能在写完前移动或销毁。

`SendRaw` ([L95-L105](file:///workspace/net/server/http_server.cc#L95-L105)) 的细节值得学：先 `Append` 进队列，**只有当前没在写时才启动 `DoWriteLoop`**（`writing_in_progress = !write_buf->IsEmpty()`）。这保证了多次 `SendRaw` 不会并发发起多个 Write，而是排队串行写。

### 6.4 连接关闭的对称性

读出错 / 写出错 / 解析出错 / 主动 Close，最终都汇到 `Close(connection_id)`，走 [5.2 节](#52-延迟销毁避免回调栈中的悬空指针)的延迟销毁流程。**单一关闭出口**是避免资源泄漏的关键。

### 6.5 Chromium 调用链路

前面 6.1-6.4 是 `net/server` 的"服务器侧"用法（直接 `Accept`）。真实的 Chromium 浏览器是"客户端侧"——发起 HTTP 请求时，从 `URLRequest` 到 syscall 要经过一长串调用链。本节梳理这条链路。

#### 6.5.1 完整调用栈图

```
URLRequest::Start()
  └─ HttpStreamFactory::Job::DoLoop()
       └─ DoInitConnectionImplHttp()                          // 选 socket 池
            └─ InitSocketHandleForHttpRequest()               // client_socket_pool_manager.cc L214
                 └─ InitSocketPoolHelper()                    // L81 构造 GroupId + 选 pool
                      └─ ClientSocketHandle::Init()           // client_socket_handle.cc L31
                           └─ ClientSocketPool::RequestSocket()
                                ├─ 查 idle 健康队列 → 命中则直接返回（复用）
                                ├─ 未达上限 → 创建 ConnectJob
                                │    └─ ConnectJob::Connect()
                                │         ├─ HostResolver::Resolve()    // DNS
                                │         ├─ ClientSocketFactory::CreateTransportClientSocket()
                                │         │    └─ TCPClientSocket → TCPSocket → syscall connect()
                                │         └─ (若 https) SSLClientSocket::Connect()  // TLS 握手
                                └─ 达上限 → 进 pending 队列等别人释放
       └─ (连接就绪后) HttpStreamParser::SendRequest / ReadResponseHeaders
            └─ stream_socket_->Write() / Read()               // http_stream_parser.cc L451/L594
                 └─ TCPSocket → SocketPosix/CoreImpl → syscall
  └─ (用完) ClientSocketHandle::Reset()                       // client_socket_handle.cc L74
       └─ pool_->ReleaseSocket()（健康）或 socket()->Disconnect()（坏连接）
```

#### 6.5.2 三层池化 + GroupId 分组

**第一层：ClientSocketPoolManager 按 proxy chain 选 pool**。[InitSocketPoolHelper](file:///workspace/net/socket/client_socket_pool_manager.cc#L81-L133)（L81-L133）：

```cpp
ClientSocketPool* pool =
    session->GetSocketPool(socket_pool_type, proxy_info.proxy_chain());  // L112-L113
```

不同 proxy chain（直连 / HTTP 代理 / SOCKS 代理）对应不同 pool 实例，互不干扰。

**第二层：具体 ClientSocketPool 管 idle 队列 + 并发上限**。如 `TransportClientSocketPool` 管直连 TCP socket 的复用，`SSLClientSocketPool` 管 TLS socket 的复用。

**第三层：GroupId 按 origin 分组**。[client_socket_pool.h#L117-L119](file:///workspace/net/socket/client_socket_pool.h#L117-L119)：

```cpp
// Group ID for a socket request. Requests with the same group ID are
// considered indistinguishable.
class NET_EXPORT GroupId {
```

[GroupId 构造](file:///workspace/net/socket/client_socket_pool_manager.cc#L106-L108)（L106-L108）：

```cpp
ClientSocketPool::GroupId connection_group(
    std::move(endpoint), privacy_mode,
    std::move(network_anonymization_key), secure_dns_policy,
    disable_cert_network_fetches, target_network);
```

GroupId 由 6 个维度组成：`SchemeHostPort`（目的地址）+ `PrivacyMode`（隐私模式）+ `NetworkAnonymizationKey`（网络隔离键）+ `SecureDnsPolicy`（DoH 策略）+ `disable_cert_network_fetches` + `target_network`。只有这 6 维全相同的请求才被视为"可互换"，能复用彼此的 socket。

#### 6.5.3 RequestSocket 复用优先

[ClientSocketHandle::Init](file:///workspace/net/socket/client_socket_handle.cc#L31-L61)（L31-L61）把请求交给 pool：

```cpp
int rv = pool_->RequestSocket(
    group_id, std::move(socket_params), ...,
    this, std::move(io_complete_callback), ...);  // L51-L54
if (rv == ERR_IO_PENDING) {
  callback_ = std::move(callback);                 // L55-L56 异步等
} else {
  HandleInitCompletion(rv);                        // 同步完成（复用命中）
}
```

pool 内部 `RequestSocket` 的三路决策：
1. **先查 idle 健康队列**：有同 group 的空闲 socket 且健康（[transport_client_socket_pool.cc#L113-L125](file:///workspace/net/socket/transport_client_socket_pool.cc#L113-L125) 的 `IdleSocket::IsUsable`），直接复用，返回 OK。
2. **没空闲且未达上限**：创建 `ConnectJob` 异步建连，返回 `ERR_IO_PENDING`。
3. **已达上限**：进 pending 队列，等别人 `ReleaseSocket` 后唤醒，返回 `ERR_IO_PENDING`。

`IdleSocket::IsUsable` 的健康检查（[L114-L115](file:///workspace/net/socket/transport_client_socket_pool.cc#L114-L115)）关键规则：**用过的 socket 如果收到非预期数据则不健康**（脏数据会被误认为下一个响应的开头）。但从未用过的 preconnect socket 即使有未读数据也可用（可能是 SPDY SETTINGS 帧）。

#### 6.5.4 ConnectJob 抽象建连

`ConnectJob` 把"建连"封装成异步任务，对 pool 隐藏细节：
- `TransportConnectJob`：DNS 解析 + TCP connect
- `SSLConnectJob`：在 TransportConnectJob 之上加 TLS 握手
- `HttpProxyConnectJob` / `SOCKSConnectJob`：代理隧道

分层 Job 串联：底层 Job 完成（TCP 通了）→ 上层 Job 接管（SSL 握手）→ 全部完成才交给 pool。pool 只管"给我一个连好的 socket"，不关心中间经过几层。

#### 6.5.5 Handle 解耦

上层（`HttpStreamFactory::Job`）持 `unique_ptr<ClientSocketHandle>`，不直接持有 socket。[client_socket_handle.h#L39-L44](file:///workspace/net/socket/client_socket_handle.h#L39-L44) 的类注释：

```cpp
// A container for a StreamSocket.
//
// The handle's |group_id| uniquely identifies the origin and type of the
// connection.  It is used by the ClientSocketPool to group similar connected
// client socket objects.
```

Handle 是上层和 pool 之间的"中间人"：上层用 Handle 拿 socket、用完通过 Handle 还给 pool。Handle 的 `Init`（[L83-L93](file:///workspace/net/socket/client_socket_handle.h#L83-L93)）向 pool 请求，`Reset`（[L101-L109](file:///workspace/net/socket/client_socket_handle.h#L101-L109)）归还。这种解耦让上层不感知池化细节，pool 也不感知上层的业务逻辑。

#### 6.5.6 IO 阶段 HttpStreamParser 复用三段式

连接建立后进入 HTTP 收发阶段，[HttpStreamParser](file:///workspace/net/http/http_stream_parser.cc) 复用 socket 的三段式 IO：

**发请求头**（[L451-L453](file:///workspace/net/http/http_stream_parser.cc#L451-L453)）：

```cpp
return stream_socket_->Write(
    request_headers_.get(), bytes_remaining, io_callback_,
    NetworkTrafficAnnotationTag(traffic_annotation_));
```

**读响应**（[L594-L595](file:///workspace/net/http/http_stream_parser.cc#L594-L595)）：

```cpp
return stream_socket_->Read(read_buf_.get(), read_buf_->RemainingCapacity(),
                            io_callback_);
```

`io_callback_` 是同一个回调，绑到 `HttpStreamParser` 的状态机。`HttpBasicStream` 只是薄包装，把调用委托给 `parser()`（[http_basic_stream.cc#L66-L79](file:///workspace/net/http/http_basic_stream.cc#L66-L79)）：

```cpp
int HttpBasicStream::SendRequest(...) {
  return parser()->SendRequest(...);           // L66-L69
}
int HttpBasicStream::ReadResponseHeaders(...) {
  return parser()->ReadResponseHeaders(...);   // L72-L73
}
int HttpBasicStream::ReadResponseBody(...) {
  return parser()->ReadResponseBody(...);      // L76-L79
}
```

注意这里 `stream_socket_` 就是前面 pool 给的 socket（可能是复用的），上层完全无感知是新建还是复用——**池化对 IO 代码透明**。

#### 6.5.7 Reset 双语义

用完连接后调 [Reset](file:///workspace/net/socket/client_socket_handle.h#L101-L109)（L101-L109），注释点明两种语义：

```cpp
// An initialized handle can be reset, which causes it to return to the
// un-initialized state.  This releases the underlying socket, which in the
// case of a socket that still has an established connection, indicates that
// the socket may be kept alive for use by a subsequent ClientSocketHandle.
//
// NOTE: To prevent the socket from being kept alive, be sure to call its
// Disconnect method.  This will result in the ClientSocketPool deleting the
// StreamSocket.
void Reset() override;
```

- **健康归还**（keep-alive 复用）：socket 没出错，`Reset` 把 socket 还给 pool 的 idle 队列，下次同 group 请求可复用。走 [ResetInternal](file:///workspace/net/socket/client_socket_handle.cc#L172-L208) 的 `pool_->ReleaseSocket`（[L184](file:///workspace/net/socket/client_socket_handle.cc#L184)）。
- **坏连接主动 Disconnect**：socket 出错了，调 `ResetAndCloseSocket`（[L79-L85](file:///workspace/net/socket/client_socket_handle.cc#L79-L85)），先 `socket()->Disconnect()`（[L81](file:///workspace/net/socket/client_socket_handle.cc#L81)）再 reset，pool 收到的是已断开的 socket，直接删除不进 idle 队列。**防止坏连接污染池**。

#### 6.5.8 设计要点表

| 设计 | 作用 |
|------|------|
| 三层池化（Manager / Pool / Group） | 按 proxy / origin 维度隔离，复用粒度精确 |
| GroupId 六维分组 | 同 origin + 同隐私/隔离策略的请求才复用，避免信息泄漏 |
| idle 健康检查 | 脏数据 socket 不复用，preconnect socket 宽容 |
| ConnectJob 抽象 | pool 不关心建连细节（DNS/TCP/SSL/代理），只拿结果 |
| Handle 中间人 | 上层与 pool 解耦，上层只管用和还 |
| Reset 双语义 | 健康归还复用、坏连接 Disconnect 防污染 |
| IO 阶段透明 | HttpStreamParser 不感知 socket 是新建还是复用 |

---

## 七、可复用到自己项目的设计要点清单

把上面提炼成可操作的 checklist，落地自己的网络服务时逐条对照：

1. **统一返回值约定**：IO 函数返回 `int`，`0` 成功 / 正数字节 / 负数错误 / 一个特殊值（如 `-1`）表示 pending。
2. **异步三段式**：每个 IO 操作拆 `DoXxxLoop` / `OnXxxCompleted` / `HandleXxxResult`，`do-while` 统一同步与异步路径。
3. **缓冲区引用计数 + 视图分离**：内存用 `shared_ptr`/`scoped_refptr` 保活，可读写位置用 offset/span 表达；按用途分多个 buffer 类，而非一个全能类。
4. **回调绑 WeakPtr**：异步回调一律 `BindOnce` + `GetWeakPtr()`，`WeakPtrFactory` 放成员最后。
5. **延迟销毁危险对象**：回调栈中可能被引用的对象，Close 时移入"待销毁列表"，PostTask 到下一轮再删。
6. **单一关闭出口**：所有错误路径汇到一个 `Close()`，避免散落的资源清理。
7. **写排队串行化**：用队列缓冲 + "仅当未在写时才启动写循环"的判断，避免并发 Write。
8. **分层依赖抽象**：上层只持有 socket 的抽象接口（`unique_ptr<ServerSocket>`），不依赖具体实现。
9. **背压靠 pending**：`ERR_IO_PENDING` 时退出循环还控制权给消息循环，天然不阻塞、不递归爆栈。
10. **错误码分段**：按系统/连接/证书/HTTP 等分段编号，便于归类与映射 OS 错误。
11. **fd 创建即非阻塞**：socket 一打开就 `SetNonBlocking`，所有 syscall 默认非阻塞，是 Reactor 模式的前提（见 [socket_posix.cc#L125](file:///workspace/net/socket/socket_posix.cc#L125)）。
12. **Reactor 模式**：非阻塞 syscall + `EAGAIN`/`EINPROGRESS` 映射成 `ERR_IO_PENDING` + 注册 fd watcher 等回调（见 [socket_posix.cc#L82-L83](file:///workspace/net/socket/socket_posix.cc#L82-L83)、[L329-L331](file:///workspace/net/socket/socket_posix.cc#L329-L331)）。
13. **HANDLE_EINTR**：被信号中断的 syscall 静默重试，对上层完全透明（见 [socket_posix.cc#L463](file:///workspace/net/socket/socket_posix.cc#L463)、[L519](file:///workspace/net/socket/socket_posix.cc#L519)）。
14. **全双工独立 watcher**：read 和 write 用不同 watcher，可同时 pending 互不干扰（见 [socket_posix.cc#L99-L103](file:///workspace/net/socket/socket_posix.cc#L99-L103)）。
15. **Unretained vs WeakPtr 生命周期推理**：当回调发源体是 `this` 的成员（成员先于 this 析构且析构时停 watcher），可用 `Unretained` 省去 `WeakPtrFactory` 开销（见 [tcp_server_socket.cc#L96-L97](file:///workspace/net/socket/tcp_server_socket.cc#L96-L97) 注释）。
16. ★ **Proactor 而非 Reactor**：Windows IOCP 在发起 IO 时就把 buffer 交给内核，完成通知时数据已就位，与 POSIX "通知可读后再 read" 不同（见 3.6.1 对比表）。
17. ★ **FILE_SKIP_COMPLETION_PORT_ON_SUCCESS 优化**：同步完成的 IO 不投完成包，省一次 PostTask 开销（见 [tcp_socket_io_completion_port_win.cc#L370-L373](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L370-L373)）。
18. ★ **buffer 发起时交内核**：`WSASend`/`WSARecv` 异步时把 IOBuffer 存进 `IOContext` 保活，完成前不能动（见 [L320-L322](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L320-L322)）。
19. ★ **ReadIfReady 零字节 read**：用零字节 overlapped read 等"可读"通知，不持有调用者 buffer，可随时取消（见 [L515-L524](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L515-L524)）。
20. ★ **context 用 unique_ptr 所有权流转**：发起时 `release()` 交给 IOCP，完成时 `unique_ptr` 接管，RAII 自动清理，零泄漏（见 [L647](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L647)）。
21. ★ **core_keep_alive 自保活**：IOContext 持 `scoped_refptr<CoreImpl>`，socket 析构后 IO 完成时 Core 仍活着，`Detach` 清空指针防悬空（见 [L132-L134](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L132-L134)、[L602-L609](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L602-L609)）。
22. ★ **connect 不能用 IOCP**：用 `WSAEventSelect` + `WSACreateEvent` + `ObjectWatcher` 走事件通知（见 [L611-L623](file:///workspace/net/socket/tcp_socket_io_completion_port_win.cc#L611-L623)）。
23. **Handle 解耦**：上层持 `unique_ptr<ClientSocketHandle>` 而非裸 socket，用完 Reset 归还，不感知池化细节（见 [client_socket_handle.h#L39-L44](file:///workspace/net/socket/client_socket_handle.h#L39-L44)）。
24. **socket 池化分组复用**：按 proxy chain 选 pool + GroupId 六维分组，同 origin 同策略的请求复用 idle socket（见 [client_socket_pool_manager.cc#L106-L113](file:///workspace/net/socket/client_socket_pool_manager.cc#L106-L113)）。
25. **ConnectJob 抽象建连**：DNS+TCP+SSL+代理封装成异步 Job 串联，pool 不关心建连细节只拿结果。
26. **坏连接主动 Disconnect**：socket 出错时 `ResetAndCloseSocket` 先 `Disconnect` 再还池，防止坏连接进 idle 队列被复用（见 [client_socket_handle.cc#L79-L85](file:///workspace/net/socket/client_socket_handle.cc#L79-L85)）。

---

## 八、待补充

- [ ] 阶段 5：套用上述模式写一个自包含的 TCP echo server demo（Windows IOCP 版本优先，POSIX 版本作为对照）
- [ ] 阶段 6：集成到自己的项目（待领导安排项目方向）
- [ ] 后续深入：Happy Eyeballs（IPv4/IPv6 竞速连接，`TCPClientSocket` 多地址尝试）
- [ ] 后续深入：`SSLClientSocket` 分层（TLS 握手状态机、证书验证、session resumption）
- [ ] 后续深入：socket_pool 连接复用细节（idle 超时回收、preconnect 预连接、pending 队列优先级）
- [ ] Windows 专项：`TCPSocketDefaultWin`（旧 ObjectWatcher 实现）与 `TcpSocketIoCompletionPortWin`（新 IOCP 实现）的迁移历史与性能对比
- [ ] Windows 专项：`udp_socket_win.cc`（UDP 的 IOCP 实现与 TCP 的差异）

---

## 九、本次 net/socket 代码升级分析（提交 4ba3f2d）

> 本仓库在 `studysocket` 分支基础上，对 `net/socket/` 做了一次整体升级（提交 `4ba3f2d Fix:新版本`，从 Chromium 主线同步最新实现）。本章分析 socket 层的具体变化与好处。

### 9.1 新增文件（4 个）

**1. [delayed_socket_config.h](file:///workspace/net/socket/delayed_socket_config.h) + [delayed_stream_socket.{h,cc}](file:///workspace/net/socket/delayed_stream_socket.h)**
- **作用**：`StreamSocket` 的包装器，注入真实的网络延迟和带宽限制
- **架构**（[头文件注释 L36-L87](file:///workspace/net/socket/delayed_stream_socket.h#L36-L87)）：
  - 读路径：`OS Socket → [Download BottleneckBuffer] → Consumer`，缓冲满了就停止读 OS socket，自然形成 TCP 接收窗口收缩的背压
  - 写路径：`Producer → [Upload BottleneckBuffer] → OS Socket`，缓冲满了 `Write()` 返回 `ERR_IO_PENDING`，从源头限流
  - 延迟模型：Connect 延迟一个 RTT、每 chunk 标记半 RTT、`BandwidthThrottle` 处理带宽限制
- **好处**：让测试能真实模拟慢速网络、高延迟链路，无需真实网络环境。**这正是第四章异步三段式与背压设计的完整演练场**——验证 `ERR_IO_PENDING` 退出 + 回调重入是否真的工作。

**2. [read_multiple_emulator.{h,cc}](file:///workspace/net/socket/read_multiple_emulator.h)**
- **作用**：用 `Read()` 模拟 `ReadMultiple()`（[头文件注释 L23-L29](file:///workspace/net/socket/read_multiple_emulator.h#L23-L29)）
- **背景**：`QuicUseReadMultiple` feature 开启时，部分 DatagramClientSocket 还没原生实现 `ReadMultiple()`
- **好处**：临时兼容，避免崩溃。标记为 TODO，等所有 socket 实现原生 `ReadMultiple()` 后删除。

### 9.2 关键修改：连接建立路径的升级

**1. Happy Eyeballs v2 与动态 IPv6 回退时间**（[transport_connect_job.cc#L413-L418](file:///workspace/net/socket/transport_connect_job.cc#L413-L418)）
```cpp
// 旧：静态常量 kIPv6FallbackTime = 300ms
// 新：动态计算
base::TimeDelta fallback_time = TcpConnectJob::GetIPv6FallbackTime(
    common_connect_job_params(), params_.get());
```
- **变化**：IPv6 连接失败后启动 IPv4 备用连接的延迟时间，从静态 300ms 改为基于 RTT 估算（`kIPv6FallbackBasedOnRTT` feature）
- **好处**：网络快时提前回退、网络慢时延后回退，比固定 300ms 更智能。**呼应学习笔记 6.5.4 节的 ConnectJob 抽象**——同一接口下升级建连策略，上层无感知。

**2. Stale DNS 感知与禁用**（[tcp_connect_job.h#L426-L438](file:///workspace/net/socket/tcp_connect_job.h#L426-L438)、[ssl_connect_job.h](file:///workspace/net/socket/ssl_connect_job.h)）
```cpp
// tcp_connect_job.h 新增成员
std::optional<bool> is_connected_via_stale_dns_;
const bool disable_stale_dns_;
// ssl_connect_job.h 新增成员
bool is_connected_via_stale_dns_ = false;
bool disable_stale_dns_ = false;
```
- **变化**：连接任务现在记录"是否用了过时 DNS 结果"，并支持"禁用过时 DNS"模式
- **好处**：SSL 连接失败时可以用 fresh DNS 重试，避免因 DNS 缓存过期导致连到错误服务器。**这是连接可靠性的重要提升**。

**3. ECH（Encrypted Client Hello）按域禁用**（[transport_connect_job.cc#L548-L556](file:///workspace/net/socket/transport_connect_job.cc#L548-L556)）
```cpp
// 新增 ssl_config_service 按域查询 EchMode
ssl_client_context->ssl_config_service()->GetEchMode(
    scheme_host_port->host()) == EchMode::kDisabled
```
- **变化**：ECH 现在可以按域名单独禁用（不只是全局开关）
- **好处**：对不支持 ECH 或有兼容性问题的域名精准降级，不影响其他域名。

### 9.3 关键修改：连接池管理的重构

**1. `SocketPoolState` → `SocketPoolExpandability` 重命名**（[client_socket_pool.h#L427-L456](file:///workspace/net/socket/client_socket_pool.h#L427-L456)）
```cpp
// 旧：SocketPoolState state_ = SocketPoolState::kUncapped;
// 新：SocketPoolExpandability expandability_ = SocketPoolExpandability::kUncapped;
SocketPoolState StateForTest() const { return State(); }    // 旧
SocketPoolExpandability ExpandabilityForTest() const { return Expandability(); }  // 新
```
- **变化**：把"池状态"概念重命名为"可扩展性"，语义更精确
- **好处**：`State` 太泛（可指任何状态），`Expandability` 明确表达"池还能否新增 socket"这一具体语义。命名改进提升可读性。

**2. `additional_capacity_` 从 const 变为可变**（[client_socket_pool.h#L455](file:///workspace/net/socket/client_socket_pool.h#L455)）
```cpp
const SocketPoolAdditionalCapacity additional_capacity_;    // 旧
SocketPoolAdditionalCapacity additional_capacity_;          // 新
```
- 配套新增 `SetAdditionalCapacityForTest()`（[L372-L374](file:///workspace/net/socket/client_socket_pool.h#L372-L374)）
- **好处**：运行时可以动态调整池的额外容量（之前只能在构造时设定）。便于测试和动态调优。

**3. 构造函数简化：`additional_capacity` 不再是构造参数**（[transport_client_socket_pool.h#L155-L173](file:///workspace/net/socket/transport_client_socket_pool.h#L155-L173)）
- 所有 pool 构造函数都移除了 `SocketPoolAdditionalCapacity additional_capacity` 参数
- **好处**：构造时不需要决定容量策略，统一在构造后用 `SetAdditionalCapacityForTest()` 设置。降低构造复杂度。

### 9.4 关键修改：WebSocket 端点锁的隔离增强

**[websocket_endpoint_lock_manager.h](file:///workspace/net/socket/websocket_endpoint_lock_manager.h) 关键变化**：
```cpp
// 旧：EndpointLock(WebSocketEndpointLockManager*, const IPEndPoint&);
// 新：EndpointLock(WebSocketEndpointLockManager*, const IPEndPoint&,
//                 const NetworkAnonymizationKey&);
void UnlockEndpoint(const IPEndPoint& endpoint);    // 旧
void UnlockEndpoint(const IPEndPoint& endpoint,
                    const NetworkAnonymizationKey& network_anonymization_key);  // 新
```
- **变化**：WebSocket 端点锁现在按 `IPEndPoint + NetworkAnonymizationKey` 双键区分
- **好处**：**呼应学习笔记 6.5.2 节的 GroupId 分组复用**——WebSocket 同样遵守"不同网络隔离域不共享连接"的隐私约束。之前只按 IP 端点锁，可能导致不同 NAK 的请求互相阻塞；现在精确隔离，提升并发。

### 9.5 升级对本项目（Windows 平台）的意义

| 变化 | 对 Windows 项目的意义 |
|------|---------------------|
| `DelayedStreamSocket` | Windows IOCP 环境下可注入延迟/带宽限制做真实测试，验证 IOCP 的背压处理 |
| `ReadMultipleEmulator` | UDP socket 若启用 `ReadMultiple`，Windows 版可用此模拟器临时兼容 |
| 动态 IPv6 回退 | Windows 网络栈的 Happy Eyeballs 更智能 |
| Stale DNS 感知 | Windows 下 SSL 连接失败可用 fresh DNS 重试，提升可靠性 |
| `SocketPoolExpandability` | 命名改进提升 Windows 移植代码的可读性 |
| WebSocket NAK 隔离 | Windows 下 WebSocket 连接池的隐私隔离更精确 |

### 9.6 与学习笔记的呼应

本次升级**正好印证了学习笔记里的几条核心设计**：

- **checklist 第 25 条"ConnectJob 抽象建连"**：Happy Eyeballs v2 升级、Stale DNS 处理都是在 ConnectJob 内部演进，**上层 pool 和 HttpStreamFactory 完全无感知**——这正是分层抽象的价值。
- **checklist 第 24 条"socket 池化 + 分组复用"**：WebSocket 端点锁加入 NetworkAnonymizationKey，是 GroupId 分组思想在 WebSocket 层的延伸。
- **6.5.3 节"RequestSocket 复用优先"**：`SocketPoolExpandability` 重命名让"池能否扩展"这一复用决策语义更清晰。
- **第四章"异步三段式"**：`DelayedStreamSocket` 是三段式 + 背压设计的大规模演练，可作学习样本。
