# GameBox Cloud Core V2

go-gbc2 是 GameBox Cloud Core 第二版的简称。

go-gbc2 的设计目标与 go-gbc 相比有所变化，更侧重于满足大规模在线游戏的需求。

go-gbc2 并不是一个完整的游戏服务器框架，只是一个采用“多房间”架构的分布式连接与消息处理框架。

如果要用于多人在线游戏，需要使用基于 go-gbc2 的游戏框架（这些框架会针对特定领域的游戏逐步推出）。

目标：

- 根据自定义逻辑将客户端归类到不同的 Room（房间）。
- 每个房间处理一组客户端的消息。
- 房间支持客户端 Peer-to-Peer（点对点）消息发送。
- 房间支持 Broadcast（广播）消息到房间内所有用户。
- 房间内支持多个群组，支持 Peer-to-Group（点对群组）消息群发。
- 每个客户端使用 WebSocket 和服务器保持长连接。
- 分布式架构，可以很轻松的横向 Scale-Up。

~

## 架构设计

主要架构如下图：

```

   xxx
  x   x
  x   x
   xxx             +---------+          ###############          +-----------+
    x              |         |          #             #          |           |
 xxxxxxx  < ---- > | GATEWAY | < ---- > # MESSAGE HUB # < ---- > |   LOBBY   |
    x              |         |          #             #          |           |
   x x             +---------+          ###############          +-----------+
  x   x                                                                   ^
 x     x               ^                       ^                          |
                       |                       |         +-----------+    |
 CLIENT                |                       |         |           |    |
                       |                       +------ > |   ROOM    |    |
                       |                                 |           |    |
                       |                                 +-----------+    |
                       |                                                  |
                       |                                       ▲          |
                       |      #############                    |          |
                       |      #           # < -----------------+          |
                       +--- > #   STORE   # < ----------------------------+
                              #           #
                              #############
```

在整个架构中，`Message Hub` 和 `Store` 是两个基础设施。
目前的实现，分别使用 [nats](https://nats.io) 和 [redis](https://redis.io)。

主要组件说明：

- **Gateway**: 负责和客户端之前维持长链接（WebSocket），接收来自客户端的消息，
  并将来自 Message Hub 的消息推送给客户端。

- **Lobby**: 负责管理所有的 **Room**，以及一些全局性的逻辑。
  通常以无状态的方式实现，以便实现横向扩展。

- **Room**: 负责管理一个房间内的所有客户端，并且提供点对点消息发送、群发消息等功能。
  每一个逻辑上的房间，会运行在某个 Room 进程中。
  这样做的好处是简化了 Room 的逻辑实现，缺点是 Room 进程的负载能力限制了同一个 Room 中的用户数量。
  如果需要在一个 Room 中处理大量用户，则可以在一个 Room 中划分多个次级 Room 来实现分布式处理。

- **Message Hub**: 使用基于订阅的消息中间件，完成各个组件之间的交互。
  优点是架构简单，代码逻辑清晰，容易实现横向扩展。
  缺点则是消息的传递消耗更多时间，服务端与客户端之间的交互时延更大。
  如果需要低时延的消息处理机制，可以使用 go-gbc2 的 gRPC 扩展组件。

- **Store**: 存储中间件，用于保存各个组件的状态，以及一些全局性的共享数据。
  每一个组件进程的状态都会注册到 Store 中，实现简单的服务发现与查找。
  如果有必要，可以将 Store 替换为 etcd 等其他存储系统。

附加组件：

- **Auth API**: 基于 HTTP 的用户身份认证接口，让客户端在连接到 Gateway 之前完成身份认证。

- **Admin API**: 基于 HTTP 的接口，用于读取整个系统的状态信息，并发送一些控制指令。

- **gRPC Extension**: 当需要低时延的消息处理时，可以让 Gateway 与 Lobby、Room 直接连接并交互。
  但这样做的缺点是组件之间会形成星形网络，内部消息流转变得更加复杂。
  所以建议仅在必须的情况下让 Gateway 和部分 Room 使用 gRPC 交互。

go-gbc2 最重要的优势就是为了应对大量在线用户的分布式架构。
整个系统中，Gateway、Lobby、Room 组件都是分布式的，可以按照需求部署任意数量个。

配合负载均衡，可以承载大量在线用户。

~

\- 未完，持续更新中 \-
