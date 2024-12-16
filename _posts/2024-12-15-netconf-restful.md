---
title: diff netconf-restful
description: netconf、restful比较
author: dead4ead
date: 2024-12-15 12:00:00 +0800
categories: [Netconf, RESTful]
tags: [Netconf, RESTful]
#pin: true
math: true
mermaid: true
---

## NETCONF 

### 1. **NETCONF (Network Configuration Protocol)**

- **NETCONF** 是一种网络管理协议，主要用于配置、管理和监控网络设备（如路由器、交换机等）的配置。它通过远程调用接口（如SSH）与设备进行通信。
- NETCONF 定义了一些标准操作，如获取设备状态、修改配置、回滚等，支持基于 XML 的数据交换。

### 2. **XML (eXtensible Markup Language)**

- **XML** 是一种标记语言，用于表示结构化数据。在 NETCONF 中，XML 被广泛用于数据的表示和交换。
- 通过 NETCONF，所有的网络设备的配置信息和状态信息都是通过 XML 格式进行编码和解码的。例如，配置接口、路由、ACL 等，都是通过 XML 格式进行表达。

### 3. **YANG (Yet Another Next Generation)**

- **YANG** 是一种数据建模语言，用于定义网络设备的配置和状态模型。
- YANG 模型定义了网络设备的对象模型，并规定了如何操作这些对象的方式。它是对设备配置的逻辑描述，可以描述设备的接口、路由、VLAN 等。
- YANG 模型本质上提供了一种结构化的方式来描述 NETCONF 通信中使用的 XML 数据。NETCONF 客户端和服务器根据 YANG 模型约定的结构来交换数据。

### 4. **XSD (XML Schema Definition)**

- **XSD** 是 XML 架构定义语言，用来定义 XML 文档的结构和数据类型。
- 虽然 YANG 是更高层次的模型语言，但 XSD 可以用于对 XML 数据进行结构验证，确保 XML 数据符合预定格式。
- YANG 模型在某些情况下也可以转换为 XSD 来验证 XML 数据的合法性，尤其是在没有使用 YANG 的设备或系统中，XSD 提供了另一种手段来验证 XML 的正确性。



### 它们之间的关系

- **NETCONF** 使用 **XML** 来交换数据和配置。
- **YANG** 用于定义网络设备配置的模型（这些配置最终会以 XML 格式表示）。
- **XSD** 用于定义 XML 的数据格式和结构，确保 XML 文档的有效性。虽然 **XSD** 和 **YANG** 的功能类似，但 **YANG** 更适用于网络管理，尤其是在与 NETCONF 协作时。



### 总结

- **NETCONF** 是协议，使用 **XML** 来传输数据；
- **YANG** 是数据模型语言，用于定义 NETCONF 操作的数据结构；
- **XSD** 是用来验证 XML 格式的工具，确保 XML 数据符合预定规范。



通常在 NETCONF 的场景中，**YANG** 是主要的建模语言，而 **XSD** 只是用来辅助验证 XML 数据结构是否有效。 



## RESTful 

**RESTful** 是一种基于 HTTP 协议的架构风格，用于设计网络服务，通常用于客户端和服务器之间的通信。RESTful 服务强调资源（Resource）的操作，而每个资源都可以通过唯一的 URI（Uniform Resource Identifier）访问。REST 采用简单的 HTTP 方法（如 GET、POST、PUT、DELETE）来操作资源。 

**核心特点**

- 使用 **HTTP** 协议，支持标准的 HTTP 方法（GET、POST、PUT、DELETE）。
- 数据格式通常是 **JSON** 或 **XML**。
- 无状态（stateless）：每次请求都包含足够的信息，服务器不需要记住客户端的状态。
- 资源（Resource）是 RESTful API 的核心，操作这些资源时通过 URI 来指定。

**使用场景**：适用于需要简单、易用的网络服务，尤其是在云计算、移动设备和 Web 应用程序中，RESTful API 提供了一种标准的、易于集成的方式。



## NETCONF 和 RESTful 的关系 

尽管 **NETCONF** 和 **RESTful** 协议有不同的目标和实现方式，但在某些情况下，它们之间有交集，尤其是在现代网络管理架构中，许多网络设备支持通过 RESTful API 或 NETCONF 来进行配置和管理。

### 共同点

- **网络管理**：两者都可用于网络设备的配置和管理。NETCONF 传统上更多用于企业级的设备管理，而 RESTful API 更常见于现代 Web 应用、云平台和 SDN（软件定义网络）中。
- **数据传输格式**：尽管 NETCONF 使用 **XML** 格式，许多设备在实现 RESTful API 时也支持 **JSON** 和 **XML** 格式的消息交换。

### 主要区别

- **协议和架构**：NETCONF 是一个专门设计用于网络设备配置的协议，具有更强的结构化和标准化功能（例如 **YANG** 数据模型），并且通过 XML 和 RPC 方式操作。而 **RESTful** 是一种更通用的 Web 服务设计风格，通常使用 HTTP、JSON 和 REST API 来操作资源，适合互联网时代的轻量级、易于使用的接口。
- **操作复杂度**：NETCONF 支持事务性操作（如提交和回滚配置），它适合处理更复杂的配置管理任务。而 RESTful API 更加简单直观，适合执行基本的 CRUD（创建、读取、更新、删除）操作。
- **会话管理**：NETCONF 支持持久连接，可以在同一个会话中执行多个命令，而 RESTful API 通常是无状态的，每个请求都是独立的。



### **现代网络管理中的应用**

随着 SDN 和自动化网络管理的发展，**RESTful API** 和 **NETCONF** 在网络设备管理中都有广泛应用。现在，许多现代网络设备既支持 **NETCONF** 协议，又提供 **RESTful API**，这样可以根据实际需求选择合适的协议。

#### 互补性

- **NETCONF** 更适合需要复杂、事务性操作的场景，如设备的批量配置、回滚操作等。
- **RESTful** 更适合轻量级、快速集成的场景，尤其是云平台、Web 应用和移动设备等现代网络应用。

#### 例子

- **NETCONF** 适用于传统网络设备的配置和管理，尤其是在大规模数据中心、服务提供商网络和高要求的企业网络中。
- **RESTful API** 适用于云平台管理、SDN 控制器、网络可编程性等场景。许多 SDN 解决方案（如 OpenDaylight、ONOS）和云管理平台（如 OpenStack）提供 RESTful API 用于网络设备的管理。

### 报文格式

#### NETCONF 报文 

**NETCONF** 是一个基于 XML 和 RPC 的协议，通常使用 **SSH** 或 **TLS** 作为传输协议，并在上面进行 **NETCONF** 会话。**NETCONF** 的数据传输通常通过 TCP/IP 连接进行，但这部分在报文中是隐式的，通常由 **NETCONF** 服务器（设备）和客户端之间的连接来保证。 

示例：NETCONF 报文（带有 IP 和 TCP 头）

假设你通过 **NETCONF** 协议发送一个 **edit-config** 请求来修改设备的接口配置，下面是加入网络层和传输层头信息后的示例。



**网络层（IP 头）和传输层（TCP 头）：**

```
-------------------------------------------
|        IP 头部 (IPv4)                   |
|-----------------------------------------|
| Source IP: 192.168.1.10                 |
| Destination IP: 192.168.1.20            |
| Protocol: TCP (6)                       |
-------------------------------------------
|        TCP 头部                         |
|-----------------------------------------|
| Source Port: 830 (NETCONF 默认端口)     |
| Destination Port: 830                  |
| Sequence Number: 12345                 |
| Acknowledgment Number: 67890            |
| Flags: SYN, ACK                         |
| ...                                     |
-------------------------------------------
```

**应用层（NETCONF 报文）：**

```
-------------------------------------------
|             NETCONF RPC 报文            |
|-----------------------------------------|
| <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"> |
|     <edit-config>                       |
|         <target>                        |
|             <running/>                 |
|         </target>                       |
|         <config>                        |
|             <interfaces xmlns="urn:example:interface-config"> |
|                 <interface>            |
|                     <name>Ethernet0</name> |
|                     <ip-address>192.168.1.100</ip-address> |
|                     <subnet-mask>255.255.255.0</subnet-mask> |
|                     <status>up</status> |
|                 </interface>           |
|             </interfaces>              |
|         </config>                       |
|     </edit-config>                      |
| </rpc>                                  |
-------------------------------------------

```

**解析：**

- **IP 头部**：标识源 IP 和目标 IP 地址，表明数据包的传输路径。
- **TCP 头部**：标识传输协议（如源端口号为 830，表示 **NETCONF** 默认端口），并包含 TCP 连接的状态（如序列号、确认号等）。
- **NETCONF 报文**：实际的 XML 请求体，表示客户端请求修改设备配置。



#### RESTful 报文 

**RESTful API** 通过 **HTTP** 协议进行通信，HTTP 协议在应用层上运行。**RESTful** 使用标准的 HTTP 方法（如 GET、POST、PUT、DELETE）来操作资源，通常使用 **JSON** 或 **XML** 格式进行数据交换。传输层使用 **TCP**，并在其上运行 **HTTP**。

**示例：RESTful API 报文（带有 IP 和 HTTP 头）**

假设我们要使用 **RESTful** API 查询设备的接口信息，以下是带有 **IP 头**、**TCP 头** 和 **HTTP 头** 的完整报文。



**网络层（IP 头）和传输层（TCP 头）：**

```
-------------------------------------------
|        IP 头部 (IPv4)                   |
|-----------------------------------------|
| Source IP: 192.168.1.10                 |
| Destination IP: 192.168.1.20            |
| Protocol: TCP (6)                       |
-------------------------------------------
|        TCP 头部                         |
|-----------------------------------------|
| Source Port: 12345                      |
| Destination Port: 80                    |
| Sequence Number: 54321                  |
| Acknowledgment Number: 98765            |
| Flags: SYN, ACK                         |
| ...                                     |
-------------------------------------------
```

**应用层（HTTP 报文）：**

```
-------------------------------------------
|             HTTP 请求头                |
|-----------------------------------------|
| GET /api/v1/interfaces/Ethernet0 HTTP/1.1|
| Host: device.example.com               |
| Authorization: Bearer your_access_token|
| Content-Type: application/json         |
-------------------------------------------
|             HTTP 响应报文               |
|-----------------------------------------|
| HTTP/1.1 200 OK                        |
| Content-Type: application/json         |
| Date: Tue, 06 Dec 2024 12:00:00 GMT    |
-------------------------------------------
|             JSON 响应体                 |
|-----------------------------------------|
| {                                       |
|   "interface": {                        |
|     "name": "Ethernet0",                |
|     "ip-address": "192.168.1.100",      |
|     "subnet-mask": "255.255.255.0",     |
|     "status": "up"                      |
|   }                                     |
| }                                       |
-------------------------------------------
```

**解析**：

- **IP 头部**：标识源 IP 和目标 IP 地址，表明数据包的传输路径。
- **TCP 头部**：传输层信息，如源端口号和目标端口号（通常 HTTP 使用 80 端口），并包含 TCP 连接的状态。
- **HTTP 请求头**：请求方法（GET）、请求路径（`/api/v1/interfaces/Ethernet0`）、请求头（如授权信息、内容类型等）。
- **HTTP 响应头**：响应状态码（200 OK）、内容类型和其他元数据。
- **JSON 响应体**：返回的实际数据，表示接口的配置信息。



#### 对比：NETCONF 和 RESTful 报文 

| 特性         | **NETCONF**                                      | **RESTful**                                                |
| ------------ | ------------------------------------------------ | ---------------------------------------------------------- |
| **协议**     | 基于 **XML** 和 **RPC**                          | 基于 **HTTP** 协议                                         |
| **数据格式** | **XML**（通常与 YANG 模型结合）                  | **JSON**（通常是最常用格式，也支持 XML）                   |
| **请求方法** | **edit-config**, **get-config** 等基于 RPC 调用  | **GET**, **POST**, **PUT**, **DELETE** 等标准 HTTP 方法    |
| **传输协议** | **TCP**（使用 SSH 或 TLS 封装 NETCONF）          | **TCP**（通过 HTTP/HTTPS）                                 |
| **会话管理** | 支持持久连接，基于会话管理（会话标识、持久连接） | 无状态，每个请求是独立的                                   |
| **头部信息** | 包含 **XML** 头部（如 `xmlns`）                  | 包含 **HTTP** 请求头（如 `Authorization`, `Content-Type`） |

