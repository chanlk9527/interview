# 20 - IoT 物联网平台

---

## 一、IoT 平台基础概念

1. IoT 平台的核心功能模块有哪些？设备接入、设备管理、数据采集、规则引擎、应用使能？
2. IoT 平台的整体架构一般分为哪几层？感知层、网络层、平台层、应用层？
3. 什么是设备影子（Device Shadow / Device Twin）？解决了什么问题？
4. 什么是物模型（Thing Model）？属性、服务、事件分别是什么？
5. IoT 平台如何实现设备的统一管理？设备注册、认证、分组、OTA 升级？

## 二、MQTT 协议

1. MQTT 协议的核心特点？为什么它是 IoT 领域最主流的协议？
2. MQTT 的发布/订阅模型？Broker 的角色？与传统请求/响应模型的区别？
3. MQTT 的 QoS 等级？QoS 0、QoS 1、QoS 2 的区别和适用场景？
4. MQTT 的 QoS 1 如何保证消息至少到达一次？PUBACK 机制？
5. MQTT 的 QoS 2 如何保证消息恰好到达一次？四次握手流程？
6. MQTT 的 Topic 设计规范？通配符 `+` 和 `#` 的区别？
7. MQTT 的保留消息（Retained Message）是什么？使用场景？
8. MQTT 的遗嘱消息（Will Message / LWT）是什么？如何实现设备离线通知？
9. MQTT 的 Keep Alive 机制？PINGREQ / PINGRESP？如何检测设备掉线？
10. MQTT 的 Clean Session 和 Persistent Session 的区别？MQTT 5.0 的 Session Expiry？
11. MQTT 5.0 相比 3.1.1 有哪些重要改进？共享订阅、用户属性、原因码？
12. MQTT 的共享订阅（Shared Subscription）是什么？如何实现消费端负载均衡？
13. MQTT Broker 的选型？EMQX vs Mosquitto vs HiveMQ vs VerneMQ？
14. EMQX 的集群架构？如何支撑百万级连接？
15. MQTT 的安全机制？TLS/SSL、用户名密码认证、Token 认证、证书双向认证？

## 三、CoAP 协议

1. CoAP 协议的特点？与 MQTT 的区别？适用场景？
2. CoAP 基于 UDP，如何保证可靠传输？CON / NON / ACK / RST 消息类型？
3. CoAP 的观察模式（Observe）？与 MQTT 订阅的区别？
4. CoAP 的资源发现机制？`/.well-known/core`？
5. 什么场景下选 CoAP 而不是 MQTT？

## 四、WebSocket 与 HTTP

### Q1: WebSocket 在 IoT 中的应用场景？与 MQTT over WebSocket 的关系？

**核心回答：**

WebSocket 在 IoT 中主要有两大应用场景：

**场景一：Web 管理端的实时数据推送**

IoT 平台通常有一个 Web 管理控制台，需要实时展示设备状态、传感器数据、告警信息等。传统 HTTP 轮询效率低、延迟高，WebSocket 提供了全双工的长连接通道，服务端可以主动推送数据到浏览器。

典型用途：
- 设备在线/离线状态实时变更
- 传感器数据实时曲线图
- 告警事件实时弹窗通知
- 设备指令下发后的执行结果回调

**场景二：作为 MQTT 的传输层（MQTT over WebSocket）**

浏览器原生不支持 TCP，无法直接连接 MQTT Broker。MQTT over WebSocket 就是把 MQTT 协议的数据帧封装在 WebSocket 帧里传输，让 Web 端也能使用 MQTT 的发布/订阅能力。

两者的关系：
- WebSocket 是**传输层协议**，解决的是"浏览器如何建立长连接"的问题
- MQTT 是**应用层协议**，解决的是"消息如何发布和订阅"的问题
- MQTT over WebSocket = MQTT 协议跑在 WebSocket 通道上，而不是跑在原生 TCP 上

```
传统设备:   Device  --TCP-->  MQTT Broker
Web 浏览器: Browser --WebSocket-->  MQTT Broker（Broker 的 WS 端口）
```

**追问：为什么不直接用 WebSocket 做消息推送，还要套一层 MQTT？**

因为 WebSocket 只提供了双向通信通道，不提供消息语义。你需要自己实现：
- 主题订阅/取消订阅
- 消息质量保证（QoS）
- 离线消息缓存
- 遗嘱消息

而 MQTT 已经把这些都标准化了。用 MQTT over WebSocket，Web 端和设备端可以共享同一套消息体系，不需要额外的协议转换层。

---

### Q2: WebSocket 的握手过程？从 HTTP 升级到 WebSocket？

**核心回答：**

WebSocket 连接的建立基于 HTTP 协议的 **Upgrade 机制**，整个过程只需要一次 HTTP 请求-响应：

**第一步：客户端发起升级请求**

```http
GET /ws HTTP/1.1
Host: iot-platform.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: mqtt
```

关键头部：
- `Upgrade: websocket` — 告诉服务端要升级协议
- `Connection: Upgrade` — 表示这是一个协议升级请求
- `Sec-WebSocket-Key` — 客户端生成的随机 Base64 字符串，用于握手校验
- `Sec-WebSocket-Version: 13` — WebSocket 协议版本（RFC 6455）
- `Sec-WebSocket-Protocol` — 子协议协商，IoT 场景常见值是 `mqtt`

**第二步：服务端返回 101 切换协议**

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: mqtt
```

- 状态码 `101` 表示协议切换成功
- `Sec-WebSocket-Accept` 的计算方式：将客户端的 `Sec-WebSocket-Key` 拼接一个固定的 GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，做 SHA-1 哈希后 Base64 编码。这个机制防止普通 HTTP 请求被误当作 WebSocket 连接

**第三步：协议切换完成，进入全双工通信**

握手完成后，底层 TCP 连接不断开，后续的数据传输不再是 HTTP 格式，而是 WebSocket 帧格式。双方可以随时互发消息。

```
客户端                          服务端
  |--- HTTP GET (Upgrade) ------->|
  |<-- HTTP 101 (Switching) ------|
  |                                |
  |<====== WebSocket 全双工 ======>|
  |   （二进制帧 / 文本帧）         |
```

**追问：为什么 WebSocket 要借助 HTTP 握手，而不是直接建立连接？**

两个原因：
1. **兼容性** — 复用 HTTP 的 80/443 端口，能穿透大部分防火墙和代理服务器。如果用自定义端口，很多企业网络环境会被拦截
2. **安全性** — 借助 HTTP 的认证机制（Cookie、Token），可以在握手阶段完成身份校验，不需要在 WebSocket 层再实现一套认证

---

### Q3: WebSocket 的心跳机制？Ping / Pong 帧？

**核心回答：**

WebSocket 协议（RFC 6455）内置了 **Ping/Pong 控制帧** 来实现心跳保活：

**机制原理：**

- **Ping 帧（opcode 0x9）**：任意一端都可以发送，通常由服务端定时发起
- **Pong 帧（opcode 0xA）**：收到 Ping 后必须尽快回复 Pong，Pong 的 payload 必须与 Ping 一致
- 如果一端在预期时间内没有收到 Pong 响应，就认为连接已断开，主动关闭连接

```
服务端                          客户端
  |--- Ping (payload: "hb") ----->|
  |<-- Pong (payload: "hb") ------|
  |                                |
  |  ... 30秒后 ...                |
  |--- Ping (payload: "hb") ----->|
  |<-- Pong (payload: "hb") ------|
  |                                |
  |  ... 30秒后 ...                |
  |--- Ping (payload: "hb") ----->|
  |         (超时无响应)            |
  |--- Close Frame --------------->|  ← 判定连接断开
```

**为什么需要心跳？**

1. **检测死连接** — TCP 本身的 KeepAlive 默认间隔太长（Linux 默认 2 小时），无法及时发现连接异常
2. **NAT 保活** — 很多 IoT 设备通过 NAT 网关上网，NAT 表项有超时时间（通常 5~10 分钟），心跳可以防止 NAT 映射被回收
3. **资源回收** — 服务端需要及时清理死连接，释放内存和文件描述符

**IoT 场景的实践建议：**

| 参数 | 建议值 | 说明 |
|------|--------|------|
| 心跳间隔 | 30~60 秒 | 太短浪费带宽，太长检测不及时 |
| 超时判定 | 2~3 次心跳未响应 | 避免因网络抖动误判 |
| 发起方 | 服务端 | 服务端统一管理，便于监控 |

**追问：应用层心跳 vs WebSocket 协议层心跳？**

实际项目中经常两层都做：
- **协议层**：WebSocket 的 Ping/Pong，由框架自动处理（如 Netty 的 `WebSocketServerProtocolHandler`）
- **应用层**：业务自定义的心跳消息（如 JSON `{"type":"heartbeat"}`），可以携带额外信息（设备状态、时间戳等）

在 MQTT over WebSocket 的场景下，MQTT 自身的 Keep Alive（PINGREQ/PINGRESP）就充当了应用层心跳，WebSocket 层的 Ping/Pong 作为底层保活的补充。

---

### Q4: WebSocket 与长轮询（Long Polling）、SSE（Server-Sent Events）的对比？

**核心回答：**

三者都是为了解决"服务端主动推送数据到客户端"的问题，但机制和适用场景差异很大：

| 维度 | Long Polling | SSE | WebSocket |
|------|-------------|-----|-----------|
| 通信方向 | 单向（模拟推送） | 单向（服务端→客户端） | 全双工 |
| 底层协议 | HTTP | HTTP | 独立协议（基于TCP） |
| 连接方式 | 反复建立HTTP请求 | 单个长连接 | 单个长连接 |
| 数据格式 | 任意 | 纯文本（text/event-stream） | 文本或二进制 |
| 浏览器支持 | 所有 | 现代浏览器（IE不支持） | 现代浏览器 |
| 服务端推送延迟 | 较高（取决于轮询间隔） | 低 | 极低 |
| 带宽开销 | 高（每次都有HTTP头） | 中 | 低（帧头只有2~14字节） |
| 断线重连 | 需自己实现 | 内置自动重连 | 需自己实现 |
| 适用场景 | 兼容性要求极高 | 通知、日志流、股票行情 | 实时交互、IoT、聊天 |

**Long Polling 的工作方式：**

客户端发起 HTTP 请求，服务端不立即响应，而是 hold 住连接直到有新数据或超时。客户端收到响应后立即发起下一次请求。本质上是用"不断发请求"来模拟推送，每次请求都带完整的 HTTP 头，开销大。

**SSE 的工作方式：**

客户端通过 `EventSource` API 建立连接，服务端通过 `text/event-stream` 格式持续推送事件。优点是内置自动重连和事件 ID 追踪，缺点是只能服务端→客户端单向推送，且只支持文本。

**WebSocket 的优势：**

- 全双工，客户端和服务端都能随时发消息
- 帧头极小（最小 2 字节），适合高频小数据包场景
- 支持二进制数据，适合传输传感器原始数据
- 一次握手后持续通信，没有反复建连的开销

**IoT 场景下的选择：**

- **设备端通信** → WebSocket（或直接 MQTT over TCP）
- **Web 控制台实时数据** → WebSocket 或 MQTT over WebSocket
- **简单的告警通知推送** → SSE 也够用，实现更简单
- **兼容老旧浏览器** → Long Polling 作为降级方案

---

### Q5: MQTT over WebSocket 的实现原理？为什么 Web 端需要通过 WebSocket 连接 MQTT？

**核心回答：**

**为什么需要 MQTT over WebSocket？**

MQTT 协议原生运行在 TCP 之上，但浏览器出于安全限制，不允许 JavaScript 直接创建 TCP 连接。浏览器能用的长连接协议只有 WebSocket。所以要让 Web 端使用 MQTT，就必须把 MQTT 的数据包封装在 WebSocket 帧里传输。

**实现原理：**

```
┌─────────────────────────────────────────────┐
│              Web 浏览器                       │
│  ┌─────────┐    ┌──────────────────────┐     │
│  │ 业务代码 │───>│ MQTT.js 客户端库      │     │
│  └─────────┘    │  - MQTT 协议编解码     │     │
│                 │  - 封装为 WebSocket 帧  │     │
│                 └──────────┬───────────┘     │
│                            │ WebSocket       │
└────────────────────────────┼─────────────────┘
                             │ ws://broker:8083
                             │ wss://broker:8084 (TLS)
┌────────────────────────────┼─────────────────┐
│           MQTT Broker      │                  │
│  ┌─────────────────────────┴──────────┐      │
│  │  WebSocket Listener (端口 8083/8084)│      │
│  │  - 解析 WebSocket 帧               │      │
│  │  - 提取 MQTT 数据包                │      │
│  │  - 交给 MQTT 协议引擎处理          │      │
│  └────────────────────────────────────┘      │
│  ┌────────────────────────────────────┐      │
│  │  TCP Listener (端口 1883/8883)      │      │
│  │  - 处理原生 TCP 设备连接            │      │
│  └────────────────────────────────────┘      │
└──────────────────────────────────────────────┘
```

本质上就是一层**传输层的封装/解封装**：

1. **发送方向**：MQTT 客户端库（如 MQTT.js）将 MQTT CONNECT、PUBLISH 等报文序列化为二进制，然后作为 WebSocket 二进制帧的 payload 发出
2. **接收方向**：Broker 的 WebSocket Listener 收到帧后，剥离 WebSocket 帧头，把 payload 交给 MQTT 协议引擎，后续处理和 TCP 连接完全一致

**关键实现细节：**

- MQTT Broker 需要同时监听 TCP 端口（1883）和 WebSocket 端口（8083），两种连接在 Broker 内部共享同一个消息路由引擎
- WebSocket 子协议协商时，`Sec-WebSocket-Protocol` 的值为 `mqtt`（MQTT 3.1.1）或 `mqttv5`（MQTT 5.0）
- Web 端常用的客户端库是 **MQTT.js**（JavaScript）或 **Paho**（Eclipse 出品）
- 生产环境必须用 `wss://`（WebSocket over TLS），不能用明文 `ws://`

**追问：MQTT over WebSocket 和原生 MQTT over TCP 有性能差异吗？**

有，但不大：
- WebSocket 帧头额外开销 2~14 字节，对于 IoT 小数据包来说比例不算小
- 多了一层帧的封装/解封装，CPU 开销略增
- 但对于 Web 端来说这是唯一选择，性能差异在可接受范围内
- 真正的设备端应该直接用 TCP 连接，不需要走 WebSocket

---

### Q6: HTTP 在 IoT 中的使用场景？设备配置下发、OTA 升级、REST API？

**核心回答：**

虽然 MQTT 是 IoT 的主流通信协议，但 HTTP 在 IoT 平台中仍然有不可替代的位置，主要用于**非实时、低频、大数据量**的场景：

**1. 设备配置下发与管理 API**

IoT 平台的管理后台通过 RESTful API 进行设备的 CRUD 操作：
- 设备注册 / 删除 / 查询
- 设备分组管理
- 设备配置模板下发
- 物模型的定义与更新

这些操作频率低、对实时性要求不高，用 HTTP 的请求-响应模型最自然。

**2. OTA 固件升级**

固件包通常几 MB 到几百 MB，不适合通过 MQTT 传输（MQTT 设计目标是小消息）。典型流程：
1. 平台通过 MQTT 通知设备"有新固件可用"，下发固件下载 URL
2. 设备通过 HTTP/HTTPS GET 请求下载固件包（支持断点续传）
3. 设备升级完成后通过 MQTT 上报升级结果

**3. 设备认证与 Token 获取**

设备首次接入时，通过 HTTPS 请求认证服务获取连接凭证（Token、证书等），然后再用凭证连接 MQTT Broker。这种"先 HTTP 认证，再 MQTT 通信"的模式很常见。

**4. 数据查询与历史数据导出**

- 查询设备历史数据、统计报表
- 导出 CSV / Excel 格式的数据
- 第三方系统集成（Webhook 回调）

**5. 边缘网关的本地 API**

边缘网关通常提供本地 HTTP API，供局域网内的设备或应用调用，用于本地配置、调试、数据查询等。

**总结：MQTT vs HTTP 的分工**

| 场景 | 协议选择 | 原因 |
|------|---------|------|
| 设备实时数据上报 | MQTT | 高频、小包、低延迟 |
| 设备指令下发 | MQTT | 需要推送能力 |
| 设备状态变更通知 | MQTT | 实时性要求高 |
| 管理后台 API | HTTP | 请求-响应模型，RESTful 风格 |
| OTA 固件下载 | HTTP | 大文件传输，支持断点续传 |
| 设备认证 | HTTPS | 安全性要求高，一次性操作 |
| 历史数据查询 | HTTP | 非实时，数据量大 |

---

### Q7: HTTP 长连接在 IoT 场景下的局限性？

**核心回答：**

HTTP 长连接（HTTP/1.1 Keep-Alive 或 HTTP/2 多路复用）虽然复用了 TCP 连接，但在 IoT 场景下有几个根本性的局限：

**1. 请求-响应模型的限制**

HTTP 本质是"客户端发起请求，服务端被动响应"。服务端无法主动向设备推送消息。IoT 场景中，平台需要随时向设备下发指令（开关灯、调温度、升级固件），HTTP 做不到真正的服务端推送。

虽然可以用长轮询模拟，但：
- 每次轮询都带完整 HTTP 头（几百字节到几 KB），对于资源受限的 IoT 设备来说开销太大
- 轮询间隔决定了推送延迟，无法做到真正的实时

**2. 头部开销大**

HTTP 每次请求都携带大量头部信息（Host、User-Agent、Cookie、Content-Type 等），对于 IoT 设备频繁上报的小数据包（可能只有几十字节的传感器数据），头部开销占比极高。

对比：
- HTTP 请求头：通常 200~800 字节
- MQTT PUBLISH 固定头：最小 2 字节
- WebSocket 帧头：最小 2 字节

**3. 不支持发布/订阅模式**

IoT 平台的核心通信模式是发布/订阅（一个设备发布数据，多个订阅者消费）。HTTP 天然不支持这种模式，需要在应用层自己实现消息分发逻辑，增加了架构复杂度。

**4. 连接管理效率低**

HTTP Keep-Alive 的连接有超时时间，空闲一段时间后会被关闭。对于 IoT 设备来说，可能几分钟才上报一次数据，连接频繁断开重建，浪费资源。而 MQTT 的长连接配合 Keep Alive 心跳，可以稳定维持数小时甚至数天。

**5. 对弱网环境不友好**

IoT 设备经常处于 2G/3G、Wi-Fi 信号弱、网络不稳定的环境。HTTP 没有内置的 QoS 机制，消息丢了就丢了。MQTT 的 QoS 1/2 可以保证消息在弱网下的可靠投递。

**6. 不支持离线消息**

设备离线期间，平台下发的指令无法通过 HTTP 缓存。MQTT Broker 可以为离线设备保存消息（Clean Session = false + QoS > 0），设备重新上线后自动接收。

**总结：**

HTTP 长连接解决的是"复用 TCP 连接减少握手开销"的问题，但没有改变 HTTP 的请求-响应本质。IoT 场景需要的是：全双工通信、服务端主动推送、极低的协议开销、消息可靠性保证、离线消息支持——这些恰好是 MQTT 和 WebSocket 的强项。所以 HTTP 在 IoT 中只适合做管理 API 和大文件传输，不适合做设备实时通信。

## 五、设备接入与协议适配

1. IoT 平台如何实现多协议统一接入？协议适配层的设计？
2. 如何设计一个可扩展的协议网关？
3. 设备认证的常见方案？一机一密、一型一密、X.509 证书？
4. 设备接入的安全性如何保障？TLS、DTLS、Token 鉴权？
5. 如何处理海量设备的并发连接？连接管理、负载均衡、集群化？
6. 设备上下线通知的实现方案？
7. 弱网环境下如何保证设备通信的可靠性？重连策略、消息缓存、QoS？

## 六、消息路由与数据流

1. 设备消息从上报到存储的完整链路是怎样的？
2. IoT 平台的消息路由设计？Topic 路由、规则引擎路由？
3. 如何处理设备消息的高吞吐？消息队列削峰、批量写入？
4. 设备数据的存储方案？时序数据库（TDengine / InfluxDB / TimescaleDB）的选型？
5. 设备指令下发的流程？同步下发 vs 异步下发？
6. 如何保证指令下发的可靠性？设备离线时的指令缓存？
7. 规则引擎在消息路由中的作用？数据过滤、转换、转发？

## 七、设备管理与运维

1. 设备生命周期管理？注册、激活、在线、离线、禁用、注销？
2. OTA（Over-The-Air）升级的实现方案？增量升级 vs 全量升级？
3. 设备分组与标签管理的设计？
4. 如何监控设备的在线状态？心跳检测、超时判断？
5. 设备日志的采集与分析方案？
6. 大规模设备的批量操作如何实现？批量配置、批量升级？

## 八、平台架构与高可用

1. IoT 平台的高可用架构设计？接入层、服务层、存储层的高可用？
2. MQTT Broker 集群的负载均衡策略？
3. 百万级设备连接下的性能瓶颈在哪里？如何优化？
4. IoT 平台的多租户架构设计？数据隔离、资源隔离？
5. IoT 平台的水平扩展方案？无状态服务 + 有状态连接的处理？
6. 如何做 IoT 平台的容量规划？连接数、消息吞吐、存储容量？

## 九、边缘计算与网关

1. 什么是边缘计算？在 IoT 中的作用？
2. 边缘网关的核心功能？协议转换、数据预处理、本地决策？
3. 云边协同的架构设计？数据同步、配置下发、模型更新？
4. 边缘计算与云计算的分工？哪些计算放在边缘，哪些放在云端？

## 十、安全与合规

1. IoT 设备的常见安全威胁？设备仿冒、数据窃听、中间人攻击、DDoS？
2. IoT 通信的加密方案？TLS 1.3、DTLS、端到端加密？
3. 设备固件的安全性如何保障？安全启动、固件签名？
4. IoT 数据的隐私合规？GDPR 对 IoT 数据的要求？

## 十一、实战与开放性问题

1. 你们的 IoT 平台用的什么 MQTT Broker？为什么这样选型？
2. 你们平台支撑了多少设备连接？消息吞吐量是多少？
3. 遇到过哪些 MQTT 相关的线上问题？怎么排查和解决的？
4. 如果让你从零设计一个 IoT 平台，你会怎么设计？
5. IoT 平台与传统 Web 后端开发的核心区别是什么？
6. 如何对 MQTT Broker 进行压测？关注哪些指标？
7. 你们的设备协议是怎么设计的？为什么这样设计？
