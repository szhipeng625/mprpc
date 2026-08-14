# mprpc 项目详细介绍

## 1. 项目简介

**mprpc** 是一个基于 C++ 的轻量级分布式 RPC（Remote Procedure Call，远程过程调用）网络通信框架。该项目来源于知名 C++ 分布式系统实践课程，旨在帮助开发者深入理解从单机服务到集群服务、再到分布式架构的演进过程。

mprpc 的核心目标是**让开发者能够像调用本地函数一样调用远程服务**，完全屏蔽底层网络通信、数据序列化、服务发现等复杂细节，从而专注于业务逻辑的开发。

---

## 2. 核心目标

- **透明化远程调用**：提供与本地函数调用一致的体验。
- **高性能网络通信**：基于 Reactor 模型，支持高并发场景。
- **服务自动注册与发现**：通过 ZooKeeper 实现服务的动态上下线感知。
- **跨语言数据交换**：使用 Protobuf 定义服务接口与数据结构，保证异构系统间的兼容性。
- **模块化与可扩展**：各核心组件解耦，便于替换或扩展。

---

## 3. 核心技术栈

| 技术组件 | 作用 |
|---------|------|
| **muduo 网络库** | 高性能 C++ 网络库，基于 Reactor 模式，提供非阻塞 IO 与事件循环，负责底层 TCP 通信 |
| **Protobuf** | Google 出品的高效序列化协议，用于定义服务接口、消息结构，以及数据的序列化与反序列化 |
| **ZooKeeper** | 分布式协调服务，作为服务注册中心，提供服务注册与服务发现能力 |
| **CMake** | 跨平台构建工具，管理项目编译流程 |
| **异步日志** | 自研或集成的日志模块，支持异步写入，避免阻塞业务线程 |

---

## 4. 核心架构与工作流程

### 4.1 核心组件

- **RpcProvider（服务提供者）**  
  运行在服务端，负责：
  - 将本地注册的 RPC 服务发布到网络上。
  - 启动 muduo TCP 服务器，监听客户端请求。
  - 接收请求、反序列化、调用本地服务方法、序列化响应并返回。

- **RpcChannel（服务消费者/通道）**  
  运行在客户端，作为 Protobuf Service 的 `RpcChannel` 实现，负责：
  - 向 ZooKeeper 查询目标服务地址。
  - 建立与服务器的 TCP 连接。
  - 序列化调用请求并发送。
  - 接收响应并反序列化返回给调用方。

- **ZkClient（ZooKeeper 客户端）**  
  封装 ZooKeeper C API，提供服务注册与发现接口：
  - `Create`：创建临时节点，注册服务。
  - `GetData`：获取服务节点数据（IP + Port）。
  - 处理 ZooKeeper 会话事件（如连接建立、超时）。

- **RpcController（控制器）**  
  用于在 RPC 调用过程中传递控制信息，如：
  - 错误码（`Failed`、`ErrorCode`）
  - 错误文本（`ErrorText`）
  - 是否取消调用（`IsCanceled`）
  - 调用方可通过控制器判断调用是否成功。

- **RpcConfig（配置模块）**  
  负责读取和解析配置文件（如 `test.conf`），提供：
  - RPC 服务器 IP 和端口。
  - ZooKeeper 地址。
  - 日志相关配置等。

### 4.2 服务注册与发现机制

mprpc 使用 **ZooKeeper 临时节点** 进行服务注册：

- **服务提供者** 启动时，在 ZooKeeper 上创建形如 `/service_name/method_name` 的临时节点，节点数据为 `ip:port`。
- 临时节点的特性：当服务提供者与 ZooKeeper 断开连接（如进程崩溃）时，节点自动删除，从而实现服务的自动下线。
- **服务消费者** 调用服务前，根据服务名和方法名拼接路径，从 ZooKeeper 获取对应节点的数据，解析出目标服务器的 `ip:port`。

+-------------+ +-------------+ +-------------+
| Client | | ZooKeeper | | Server |
| (RpcChannel)| | (Registry) | |(RpcProvider)|
+------+------+ +------+------+ +------+------+
| | |
| 1. 查询服务地址 | |
|------------------->| |
| 2. 返回 ip:port | |
|<-------------------| |
| | |
| 3. 建立 TCP 连接 | |
|----------------------------------------->|
| 4. 发送序列化请求 | |
|----------------------------------------->|
| | 5. 反序列化并调用 |
| | 本地服务方法 |
| | 6. 序列化响应 |
| 7. 返回序列化响应 | |
|<-----------------------------------------|
| 8. 反序列化结果 | |
---
#### 详细步骤说明：

1. **服务注册**  
   服务端启动时，`RpcProvider` 读取配置，连接 ZooKeeper，并将本地注册的所有服务方法以临时节点形式写入 ZooKeeper。

2. **服务发现**  
   客户端调用 `Stub.method()` 时，`RpcChannel` 根据服务名和方法名向 ZooKeeper 查询节点数据，获取目标 `ip:port`。

3. **建立连接**  
   `RpcChannel` 使用获取的地址创建 TCP 连接（muduo TcpClient），若连接已存在则复用。

4. **发送请求**  
   客户端将调用信息（服务名、方法名、参数）打包成 Protobuf 消息，序列化后通过 TCP 发送。

5. **服务端处理**  
   `RpcProvider` 的 `onMessage` 回调接收数据，反序列化得到请求结构，根据服务名和方法名查找本地注册的服务对象及方法描述，动态调用对应方法。

6. **返回响应**  
   服务端将调用结果序列化后发送回客户端。

7. **客户端接收**  
   `RpcChannel` 收到响应数据，反序列化填充到响应消息中，返回给调用方，完成一次 RPC。

---

## 5. 项目目录结构
mprpc/
├── bin/ # 存放编译后的可执行文件
├── build/ # CMake 构建产生的临时文件
├── example/ # 示例代码
│ ├── user.proto # Protobuf 服务定义
│ ├── callee.cpp # 服务端示例（服务提供者）
│ └── caller.cpp # 客户端示例（服务消费者）
├── lib/ # 编译生成的库文件（如 libmprpc.a）
├── src/ # 框架核心源代码
│ ├── rpcprovider.cc # 服务端核心实现
│ ├── rpcchannel.cc # 客户端核心实现
│ ├── rpcconfig.cc # 配置加载实现
│ ├── zkclient.cc # ZooKeeper 客户端实现
│ ├── rpccontroller.cc # 控制器实现
│ ├── logger.cc # 异步日志实现
│ └── ... # 其他辅助文件
├── test/ # 测试代码
├── CMakeLists.txt # 顶层 CMake 构建文件
├── autobuild.sh # 一键编译脚本
└── README.md # 项目说明文档
---

## 6. 核心源码模块分析

### 6.1 RpcProvider（服务端核心）

**主要职责：**

- 继承 `muduo::net::TcpServer` 的回调接口，处理连接与消息事件。
- 维护服务注册表：`std::unordered_map<std::string, ServiceInfo>`，其中 `ServiceInfo` 包含服务对象指针和方法的 `MethodDescriptor`。
- 提供 `NotifyService(google::protobuf::Service *service)` 接口：将本地服务对象注册到表中，并生成服务描述信息。
- 提供 `Run()` 接口：读取配置、连接 ZooKeeper、注册服务节点、启动 muduo 事件循环。
- 在 `onMessage` 中解析请求头，根据服务名和方法名查找对应方法，通过 Protobuf 反射机制动态调用。

**关键逻辑：**

```cpp
// 注册服务
void RpcProvider::NotifyService(google::protobuf::Service *service) {
    ServiceInfo service_info;
    service_info.m_service = service;
    // 获取服务描述
    const google::protobuf::ServiceDescriptor *sd = service->GetDescriptor();
    for (int i = 0; i < sd->method_count(); ++i) {
        service_info.m_methodMap[sd->method(i)->name()] = sd->method(i);
    }
    m_serviceMap[sd->name()] = service_info;
}

// 处理请求
void RpcProvider::OnMessage(const TcpConnectionPtr& conn, Buffer* buffer, Timestamp) {
    // 解析头部，得到 service_name, method_name, args_size
    // 查找服务与方法
    // 生成请求/响应对象
    // 绑定回调，调用 service->CallMethod(method, nullptr, request, response, done)
    // 在 done 回调中序列化响应并发送回客户端
}
| | |
### 4.3 一次完整 RPC 调用流程
