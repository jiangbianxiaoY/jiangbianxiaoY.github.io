---
title: Boost.Asio 1.83 使用指南
date: 2026-06-16 12:00:00
updated: 2026-06-16 12:00:00
categories:
  - C++
tags:
  - Boost
  - Asio
  - 网络编程
  - C++
---

## 一、Boost.Asio 简介

Boost.Asio 是一个跨平台的 C++ 异步 I/O 库，提供了网络编程、文件 I/O、定时器等功能。Boost 1.83 于 2023 年发布，带来了 C++20 协程支持、IOCP 改进等新特性。

### 1.1 核心概念

| 概念 | 说明 | 头文件 |
|:----|:----|:------|
| io_context | 事件循环核心 | `boost/asio.hpp` |
| executor | 执行器（C++20 协程） | `boost/asio/executor.hpp` |
| timer | 定时器 | `boost/asio/steady_timer.hpp` |
| socket | 套接字 | `boost/asio/ip/tcp.hpp` |
| strand | 串行化执行 | `boost/asio/strand.hpp` |
| coroutine | C++20 协程支持 | `boost/asio/co_spawn.hpp` |

### 1.2 编译要求

```cmake
cmake_minimum_required(VERSION 3.20)
project(asio_example)

set(CMAKE_CXX_STANDARD 20)
find_package(Boost 1.83 REQUIRED COMPONENTS asio)

add_executable(demo main.cpp)
target_link_libraries(demo Boost::asio)
```

> **注意**：Boost.Asio 是 header-only 库，但依赖 `boost_system` 和 `boost_date_time`。从 1.83 开始，大多数组件无需链接额外库。

## 二、io_context 与事件循环

`io_context` 是所有异步操作的核心调度器。

### 2.1 基本用法

```cpp
#include <boost/asio.hpp>
#include <iostream>

namespace asio = boost::asio;

int main() {
    asio::io_context ctx;

    // 投递任务
    asio::post(ctx, [] {
        std::cout << "Hello from io_context!\n";
    });

    // 运行事件循环
    ctx.run();
    return 0;
}
```

### 2.2 run() 与 poll()

| 方法 | 行为 | 阻塞? |
|:----|:----|:-----|
| `run()` | 直到所有工作完成才返回 | 是 |
| `poll()` | 执行就绪 handlers 后立即返回 | 否 |
| `run_one()` | 执行一个 handler 后返回 | 是 |
| `poll_one()` | 执行一个就绪 handler 后返回 | 否 |

```cpp
asio::io_context ctx;

// 使用 work guard 阻止 run() 退出
auto work = asio::require(ctx.get_executor(),
    asio::execution::outstanding_work.tracked);

std::thread t([&ctx] { ctx.run(); });

// 主线程做其他事情
std::this_thread::sleep_for(std::chrono::seconds(1));

// 释放 work guard，让 run() 退出
work = asio::prefer(asio::default_executor(),
    asio::execution::outstanding_work.untracked);
t.join();
```

## 三、定时器

### 3.1 同步定时器

```cpp
asio::io_context ctx;
asio::steady_timer timer(ctx, std::chrono::seconds(3));

std::cout << "等待 3 秒...\n";
timer.wait(); // 阻塞等待
std::cout << "时间到！\n";
```

### 3.2 异步定时器

```cpp
void on_timeout(asio::steady_timer& t, const std::error_code& ec) {
    if (!ec) {
        std::cout << "定时器触发\n";
        // 重新设置，实现周期性
        t.expires_at(t.expiry() + std::chrono::seconds(1));
        t.async_wait([&t](auto ec) { on_timeout(t, ec); });
    }
}

asio::io_context ctx;
asio::steady_timer timer(ctx, std::chrono::seconds(1));
timer.async_wait([&timer](auto ec) { on_timeout(timer, ec); });
ctx.run();
```

### 3.3 C++20 协程定时器

```cpp
#include <boost/asio/experimental/awaitable_operators.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>

using namespace boost::asio::experimental::awaitable_operators;

asio::awaitable<void> timer_coroutine() {
    asio::steady_timer timer(co_await asio::this_coro::executor);

    for (int i = 0; i < 5; ++i) {
        timer.expires_after(std::chrono::seconds(1));
        co_await timer.async_wait(asio::use_awaitable);
        std::cout << "tick " << i + 1 << "\n";
    }
}

int main() {
    asio::io_context ctx;
    asio::co_spawn(ctx, timer_coroutine(), asio::detached);
    ctx.run();
    return 0;
}
```

## 四、TCP 网络编程

### 4.1 TCP 服务端

```cpp
#include <boost/asio.hpp>
#include <memory>

namespace asio = boost::asio;
using tcp = asio::ip::tcp;

class Session : public std::enable_shared_from_this<Session> {
    tcp::socket socket_;
    std::array<char, 1024> data_;

public:
    Session(tcp::socket socket) : socket_(std::move(socket)) {}

    void start() { do_read(); }

private:
    void do_read() {
        auto self = shared_from_this();
        socket_.async_read_some(asio::buffer(data_),
            [this, self](std::error_code ec, size_t len) {
                if (!ec) {
                    std::cout << "收到: "
                              << std::string(data_.data(), len) << "\n";
                    do_write(len);
                }
            });
    }

    void do_write(size_t len) {
        auto self = shared_from_this();
        asio::async_write(socket_, asio::buffer(data_, len),
            [this, self](std::error_code ec, size_t) {
                if (!ec) do_read();
            });
    }
};

class Server {
    tcp::acceptor acceptor_;

public:
    Server(asio::io_context& ctx, uint16_t port)
        : acceptor_(ctx, tcp::endpoint(tcp::v4(), port)) {
        do_accept();
    }

private:
    void do_accept() {
        acceptor_.async_accept(
            [this](std::error_code ec, tcp::socket socket) {
                if (!ec) {
                    std::make_shared<Session>(std::move(socket))->start();
                }
                do_accept();
            });
    }
};

int main() {
    asio::io_context ctx;
    Server server(ctx, 8080);
    std::cout << "服务器启动于 8080 端口\n";
    ctx.run();
    return 0;
}
```

### 4.2 TCP 客户端

```cpp
asio::awaitable<void> tcp_client() {
    auto executor = co_await asio::this_coro::executor;
    tcp::socket socket(executor);

    try {
        co_await socket.async_connect(
            tcp::endpoint(asio::ip::make_address("127.0.0.1"), 8080),
            asio::use_awaitable);

        std::string msg = "Hello, Asio!";
        co_await asio::async_write(socket,
            asio::buffer(msg), asio::use_awaitable);

        std::array<char, 1024> reply;
        auto len = co_await socket.async_read_some(
            asio::buffer(reply), asio::use_awaitable);
        std::cout << "回复: "
                  << std::string(reply.data(), len) << "\n";

    } catch (const std::exception& e) {
        std::cerr << "错误: " << e.what() << "\n";
    }
}

int main() {
    asio::io_context ctx;
    asio::co_spawn(ctx, tcp_client(), asio::detached);
    ctx.run();
}
```

## 五、UDP 编程

### 5.1 UDP 服务端

```cpp
using udp = asio::ip::udp;

class UdpServer {
    udp::socket socket_;
    udp::endpoint remote_;
    std::array<char, 65536> data_;

public:
    UdpServer(asio::io_context& ctx, uint16_t port)
        : socket_(ctx, udp::endpoint(udp::v4(), port)) {
        do_receive();
    }

private:
    void do_receive() {
        socket_.async_receive_from(
            asio::buffer(data_), remote_,
            [this](std::error_code ec, size_t len) {
                if (!ec) {
                    std::cout << "来自 " << remote_
                              << ": " << std::string(data_.data(), len) << "\n";
                    do_send(len);
                } else {
                    do_receive();
                }
            });
    }

    void do_send(size_t len) {
        socket_.async_send_to(
            asio::buffer(data_, len), remote_,
            [this](std::error_code, size_t) {
                do_receive();
            });
    }
};
```

## 六、高级特性

### 6.1 Strand 串行化

Strand 用于在多线程环境中保证 handlers 不并发执行：

```cpp
asio::io_context ctx;
auto strand = asio::make_strand(ctx);

int shared_data = 0;

// 这些任务会串行执行
asio::post(strand, [&] { ++shared_data; });
asio::post(strand, [&] { ++shared_data; });
asio::post(strand, [&] { ++shared_data; });

// 多线程运行
std::vector<std::thread> threads;
for (int i = 0; i < 4; ++i)
    threads.emplace_back([&ctx] { ctx.run(); });

for (auto& t : threads) t.join();
std::cout << "结果: " << shared_data << "\n"; // 3
```

### 6.2 协程超时

```cpp
asio::awaitable<void> with_timeout() {
    auto executor = co_await asio::this_coro::executor;
    asio::steady_timer timer(executor);
    timer.expires_after(std::chrono::seconds(5));

    tcp::socket socket(executor);

    try {
        // 异步连接，5 秒超时
        co_await (
            socket.async_connect(
                tcp::endpoint(asio::ip::make_address("192.168.1.1"), 80),
                asio::use_awaitable
            ) ||
            (timer.async_wait(asio::use_awaitable),
                throw std::runtime_error("连接超时"))
        );
    } catch (const std::exception& e) {
        std::cerr << "超时: " << e.what() << "\n";
    }
}
```

### 6.3 SSL/TLS

```cpp
#include <boost/asio/ssl.hpp>

namespace ssl = asio::ssl;

ssl::context ssl_ctx(ssl::context::tlsv12_client);
ssl_ctx.set_default_verify_paths();
ssl_ctx.set_verify_mode(ssl::verify_peer);

ssl::stream<tcp::socket> ssl_socket(ctx, ssl_ctx);

// 异步 SSL 握手
ssl_socket.async_handshake(ssl::stream_base::client,
    [](std::error_code ec) {
        if (!ec)
            std::cout << "SSL 握手成功\n";
    });
```

## 七、调试与错误处理

### 7.1 错误码

```cpp
void handle_connect(std::error_code ec) {
    switch (ec.value()) {
    case 0:
        std::cout << "连接成功\n";
        break;
    case asio::error::connection_refused:
        std::cerr << "连接被拒绝\n";
        break;
    case asio::error::connection_reset:
        std::cerr << "连接重置\n";
        break;
    case asio::error::timed_out:
        std::cerr << "连接超时\n";
        break;
    default:
        std::cerr << "未知错误: " << ec.message() << "\n";
        break;
    }
}
```

### 7.2 日志跟踪

```cpp
// 定义跟踪 handler
template <typename F>
auto traced(F f) {
    return [f = std::move(f)](auto... args) {
        std::cout << ">> handler 调用\n";
        f(args...);
        std::cout << "<< handler 返回\n";
    };
}

// 使用
socket.async_connect(endpoint, traced([](auto ec) {
    if (!ec) std::cout << "已连接\n";
}));
```

## 八、性能建议

1. **使用 `io_context::run()` 多线程**：`n` 个线程通常可获得接近 `n` 倍的吞吐量
2. **减少内存分配**：复用 `buffer`，使用 `bind_executor` 避免 lambda 捕获
3. **避免 blocking 操作**：不要在 handler 中执行阻塞 I/O
4. **选择合适的 buffer 策略**：

| 策略 | 适用场景 | 特点 |
|:----|:--------|:----|
| `asio::buffer(string)` | 小数据 | 简单直接 |
| `asio::dynamic_buffer` | 变长消息 | 自动扩容 |
| 内存池 | 高频小包 | 零分配 |

## 九、总结

Boost.Asio 1.83 在 C++20 协程支持、执行器模型方面做了重要改进。以下是最常用的 API：

| API | 用途 |
|:----|:----|
| `post()` / `defer()` / `dispatch()` | 任务投递 |
| `steady_timer::async_wait()` | 异步定时 |
| `ip::tcp::socket::async_read_some()` | TCP 读取 |
| `async_write()` | 异步写入 |
| `co_spawn()` | 启动协程 |
| `make_strand()` | 创建串行执行器 |

选择合适的异步模型（回调节点 vs. 协程）可以显著降低代码复杂度。
