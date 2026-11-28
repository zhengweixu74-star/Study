# 长连接：建立、心跳保活、断开检测与重连

[返回笔记目录](../README.md)

## 本文目录

- [长连接的建立](#长连接的建立)
- [心跳：保留 NAT 映射与探活](#心跳保留-nat-映射与探活)
- [连接的关闭：发起方与关闭顺序](#连接的关闭发起方与关闭顺序)
- [异常断开的检测：RST 与写入超时](#异常断开的检测rst-与写入超时)
- [客户端判断网络状态](#客户端判断网络状态)
- [断线重连与消息补齐](#断线重连与消息补齐)
- [最小示例：WebSocket 的心跳与重连](#最小示例websocket-的心跳与重连)

## 长连接的建立

长连接是客户端和服务端建立后长时间保持、反复用来收发数据的 TCP 连接。一次请求结束后，长连接不关闭。服务端也可以随时通过这条连接主动向客户端发消息，所以即时消息、大模型的流式回复都依赖长连接。

以 WebSocket 长连接为例，客户端按以下顺序建立连接：

```text
DNS 解析 → TCP 三次握手 → TLS 握手 → WebSocket 升级 → 鉴权 → 开始收发
```

1. DNS 解析、TCP 三次握手和 TLS 握手与普通 HTTPS 请求相同，见[网络请求与代理](network-and-proxy.md#连接超时与幂等重试)。TLS 握手结束后，双方得到同一把会话密钥，之后的数据都用这把密钥加密。
2. 客户端发送带 `Upgrade: websocket` 请求头的 HTTP 请求，服务端返回 `101 Switching Protocols`，这条连接改为传输 WebSocket 帧，见[WebSocket：双向长连接](http-versions-and-streaming.md#websocket双向长连接)。
3. 客户端在连接上发送 token 和设备 ID。服务端校验通过后，记录“这个用户在这条连接上”。之后发给这个用户的消息，服务端从这条连接推下去。

## 心跳：保留 NAT 映射与探活

心跳是客户端定时在长连接上发送的很小的数据包，例如 `ping`；服务端收到后回复 `pong`。心跳有两个作用：保留 NAT 映射，以及探测连接是否还能用。

**保留 NAT 映射。** NAT（网络地址转换）是运营商网关上的地址转换功能。手机没有公网 IP，网关为每条 TCP 连接记录一条“内网地址 ↔ 公网地址”的映射。

一条连接长时间没有数据时，网关会删除这条映射，部分运营商网关的超时只有几分钟。映射被删除后，客户端和服务端都收不到任何通知，双方仍认为连接正常，但服务端发出的消息已经到不了手机。客户端在 NAT 超时之前发送一次心跳，网关就会保留映射。

**探活。** 探活是判断对方是否还能收发数据。例如心跳间隔为 30 秒：

- 客户端每 30 秒发送一次 `ping`。连续 3 次没有收到 `pong`，即 90 秒后，客户端判定连接已断开，主动关闭连接并重连。
- 服务端超过 90 秒没有收到这个客户端的心跳，就关闭这条连接，释放连接占用的内存。

TCP 协议自带的 keepalive 也能探活，但多数系统默认连接空闲 2 小时后才开始探测，远长于 NAT 超时，所以长连接使用应用层心跳。

**心跳间隔的取舍。** 心跳间隔要小于 NAT 超时，否则映射会被删除。手机每次发送心跳都要唤醒蜂窝射频，间隔越短越耗电、耗流量。有的应用使用智能心跳：逐步拉长间隔，测出当前网络的 NAT 超时，再取一个略小的值作为心跳间隔。

## 连接的关闭：发起方与关闭顺序

客户端和服务端都可以主动关闭长连接。在 TCP 中，主动关闭方是先发送 FIN 的一方；FIN 是 TCP 报文中表示“我这边不再发送数据”的标志位。

| 发起方 | 常见场景 |
| --- | --- |
| 客户端 | 用户离开聊天页面；应用退到后台；连接长时间空闲；网络从 Wi-Fi 切到蜂窝网络，旧连接不可用；心跳超时，判定连接已失效 |
| 服务端 | 客户端超过约定时间没有发送数据或心跳；token 过期；同一账号在其他设备登录，服务端让旧设备下线；服务发布或重启；服务端负载过高，主动释放一部分连接 |

**服务端的空闲超时。** 服务端或网关给每条连接设置最长空闲时间，超时后主动关闭连接，客户端此前不需要发送任何数据。例如 Nginx 转发 WebSocket 时，`proxy_read_timeout` 默认为 60 秒：60 秒内没有收到客户端的任何数据，Nginx 就关闭这条连接。所以客户端的心跳间隔还要小于服务端的空闲超时。

**正常关闭的顺序。** 正常关闭时，连接按协议从上到下逐层关闭：

```text
WebSocket Close 帧 → TLS close_notify 告警 → TCP 四次挥手
```

- WebSocket Close 帧是 WebSocket 协议的关闭消息，帧中带有关闭码，例如 1000 表示正常关闭，1001 表示一端正在离开。
- `close_notify` 是 TLS 协议的告警消息，通知对方不再发送加密数据。
- TCP 四次挥手中，双方各发送一次 FIN，各回复一次 ACK；ACK 是确认收到的报文。

**一次 HTTP 响应结束时，TCP 连接不一定关闭。** 服务端通过 SSE 返回一段流式回复，回复结束时 HTTP 响应结束。URLSession 把底层 TCP 连接留在连接池中，下一次向同一服务器发请求时直接复用这条连接，省去 TCP 握手和 TLS 握手。连接空闲超过超时时间后，URLSession 才关闭这条连接。

**应用退到后台时由客户端关闭。** iOS 应用退到后台后会被挂起，进程停止执行，无法再收发数据，见[iOS 后台执行机制](../ios/background-execution.md#挂起与后台执行的前提)。所以客户端在进入后台时主动关闭长连接，让服务端及时释放资源；回到前台时重新建立连接。应用在后台期间，消息通过系统推送通道送达，见[移动端远程推送](mobile-push-delivery.md#系统推送通道的作用)。

## 异常断开的检测：RST 与写入超时

连接断开分为正常关闭和异常断开两种：

- 正常关闭时，主动方发送 Close 帧和 FIN。对方读到连接结束，立即得知连接已关闭。
- 异常断开时，对方收不到任何通知。

一端的进程崩溃时，对方仍能收到通知。进程崩溃后，所在机器的内核仍在运行；内核替崩溃的进程关闭全部连接，并向对方发送 FIN 或 RST。

以下情况属于异常断开：

- 服务端机器断电，或内核崩溃。
- 手机失去网络信号。
- 运营商网关删除了 NAT 映射。

**异常断开要等下一次发送数据才能发现。** TCP 只通过收发数据包了解对方的状态。连接空闲时两端之间没有数据包，客户端的接收回调只会一直等待，不会报错。客户端发出一个数据包后，根据对方的反馈才能发现连接已失效，反馈有 RST 和写入超时两种。

**RST。** RST（Reset）是 TCP 报文中表示“这条连接不存在，请立即关闭”的标志位。TCP 规定：主机收到一个数据包，如果本机没有这个数据包所属连接的记录，就回复 RST。

以服务端机器重启为例：

1. 10:00:00，服务端机器重启，内核中所有 TCP 连接的记录被清空，客户端不知道。
2. 10:00:20，客户端按计划发送心跳 `ping`。
3. 服务端内核收到 `ping`，查不到这条连接，回复 RST。
4. 客户端内核收到 RST，立即关闭连接；应用得到 `ECONNRESET` 错误（connection reset by peer），开始重连。

从发出心跳到发现断开只需要一个往返时间，通常为几十毫秒。

**写入超时。** 写入超时是客户端发出的数据一直收不到 ACK，内核重传达到上限后放弃连接，并报告 `ETIMEDOUT` 错误。

以手机进入电梯失去信号为例：

1. 10:00:00，手机失去信号。
2. 10:00:20，应用发送心跳 `ping`。`ping` 写入内核的发送缓冲区后，发送调用就成功返回，此时没有任何错误。
3. 内核发出 `ping`，收不到 ACK，于是按越来越长的间隔重传，例如 0.2 秒、0.4 秒、0.8 秒。
4. 重传次数达到上限后，内核报告超时。Linux 默认配置下，这个过程约需 15 分钟。

内核报告超时太慢，所以应用自己统计心跳：连续 3 次（90 秒）收不到 `pong`，就判定连接已断开，见上文[心跳：保留 NAT 映射与探活](#心跳保留-nat-映射与探活)。

客户端判定连接断开的依据汇总如下：

| 依据 | 对应情况 |
| --- | --- |
| 读到连接结束（对方发来的 FIN） | 对方正常关闭，或对方进程崩溃 |
| 收到 RST | 对方主机没有这条连接的记录，例如服务端机器重启 |
| 写入超时 | 对方主机或中间网络无法到达 |
| 心跳超时 | 异常断开，应用比内核更早判定 |

## 客户端判断网络状态

客户端用系统接口和自己的统计判断网络状态：

| 手段 | 得到的信息 |
| --- | --- |
| `NWPathMonitor` | `status`：当前是否有可用网络；`isExpensive`：是否在使用蜂窝网络或个人热点；`isConstrained`：用户是否开启了低数据模式；当前使用的网络接口类型，例如 Wi-Fi 或蜂窝 |
| `URLSessionTaskMetrics` | 每个请求的 DNS 解析、建立连接、收到首个字节的耗时；可以据此计算往返时延和吞吐量 |
| 心跳往返时间 | 从发送 `ping` 到收到 `pong` 的耗时，直接反映长连接当前的延迟 |
| 请求结果统计 | 最近一段时间内请求的超时率和失败率 |

TCP 的拥塞窗口和接收窗口由内核自动调整发送速度，见[TCP 慢启动与快重传](network-and-proxy.md#tcp-慢启动与快重传)。iOS 没有提供公开接口让应用读取这两个窗口的大小，所以应用判断网络状态时使用上表中的手段。

网络较差时，客户端通常降级业务：降低图片清晰度、推迟不紧急的上传、合并多个请求。长连接本身保留：空闲长连接只在心跳时传输几十字节，关闭它腾不出多少带宽。手机上需要节省的主要是电量，做法是在 NAT 超时允许的范围内拉长心跳间隔，或者在后台关闭长连接。

## 断线重连与消息补齐

**指数退避与随机抖动。** 客户端断线后按指数退避重连：第 1 次等 1 秒，第 2 次等 2 秒，第 3 次等 4 秒，依次翻倍，达到上限（例如 60 秒）后不再增加。每次等待时间再加上一个随机值，这个随机值叫抖动。

服务端重启时，大量客户端同时断线。所有客户端都立即重连，或者都按相同的间隔重连，就会在同一时刻向刚恢复的服务端发起大量连接，再次压垮服务端。指数退避拉开每个客户端的重试间隔，抖动让不同客户端的重连时刻错开。

**网络切换后重建连接。** 手机从 Wi-Fi 切到蜂窝网络后，本机 IP 地址改变。TCP 连接由双方的 IP 地址和端口标识，所以旧连接已经不可用。客户端用 `NWPathMonitor` 监听网络变化，收到变化通知后主动关闭旧连接，立即建立新连接，不等待心跳超时。

**重连后补齐消息。** 服务端给每条消息分配一个递增的序号 `seq`。客户端重连并鉴权时，带上本地最后收到的 `seq`，服务端补发断线期间的消息。例如客户端最后收到 `seq = 1024`，断线期间服务端产生了 1025 到 1030 号消息；重连时客户端发送 `lastSeq = 1024`，服务端依次补发这 6 条消息。

## 最小示例：WebSocket 的心跳与重连

```swift
final class LongConnection {
    private var task: URLSessionWebSocketTask?
    private var heartbeatTimer: Timer?
    private var unansweredPings = 0
    private var retryDelay: TimeInterval = 1

    // 在主线程调用
    func connect() {
        // URLSession 在内部完成 DNS 解析、TCP 握手、TLS 握手和 WebSocket 升级
        let task = URLSession.shared.webSocketTask(with: URL(string: "wss://example.com/ws")!)
        self.task = task
        task.resume()
        task.send(.string(#"{"type":"auth","token":"xxx","lastSeq":1024}"#)) { _ in }
        receive(on: task)
        startHeartbeat()
    }

    private func receive(on task: URLSessionWebSocketTask) {
        task.receive { [weak self] result in
            DispatchQueue.main.async {
                // 旧连接的回调晚到时直接忽略，避免重复重连
                guard let self, task === self.task else { return }
                switch result {
                case .success:
                    self.retryDelay = 1          // 收到数据，连接正常，重置退避时间
                    self.receive(on: task)       // 继续等待下一条消息
                case .failure:
                    self.reconnect()             // 读到连接结束、RST 或其他错误
                }
            }
        }
    }

    private func startHeartbeat() {
        heartbeatTimer = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { [weak self] _ in
            guard let self, let task = self.task else { return }
            if self.unansweredPings >= 3 {       // 连续 3 次没有收到 pong
                self.reconnect()
                return
            }
            self.unansweredPings += 1
            task.sendPing { [weak self] error in // 收到 pong 或发送失败时回调
                DispatchQueue.main.async {
                    guard let self, task === self.task, error == nil else { return }
                    self.unansweredPings = 0
                }
            }
        }
    }

    private func reconnect() {
        heartbeatTimer?.invalidate()
        task?.cancel(with: .goingAway, reason: nil)
        task = nil
        unansweredPings = 0
        let delay = retryDelay + Double.random(in: 0...1)   // 指数退避加随机抖动
        retryDelay = min(retryDelay * 2, 60)
        DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
            self?.connect()
        }
    }
}
```
