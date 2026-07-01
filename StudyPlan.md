# Chromium net 模块 Socket 编程学习笔记

> 目标：理解 Chromium `net` 模块的分层架构，吃透 socket 的封装、生命周期与异步 IO 模型，提炼出可复用到自己项目的现代 C++ 网络编程范式。
>
> 说明：本仓库目前只包含 `net/` 的部分子目录，**`net/socket/` 目录（`ServerSocket`/`StreamSocket`/`TCPServerSocket` 等底层封装）未入库**。因此 socket API 的"用法"主要从 [net/server/http_server.cc](net/server/http_server.cc) 这个真实用例中归纳；"设计思想"则从 [net/base/](net/base/) 的核心抽象（IOBuffer、错误码、回调）中提炼。后续若需要看底层 socket 封装本身的实现，需要补齐 `net/socket/` 目录。

---

## 一、学习路线（分阶段 + 进度跟踪）

| 阶段 | 主题 | 关键文件 | 状态 |
|------|------|----------|------|
| 0 | 环境与代码组织 | `net/README.md`、目录结构 | ✅ 已完成 |
| 1 | 核心抽象：错误码 / IOBuffer / 回调 | [net/base/net_errors.h](net/base/net_errors.h)、[net/base/io_buffer.h](net/base/io_buffer.h)、[net/base/completion_once_callback.h](net/base/completion_once_callback.h) | ✅ 已完成 |
| 2 | 异步 IO 三段式模式（Do/On/Handle） | [net/server/http_server.cc](net/server/http_server.cc) | ✅ 已完成 |
| 3 | 生命周期管理：WeakPtr + 延迟销毁 | [net/server/http_server.cc](net/server/http_server.cc) | ✅ 已完成 |
| 4 | 实战案例：HttpServer / HttpConnection / WebSocket | [net/server/](net/server/) | ✅ 已完成 |
| 5 | 落地 demo（套用模式写一个 TCP echo server） | — | ⏳ 待后续需求 |
| 6 | 集成到自己的项目 | — | ⏳ 待领导安排 |

> 进度更新约定：每完成一阶段把状态从 ⏳ 改为 ✅，并在对应章节补充笔记。

---

## 二、架构与分层概览

Chromium 的 `net` 模块是典型的分层设计，自底向上：

```
┌─────────────────────────────────────────────┐
│  应用层：net/server (HttpServer, WebSocket)   │  ← 本仓库有
├─────────────────────────────────────────────┤
│  协议层：net/http, net/spdy, net/quic, net/ssl│  ← 部分有
├─────────────────────────────────────────────┤
│  Socket 层：net/socket (StreamSocket 等)      │  ← 本仓库未包含
├─────────────────────────────────────────────┤
│  基础抽象：net/base (IOBuffer, errors, ...)   │  ← 本仓库有
└─────────────────────────────────────────────┘
```

- **net/base**：与协议无关的基础设施。错误码、IP 地址、IOBuffer、回调、网络变化通知等都在这里。是学习"设计思想"的最佳入口。
- **net/socket**：对原生 socket（POSIX / Winsock）的跨平台封装，定义了 `StreamSocket`、`ServerSocket`、`TCPServerSocket`、`SSLClientSocket` 等抽象。本仓库未包含，但从 `http_server.cc` 的 `#include "net/socket/server_socket.h"` 等可推断其接口。
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
6. **单一关闭出口**：所有错误路径汇到一个 `Close()`，避免散落的资源清理。
7. **写排队串行化**：用队列缓冲 + "仅当未在写时才启动写循环"的判断，避免并发 Write。
8. **分层依赖抽象**：上层只持有 socket 的抽象接口（`unique_ptr<ServerSocket>`），不依赖具体实现。
9. **背压靠 pending**：`ERR_IO_PENDING` 时退出循环还控制权给消息循环，天然不阻塞、不递归爆栈。
10. **错误码分段**：按系统/连接/证书/HTTP 等分段编号，便于归类与映射 OS 错误。

---

## 八、待补充

- [ ] 阶段 5：套用上述模式写一个自包含的 TCP echo server demo（待后续需求）
- [ ] 阶段 6：集成到自己的项目（待领导安排项目方向）
- [ ] 若需要深入底层 socket 封装本身，补齐 `net/socket/` 目录源码（`stream_socket.h`、`tcp_server_socket_posix.cc` 等），重点看 `Read`/`Write` 如何把 POSIX `recv`/`send` 的 `EWOULDBLOCK` 翻译成 `ERR_IO_PENDING` 并注册到 message loop。
