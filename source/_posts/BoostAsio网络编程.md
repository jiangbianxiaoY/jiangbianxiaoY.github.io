---
title: Boost.Asio 网络编程进阶：会话层、连接池、TCP/UDP 服务器完整教程
date: 2026-06-17 12:00:00
updated: 2026-06-17 12:00:00
categories:
  - C++
tags:
  - Boost
  - Asio
  - 网络编程
  - C++
  - 会话层
  - 连接池
  - TCP
  - UDP
---

## 一、引言

Boost.Asio 是 C++ 中最强大的异步网络库之一。本文从生产环境的角度，深入讲解如何构建 **会话层（Session Layer）**、**连接池（Connection Pool）**、**TCP 服务器** 和 **UDP 服务器** 的完整方案。

### 前置知识

- 熟悉 C++17/20 基础语法
- 了解 Boost.Asio 基本概念（io_context、socket、async_* 模式）
- 熟悉 CMake 构建系统

### 项目结构

```
asio_network/
├── CMakeLists.txt
├── include/
│   ├── session.hpp          # 会话层抽象
│   ├── protocol.hpp         # 协议定义
│   ├── connection_pool.hpp  # 连接池
│   ├── tcp_server.hpp       # TCP 服务器
│   └── udp_server.hpp       # UDP 服务器
└── examples/
    ├── tcp_echo.cpp
    └── udp_echo.cpp
```

## 二、协议设计

在设计会话层之前，先确定网络协议格式。我们采用最常见的 **长度前缀（Length-Prefixed）** 协议：

### 2.1 消息格式

```
+----------------+----------------+------------------+
| 消息长度 (4B)   | 消息类型 (1B)   | 消息负载 (N B)    |
+----------------+----------------+------------------+
```

### 2.2 协议定义

```cpp
// protocol.hpp
#pragma once
#include <boost/asio.hpp>
#include <vector>
#include <cstdint>
#include <functional>

namespace net {

namespace asio = boost::asio;

// 消息类型枚举
enum class MessageType : uint8_t {
    HEARTBEAT      = 0x01,
    DATA           = 0x02,
    CLOSE          = 0x03,
    ACK            = 0x04,
};

// 消息头部固定 5 字节
#pragma pack(push, 1)
struct MessageHeader {
    uint32_t length;      // 负载长度（网络字节序）
    uint8_t  type;        // 消息类型
};
#pragma pack(pop)

// 完整消息
struct Message {
    MessageHeader header;
    std::vector<char> body;

    Message() {
        header.length = 0;
        header.type = 0;
    }

    explicit Message(MessageType type, const void* data = nullptr,
                     size_t len = 0) {
        header.length = htonl(static_cast<uint32_t>(len));
        header.type   = static_cast<uint8_t>(type);
        if (data && len > 0) {
            body.resize(len);
            std::memcpy(body.data(), data, len);
        }
    }

    uint32_t body_length() const {
        return ntohl(header.length);
    }

    size_t total_length() const {
        return sizeof(MessageHeader) + body_length();
    }
};

// 序列化
inline std::vector<asio::const_buffer> to_buffers(const Message& msg) {
    return {
        asio::buffer(&msg.header, sizeof(MessageHeader)),
        asio::buffer(msg.body)
    };
}

} // namespace net
```

## 三、会话层（Session Layer）

会话层封装了单条连接的生命周期管理，包括建立、读写、保活、关闭等。

### 3.1 会话接口

```cpp
// session.hpp
#pragma once
#include "protocol.hpp"
#include <boost/asio.hpp>
#include <memory>
#include <functional>
#include <chrono>

namespace net {

// 会话回调
struct SessionCallbacks {
    std::function<void(const Message&)> on_message;   // 收到消息
    std::function<void(std::error_code)> on_error;    // 出错
    std::function<void()> on_close;                   // 主动关闭
};

// 会话类型
enum class SessionType {
    TCP_CLIENT,
    TCP_SERVER_SIDE,
    UDP,
};

// 会话基类
class Session : public std::enable_shared_from_this<Session> {
public:
    virtual ~Session() = default;

    SessionType type() const { return type_; }
    bool is_open() const { return open_; }

    virtual void start() = 0;
    virtual void close() = 0;
    virtual void send_message(const Message& msg) = 0;

    void set_callbacks(SessionCallbacks cbs) { cbs_ = std::move(cbs); }

    // 上次活跃时间
    std::chrono::steady_clock::time_point last_active() const {
        return last_active_;
    }

protected:
    Session(asio::io_context& ctx, SessionType type)
        : ctx_(ctx), type_(type), open_(false),
          heartbeat_timer_(ctx),
          last_active_(std::chrono::steady_clock::now()) {}

    asio::io_context& ctx_;
    SessionType type_;
    bool open_;
    SessionCallbacks cbs_;
    asio::steady_timer heartbeat_timer_;
    std::chrono::steady_clock::time_point last_active_;

    void update_active() {
        last_active_ = std::chrono::steady_clock::now();
    }

    // 心跳检测（默认 30s 无数据则断开）
    virtual void start_heartbeat(int interval_sec = 30) {
        heartbeat_timer_.expires_after(
            std::chrono::seconds(interval_sec));
        heartbeat_timer_.async_wait(
            [this, self = shared_from_this()](std::error_code ec) {
                if (ec) return;
                auto now = std::chrono::steady_clock::now();
                auto elapsed = std::chrono::duration_cast<
                    std::chrono::seconds>(
                    now - last_active_).count();
                if (elapsed >= interval_sec) {
                    if (cbs_.on_error) {
                        cbs_.on_error(
                            asio::error::timed_out);
                    }
                    close();
                } else {
                    start_heartbeat(interval_sec);
                }
            });
    }
};

} // namespace net
```

### 3.2 TCP 会话实现

```cpp
// tcp_session.hpp
#pragma once
#include "session.hpp"
#include <boost/asio.hpp>
#include <queue>

namespace net {

using tcp = asio::ip::tcp;

class TcpSession : public Session {
public:
    TcpSession(asio::io_context& ctx, tcp::socket socket,
               SessionType type = SessionType::TCP_SERVER_SIDE)
        : Session(ctx, type)
        , socket_(std::move(socket))
        , read_header_(true)
        , read_pos_(0) {}

    tcp::socket& socket() { return socket_; }

    void start() override {
        open_ = true;
        // 设置 TCP_NODELAY 禁用 Nagle 算法
        boost::system::error_code ec;
        socket_.set_option(tcp::no_delay(true), ec);
        do_read_header();
        start_heartbeat();
    }

    void close() override {
        if (!open_) return;
        open_ = false;
        heartbeat_timer_.cancel();
        boost::system::error_code ec;
        socket_.shutdown(tcp::socket::shutdown_both, ec);
        socket_.close(ec);
        if (cbs_.on_close) cbs_.on_close();
    }

    void send_message(const Message& msg) override {
        bool write_in_progress = !write_queue_.empty();
        write_queue_.push(msg);
        if (!write_in_progress) {
            do_write();
        }
    }

private:
    tcp::socket socket_;
    bool read_header_;
    MessageHeader incoming_header_;
    std::vector<char> incoming_body_;
    size_t read_pos_;

    std::queue<Message> write_queue_;

    void do_read_header() {
        read_header_ = true;
        read_pos_ = 0;
        asio::async_read(socket_,
            asio::buffer(&incoming_header_, sizeof(MessageHeader)),
            [this, self = shared_from_this()](
                std::error_code ec, size_t) {
                if (!ec) {
                    auto body_len = incoming_header_.body_length();
                    if (body_len > 0) {
                        incoming_body_.resize(body_len);
                        do_read_body();
                    } else {
                        on_message_received();
                        do_read_header();
                    }
                } else {
                    handle_error(ec);
                }
            });
    }

    void do_read_body() {
        read_header_ = false;
        asio::async_read(socket_,
            asio::buffer(incoming_body_),
            [this, self = shared_from_this()](
                std::error_code ec, size_t) {
                if (!ec) {
                    on_message_received();
                    do_read_header();
                } else {
                    handle_error(ec);
                }
            });
    }

    void do_write() {
        if (write_queue_.empty()) return;
        auto& msg = write_queue_.front();
        update_active();
        asio::async_write(socket_, to_buffers(msg),
            [this, self = shared_from_this()](
                std::error_code ec, size_t) {
                if (!ec) {
                    write_queue_.pop();
                    do_write();
                } else {
                    handle_error(ec);
                }
            });
    }

    void on_message_received() {
        update_active();
        Message msg;
        msg.header = incoming_header_;
        msg.body = std::move(incoming_body_);
        if (msg.header.type ==
            static_cast<uint8_t>(MessageType::HEARTBEAT)) {
            // 自动回复心跳
            send_message(Message(MessageType::HEARTBEAT));
            return;
        }
        if (cbs_.on_message) {
            cbs_.on_message(msg);
        }
    }

    void handle_error(std::error_code ec) {
        open_ = false;
        if (ec != asio::error::operation_aborted) {
            if (cbs_.on_error) cbs_.on_error(ec);
        }
        close();
    }
};

} // namespace net
```

> **关键设计**：
>
> - **写队列**：确保异步写操作不会交错，避免数据混叠
> - **心跳自动回复**：收到心跳包后自动回复，业务层无需关心
> - **TCP_NODELAY**：禁用 Nagle 算法，降低小包延迟
> - **读状态机**：读头 → 读体 → 读头的状态切换，准确处理粘包

### 3.3 创建 TcpSession 的工厂函数

```cpp
// tcp_session.hpp 尾部追加
namespace net {

inline std::shared_ptr<TcpSession> make_tcp_session(
    asio::io_context& ctx, tcp::socket socket,
    SessionCallbacks cbs) {
    auto session = std::make_shared<TcpSession>(
        ctx, std::move(socket));
    session->set_callbacks(std::move(cbs));
    return session;
}

} // namespace net
```

## 四、连接池（Connection Pool）

连接池用于管理一组 TCP 连接，支持自动扩容、空闲回收、健康检查和负载均衡。

### 4.1 连接池类

```cpp
// connection_pool.hpp
#pragma once
#include "tcp_session.hpp"
#include <boost/asio.hpp>
#include <vector>
#include <deque>
#include <mutex>
#include <atomic>
#include <memory>

namespace net {

struct PoolConfig {
    size_t min_connections = 4;        // 最小连接数
    size_t max_connections = 32;       // 最大连接数
    int idle_timeout_sec = 60;         // 空闲超时（秒）
    int health_check_sec = 30;         // 健康检查间隔（秒）
    int connect_timeout_ms = 5000;     // 连接超时（毫秒）
    std::string host;                  // 目标主机
    uint16_t port = 0;                 // 目标端口
};

class ConnectionPool : public std::enable_shared_from_this<ConnectionPool> {
public:
    ConnectionPool(asio::io_context& ctx, PoolConfig cfg)
        : ctx_(ctx), cfg_(std::move(cfg)),
          health_timer_(ctx),
          running_(false) {}

    ~ConnectionPool() { stop(); }

    // 启动连接池（预热 min_connections 个连接）
    void start() {
        running_ = true;
        for (size_t i = 0; i < cfg_.min_connections; ++i) {
            create_connection();
        }
        start_health_check();
    }

    // 停止连接池
    void stop() {
        running_ = false;
        health_timer_.cancel();
        {
            std::lock_guard<std::mutex> lock(mutex_);
            for (auto& session : idle_sessions_) {
                session->close();
            }
            idle_sessions_.clear();
            for (auto& session : active_sessions_) {
                session->close();
            }
            active_sessions_.clear();
        }
    }

    // 获取一个空闲连接（RAII 包装）
    class PooledSession {
    public:
        PooledSession() : pool_(nullptr) {}

        PooledSession(std::shared_ptr<ConnectionPool> pool,
                      std::shared_ptr<TcpSession> session)
            : pool_(std::move(pool)), session_(std::move(session)) {}

        ~PooledSession() {
            if (pool_ && session_) {
                pool_->release(session_);
            }
        }

        PooledSession(PooledSession&& other) noexcept
            : pool_(std::move(other.pool_))
            , session_(std::move(other.session_)) {}

        PooledSession& operator=(PooledSession&& other) noexcept {
            if (this != &other) {
                if (pool_ && session_) pool_->release(session_);
                pool_ = std::move(other.pool_);
                session_ = std::move(other.session_);
            }
            return *this;
        }

        PooledSession(const PooledSession&) = delete;
        PooledSession& operator=(const PooledSession&) = delete;

        TcpSession* operator->() { return session_.get(); }
        TcpSession& operator*() { return *session_; }
        explicit operator bool() const { return session_ != nullptr; }

    private:
        std::shared_ptr<ConnectionPool> pool_;
        std::shared_ptr<TcpSession> session_;
    };

    // 异步获取空闲连接
    void acquire(std::function<void(PooledSession)> callback) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (!idle_sessions_.empty()) {
            auto session = std::move(idle_sessions_.front());
            idle_sessions_.pop_front();
            active_sessions_.insert(session);
            asio::post(ctx_,
                [cb = std::move(callback),
                 s = std::move(session),
                 self = shared_from_this()]() mutable {
                    cb(PooledSession(self, std::move(s)));
                });
        } else if (total_connections() < cfg_.max_connections) {
            // 异步创建新连接
            pending_callbacks_.push_back(std::move(callback));
            create_connection();
        } else {
            // 达到上限，等待释放
            pending_callbacks_.push_back(std::move(callback));
        }
    }

    // 统计
    size_t idle_count() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return idle_sessions_.size();
    }

    size_t active_count() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return active_sessions_.size();
    }

    size_t total_connections() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return idle_sessions_.size() + active_sessions_.size();
    }

private:
    asio::io_context& ctx_;
    PoolConfig cfg_;
    mutable std::mutex mutex_;
    std::deque<std::shared_ptr<TcpSession>> idle_sessions_;
    std::set<std::shared_ptr<TcpSession>> active_sessions_;
    std::deque<std::function<void(PooledSession)>> pending_callbacks_;
    asio::steady_timer health_timer_;
    std::atomic<bool> running_;

    // 创建新连接到目标服务器
    void create_connection() {
        if (!running_) return;

        auto socket = std::make_shared<tcp::socket>(ctx_);
        auto& sock_ref = *socket;

        sock_ref.async_connect(
            tcp::endpoint(
                asio::ip::make_address(cfg_.host), cfg_.port),
            [this, socket = std::move(socket),
             self = shared_from_this()](
                std::error_code ec) mutable {
                if (ec) {
                    std::lock_guard<std::mutex> lock(mutex_);
                    notify_pending(ec);
                    return;
                }

                auto session = std::make_shared<TcpSession>(
                    ctx_, std::move(*socket),
                    SessionType::TCP_CLIENT);
                SessionCallbacks cbs;
                cbs.on_error = [this, self,
                                weak = std::weak_ptr<TcpSession>(
                                    session)](
                                    std::error_code) {
                    if (auto s = weak.lock()) {
                        remove_from_pool(s);
                    }
                };
                cbs.on_close = [this, self,
                                weak = std::weak_ptr<TcpSession>(
                                    session)]() {
                    if (auto s = weak.lock()) {
                        remove_from_pool(s);
                    }
                };
                session->set_callbacks(std::move(cbs));
                session->start();

                std::lock_guard<std::mutex> lock(mutex_);
                if (!pending_callbacks_.empty()) {
                    auto cb = std::move(pending_callbacks_.front());
                    pending_callbacks_.pop_front();
                    active_sessions_.insert(session);
                    asio::post(ctx_,
                        [cb = std::move(cb),
                         s = session,
                         p = self]() mutable {
                            cb(PooledSession(p, s));
                        });
                } else {
                    idle_sessions_.push_back(session);
                }
            });
    }

    // 归还连接到池
    void release(std::shared_ptr<TcpSession> session) {
        if (!session || !session->is_open()) return;

        std::lock_guard<std::mutex> lock(mutex_);
        active_sessions_.erase(session);

        // 优先分配给等待的请求
        if (!pending_callbacks_.empty()) {
            auto cb = std::move(pending_callbacks_.front());
            pending_callbacks_.pop_front();
            active_sessions_.insert(session);
            asio::post(ctx_,
                [cb = std::move(cb), s = session,
                 self = shared_from_this()]() mutable {
                    cb(PooledSession(self, s));
                });
        } else {
            idle_sessions_.push_back(session);
        }
    }

    // 从池中移除（连接断开时调用）
    void remove_from_pool(std::shared_ptr<TcpSession> session) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = std::find(idle_sessions_.begin(),
                            idle_sessions_.end(), session);
        if (it != idle_sessions_.end()) {
            idle_sessions_.erase(it);
        } else {
            active_sessions_.erase(session);
        }

        // 补充连接（低于最小值时）
        size_t total = idle_sessions_.size() + active_sessions_.size();
        if (total < cfg_.min_connections && running_) {
            asio::post(ctx_,
                [this, self = shared_from_this()]() {
                    create_connection();
                });
        }
    }

    void notify_pending(std::error_code ec) {
        if (pending_callbacks_.empty()) return;
        auto cb = std::move(pending_callbacks_.front());
        pending_callbacks_.pop_front();
        asio::post(ctx_,
            [cb = std::move(cb)]() {
                cb(PooledSession()); // 空会话表示失败
            });
    }

    // 健康检查：移除空闲超时的连接
    void start_health_check() {
        health_timer_.expires_after(
            std::chrono::seconds(cfg_.health_check_sec));
        health_timer_.async_wait(
            [this, self = shared_from_this()](
                std::error_code ec) {
                if (ec || !running_) return;

                std::lock_guard<std::mutex> lock(mutex_);
                auto now = std::chrono::steady_clock::now();
                size_t idle_count = idle_sessions_.size();

                while (!idle_sessions_.empty() &&
                       idle_sessions_.size() > cfg_.min_connections) {
                    auto& front = idle_sessions_.front();
                    auto elapsed =
                        std::chrono::duration_cast<
                            std::chrono::seconds>(
                            now - front->last_active()).count();
                    if (elapsed >= cfg_.idle_timeout_sec) {
                        front->close();
                        idle_sessions_.pop_front();
                    } else {
                        break;
                    }
                }

                start_health_check();
            });
    }
};

} // namespace net
```

> **关键设计**：
>
> - **RAII PooledSession**：析构时自动归还，避免忘记释放
> - **等待队列**：连接数达到上限时，请求排队等待
> - **自动补充**：连接断开后自动创建新连接，保持最小连接数
> - **空闲回收**：超过 `idle_timeout_sec` 的空闲连接自动关闭

## 五、TCP 服务器

构建一个支持会话管理、优雅关闭的完整 TCP 服务器。

### 5.1 TCP 服务器类

```cpp
// tcp_server.hpp
#pragma once
#include "tcp_session.hpp"
#include <boost/asio.hpp>
#include <memory>
#include <set>

namespace net {

using tcp = asio::ip::tcp;

struct TcpServerConfig {
    uint16_t port = 8080;
    std::string bind_address = "0.0.0.0";
    int heartbeat_sec = 30;         // 心跳检测间隔
    size_t max_sessions = 1024;     // 最大连接数
    bool reuse_address = true;
};

class TcpServer {
public:
    TcpServer(asio::io_context& ctx, TcpServerConfig cfg)
        : ctx_(ctx), cfg_(std::move(cfg)),
          acceptor_(ctx),
          stopped_(false) {}

    // 启动监听
    void start() {
        tcp::endpoint endpoint(
            asio::ip::make_address(cfg_.bind_address),
            cfg_.port);
        acceptor_.open(endpoint.protocol());

        if (cfg_.reuse_address) {
            acceptor_.set_option(
                tcp::acceptor::reuse_address(true));
        }

        acceptor_.bind(endpoint);
        acceptor_.listen(
            boost::asio::socket_base::max_listen_connections);

        stopped_ = false;
        do_accept();
        std::cout << "[TCP 服务器] 已启动于 "
                  << endpoint << "\n";
    }

    // 停止服务器（优雅关闭）
    void stop() {
        stopped_ = true;
        boost::system::error_code ec;
        acceptor_.close(ec);

        // 关闭所有会话
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& session : sessions_) {
            session->close();
        }
        sessions_.clear();
    }

    // 广播消息给所有会话
    void broadcast(const Message& msg) {
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& session : sessions_) {
            if (session->is_open()) {
                session->send_message(msg);
            }
        }
    }

    // 获取活跃连接数
    size_t session_count() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return sessions_.size();
    }

    // 设置会话回调（用于处理业务消息）
    void set_message_handler(
        std::function<void(std::shared_ptr<TcpSession>,
                           const Message&)> handler) {
        msg_handler_ = std::move(handler);
    }

private:
    asio::io_context& ctx_;
    TcpServerConfig cfg_;
    tcp::acceptor acceptor_;
    std::atomic<bool> stopped_;
    mutable std::mutex mutex_;
    std::set<std::shared_ptr<TcpSession>> sessions_;
    std::function<void(std::shared_ptr<TcpSession>,
                       const Message&)> msg_handler_;

    void do_accept() {
        if (stopped_) return;

        acceptor_.async_accept(
            [this](std::error_code ec, tcp::socket socket) {
                if (ec) {
                    if (ec != asio::error::operation_aborted) {
                        do_accept();
                    }
                    return;
                }

                // 检查是否超过最大连接数
                {
                    std::lock_guard<std::mutex> lock(mutex_);
                    if (sessions_.size() >= cfg_.max_sessions) {
                        std::cerr << "[TCP 服务器] 达到最大连接数，"
                                  << "拒绝新连接\n";
                        socket.close();
                        do_accept();
                        return;
                    }
                }

                auto session = std::make_shared<TcpSession>(
                    ctx_, std::move(socket));

                SessionCallbacks cbs;
                cbs.on_message = [this, session](const Message& msg) {
                    if (msg_handler_) {
                        msg_handler_(session, msg);
                    }
                };
                cbs.on_error = [this, session](std::error_code) {
                    remove_session(session);
                };
                cbs.on_close = [this, session]() {
                    remove_session(session);
                };
                session->set_callbacks(std::move(cbs));
                session->start();

                {
                    std::lock_guard<std::mutex> lock(mutex_);
                    sessions_.insert(session);
                }

                std::cout << "[TCP 服务器] 新连接 ("
                          << socket.remote_endpoint() << "), "
                          << "当前连接数: " << session_count() << "\n";

                do_accept();
            });
    }

    void remove_session(std::shared_ptr<TcpSession> session) {
        std::lock_guard<std::mutex> lock(mutex_);
        sessions_.erase(session);
    }
};

} // namespace net
```

### 5.2 TCP 回显服务器示例

```cpp
// examples/tcp_echo.cpp
#include "tcp_server.hpp"
#include <iostream>
#include <csignal>

net::TcpServer* g_server = nullptr;

void signal_handler(int) {
    if (g_server) {
        std::cout << "\n正在关闭服务器...\n";
        g_server->stop();
    }
}

int main() {
    net::asio::io_context ctx;

    net::TcpServerConfig cfg;
    cfg.port = 8080;
    cfg.max_sessions = 1024;

    net::TcpServer server(ctx, cfg);

    // 设置消息处理：回显
    server.set_message_handler(
        [&server](std::shared_ptr<net::TcpSession> session,
                  const net::Message& msg) {
            auto type = static_cast<net::MessageType>(
                msg.header.type);
            std::cout << "收到消息, 类型="
                      << static_cast<int>(type)
                      << ", 长度=" << msg.body_length() << "\n";

            if (type == net::MessageType::DATA) {
                // 回显
                session->send_message(msg);
            } else if (type == net::MessageType::CLOSE) {
                std::cout << "客户端请求关闭\n";
                session->close();
            }
        });

    g_server = &server;
    std::signal(SIGINT, signal_handler);
    std::signal(SIGTERM, signal_handler);

    server.start();
    ctx.run();

    std::cout << "服务器已退出\n";
    return 0;
}
```

## 六、UDP 服务器

UDP 与 TCP 不同：无连接、需要手动跟踪对端地址、消息边界天然保留。

### 6.1 UDP 会话封装

```cpp
// udp_session.hpp
#pragma once
#include "session.hpp"
#include <boost/asio.hpp>
#include <unordered_map>

namespace net {

using udp = asio::ip::udp;

// UDP "会话"——实质是按 remote endpoint 分组的虚拟连接
class UdpSession : public Session {
public:
    UdpSession(asio::io_context& ctx, udp::socket& socket,
               udp::endpoint remote)
        : Session(ctx, SessionType::UDP)
        , socket_(socket)
        , remote_(remote) {}

    udp::endpoint remote() const { return remote_; }

    void start() override {
        open_ = true;
        start_heartbeat(15); // UDP 会话超时更短
    }

    void close() override {
        if (!open_) return;
        open_ = false;
        heartbeat_timer_.cancel();
        if (cbs_.on_close) cbs_.on_close();
    }

    void send_message(const Message& msg) override {
        if (!open_) return;
        update_active();
        socket_.async_send_to(
            to_buffers(msg), remote_,
            [this, self = shared_from_this()](
                std::error_code ec, size_t) {
                if (ec && cbs_.on_error) {
                    cbs_.on_error(ec);
                }
            });
    }

private:
    udp::socket& socket_;
    udp::endpoint remote_;
};

} // namespace net
```

### 6.2 UDP 服务器

```cpp
// udp_server.hpp
#pragma once
#include "udp_session.hpp"
#include <boost/asio.hpp>
#include <unordered_map>

namespace net {

struct UdpServerConfig {
    uint16_t port = 9090;
    std::string bind_address = "0.0.0.0";
    int session_timeout_sec = 30;  // UDP 虚拟会话超时
};

class UdpServer {
public:
    UdpServer(asio::io_context& ctx, UdpServerConfig cfg)
        : ctx_(ctx), cfg_(std::move(cfg)),
          socket_(ctx),
          cleanup_timer_(ctx) {}

    void start() {
        udp::endpoint endpoint(
            asio::ip::make_address(cfg_.bind_address),
            cfg_.port);
        socket_.open(endpoint.protocol());
        socket_.bind(endpoint);

        do_receive();
        start_cleanup();
        std::cout << "[UDP 服务器] 已启动于 "
                  << endpoint << "\n";
    }

    void stop() {
        cleanup_timer_.cancel();
        boost::system::error_code ec;
        socket_.close(ec);
        {
            std::lock_guard<std::mutex> lock(mutex_);
            for (auto& [_, session] : sessions_) {
                session->close();
            }
            sessions_.clear();
        }
    }

    void set_message_handler(
        std::function<void(std::shared_ptr<UdpSession>,
                           const Message&)> handler) {
        msg_handler_ = std::move(handler);
    }

    // 发送消息到指定对端
    void send_to(const udp::endpoint& remote,
                 const Message& msg) {
        socket_.async_send_to(
            to_buffers(msg), remote,
            [](std::error_code, size_t) {});
    }

private:
    asio::io_context& ctx_;
    UdpServerConfig cfg_;
    udp::socket socket_;
    mutable std::mutex mutex_;
    std::unordered_map<
        std::string, std::shared_ptr<UdpSession>> sessions_;
    std::function<void(std::shared_ptr<UdpSession>,
                       const Message&)> msg_handler_;
    asio::steady_timer cleanup_timer_;
    std::array<char, 65536> recv_buffer_;
    udp::endpoint remote_;

    void do_receive() {
        socket_.async_receive_from(
            asio::buffer(recv_buffer_), remote_,
            [this](std::error_code ec, size_t len) {
                if (ec) {
                    if (ec != asio::error::operation_aborted) {
                        do_receive();
                    }
                    return;
                }

                // 解析消息
                if (len >= sizeof(MessageHeader)) {
                    MessageHeader hdr;
                    std::memcpy(&hdr, recv_buffer_.data(),
                                sizeof(MessageHeader));
                    Message msg;
                    msg.header = hdr;
                    msg.body.assign(
                        recv_buffer_.data() + sizeof(MessageHeader),
                        recv_buffer_.data() + len);

                    auto session = get_or_create_session(remote_);

                    // 自动回复心跳
                    if (msg.header.type ==
                        static_cast<uint8_t>(MessageType::HEARTBEAT)) {
                        session->send_message(
                            Message(MessageType::HEARTBEAT));
                    } else if (msg_handler_) {
                        msg_handler_(session, msg);
                    }
                }

                do_receive();
            });
    }

    std::shared_ptr<UdpSession> get_or_create_session(
        const udp::endpoint& remote) {
        auto key = remote.address().to_string()
                   + ":" + std::to_string(remote.port());

        std::lock_guard<std::mutex> lock(mutex_);
        auto it = sessions_.find(key);
        if (it != sessions_.end()) {
            it->second->update_active();
            return it->second;
        }

        auto session = std::make_shared<UdpSession>(
            ctx_, socket_, remote);
        SessionCallbacks cbs;
        cbs.on_error = [this, key](std::error_code) {
            remove_session(key);
        };
        cbs.on_close = [this, key]() {
            remove_session(key);
        };
        session->set_callbacks(std::move(cbs));
        session->start();

        sessions_[key] = session;
        return session;
    }

    void remove_session(const std::string& key) {
        std::lock_guard<std::mutex> lock(mutex_);
        sessions_.erase(key);
    }

    // 定期清理超时的 UDP 虚拟会话
    void start_cleanup() {
        cleanup_timer_.expires_after(std::chrono::seconds(10));
        cleanup_timer_.async_wait(
            [this](std::error_code ec) {
                if (ec) return;
                auto now = std::chrono::steady_clock::now();

                std::lock_guard<std::mutex> lock(mutex_);
                for (auto it = sessions_.begin();
                     it != sessions_.end();) {
                    auto elapsed = std::chrono::duration_cast<
                        std::chrono::seconds>(
                        now - it->second->last_active()).count();
                    if (elapsed >= cfg_.session_timeout_sec) {
                        it->second->close();
                        it = sessions_.erase(it);
                    } else {
                        ++it;
                    }
                }

                start_cleanup();
            });
    }
};

} // namespace net
```

### 6.3 UDP 回显服务器示例

```cpp
// examples/udp_echo.cpp
#include "udp_server.hpp"
#include <iostream>
#include <csignal>

net::UdpServer* g_server = nullptr;

void signal_handler(int) {
    if (g_server) {
        std::cout << "\n正在关闭 UDP 服务器...\n";
        g_server->stop();
    }
}

int main() {
    net::asio::io_context ctx;

    net::UdpServerConfig cfg;
    cfg.port = 9090;
    cfg.session_timeout_sec = 30;

    net::UdpServer server(ctx, cfg);

    server.set_message_handler(
        [](std::shared_ptr<net::UdpSession> session,
           const net::Message& msg) {
            auto type = static_cast<net::MessageType>(
                msg.header.type);
            std::cout << "UDP 收到来自 "
                      << session->remote()
                      << ", 类型=" << static_cast<int>(type)
                      << ", 长度=" << msg.body_length() << "\n";

            if (type == net::MessageType::DATA) {
                // 回显
                session->send_message(msg);
            }
        });

    g_server = &server;
    std::signal(SIGINT, signal_handler);
    std::signal(SIGTERM, signal_handler);

    server.start();
    ctx.run();

    std::cout << "UDP 服务器已退出\n";
    return 0;
}
```

## 七、连接池使用示例

```cpp
// examples/pool_client.cpp
#include "connection_pool.hpp"
#include <iostream>
#include <chrono>
#include <thread>

int main() {
    net::asio::io_context ctx;

    net::PoolConfig cfg;
    cfg.host = "127.0.0.1";
    cfg.port = 8080;
    cfg.min_connections = 4;
    cfg.max_connections = 32;
    cfg.idle_timeout_sec = 60;

    auto pool = std::make_shared<net::ConnectionPool>(ctx, cfg);
    pool->start();

    // 在工作线程中运行 io_context
    std::thread worker([&ctx] { ctx.run(); });

    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    // 发送 10 条消息
    for (int i = 0; i < 10; ++i) {
        pool->acquire(
            [i](net::ConnectionPool::PooledSession session) {
                if (!session) {
                    std::cerr << "获取连接失败\n";
                    return;
                }
                std::string msg =
                    "Hello #" + std::to_string(i);
                net::Message m(net::MessageType::DATA,
                               msg.data(), msg.size());
                session->send_message(m);
                std::cout << "已发送: " << msg << "\n";
                // PooledSession 析构时自动归还
            });
    }

    std::this_thread::sleep_for(std::chrono::seconds(3));
    pool->stop();

    ctx.stop();
    worker.join();
    return 0;
}
```

## 八、CMake 构建

```cmake
cmake_minimum_required(VERSION 3.22)
project(asio_network VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Boost 1.83 REQUIRED COMPONENTS asio)

# 头文件库
add_library(asio_network INTERFACE)
target_include_directories(asio_network INTERFACE include)
target_link_libraries(asio_network INTERFACE Boost::asio)

# 示例
add_executable(tcp_echo examples/tcp_echo.cpp)
target_link_libraries(tcp_echo asio_network)

add_executable(udp_echo examples/udp_echo.cpp)
target_link_libraries(udp_echo asio_network)

add_executable(pool_client examples/pool_client.cpp)
target_link_libraries(pool_client asio_network)
```

## 九、性能与生产建议

### 9.1 线程模型

| 方案 | 适用场景 | 说明 |
|:----|:--------|:----|
| 单线程 `run()` | 简单服务 | 无锁，延迟最低 |
| `run()` 多线程 (n 个线程) | CPU 密集型 | 吞吐量接近 n 倍 |
| 每个 io_context 一个线程 | 多核隔离 | 避免跨核同步 |

### 9.2 优化策略

1. **Buffer 复用**：使用 `asio::dynamic_buffer` 或自定义内存池
2. **批量 flush**：高频小包时，合并写入减少系统调用
3. **读写分离**：大流量场景使用 `spawn` 分离读写协程
4. **零拷贝**：利用 `asio::buffer` 避免不必要的内存拷贝

### 9.3 注意事项

- **Always check `error_code`**：忽略错误会导致静默失败
- **避免在 handler 中抛异常**：未捕获异常会调用 `std::terminate`
- **使用 `shared_from_this` 延长生命周期**：确保异步操作完成前对象不被销毁
- **UDP 数据包大小限制**：IPv4 下建议不超过 1472 字节（避免 IP 分片）
- **设置 `SO_RCVBUF` / `SO_SNDBUF`**：大流量场景下增大内核缓冲区

## 十、总结

本文构建了一套完整的 Boost.Asio 网络框架：

| 组件 | 文件 | 核心功能 |
|:----|:----|:--------|
| 协议 | `protocol.hpp` | 长度前缀消息格式 |
| 会话层 | `session.hpp` / `tcp_session.hpp` | 连接生命周期、心跳、粘包处理 |
| 连接池 | `connection_pool.hpp` | 连接复用、自动扩容、空闲回收 |
| TCP 服务器 | `tcp_server.hpp` | 异步 Accept、会话管理、广播 |
| UDP 服务器 | `udp_server.hpp` | 虚拟会话、对端追踪、超时清理 |

这套设计适用于大多数 C++ 后端服务场景，在此基础之上可以叠加自有的业务协议、序列化层和加密层。
