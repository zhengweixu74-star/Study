# 网络请求与代理

[返回笔记目录](../README.md)

## 本文目录

- [HTTP 请求方法、安全性与幂等性](#http-请求方法安全性与幂等性)
  - [常见请求方法](#常见请求方法)
  - [GET 与 POST 的区别](#get-与-post-的区别)
- [请求正文格式与 JSON 字段](#请求正文格式与-json-字段)
- [使用 JSON 上传文件](#使用-json-上传文件)
- [gzip 压缩与抓包正文显示](#gzip-压缩与抓包正文显示)
- [multipart 文件上传与请求体结构](#multipart-文件上传与请求体结构)
  - [文件字段、文件名与两层头部](#文件字段文件名与两层头部)
  - [boundary 与完整请求体示例](#boundary-与完整请求体示例)
  - [服务端如何取得文件](#服务端如何取得文件)
- [Swift 构造 multipart 请求与上传](#swift-构造-multipart-请求与上传)
  - [分隔行如何变成字节](#分隔行如何变成字节)
  - [单个 JPEG 文件的最小示例](#单个-jpeg-文件的最小示例)
  - [大文件的内存边界](#大文件的内存边界)
- [连接、超时与幂等重试](#连接超时与幂等重试)
- [TCP 慢启动与快重传](#tcp-慢启动与快重传)
- [UDP 上的可靠传输与 QUIC](#udp-上的可靠传输与-quic)
  - [数据包编号与数据位置](#数据包编号与数据位置)
  - [TCP 序号与重传的区别](#tcp-序号与重传的区别)
  - [乱序缓存与流内等待](#乱序缓存与流内等待)
  - [QUIC、TLS 与 HTTPS](#quictls-与-https)
- [代理连接与 HTTPS 隧道](#代理连接与-https-隧道)
- [HTTPS 抓包与两段 TLS 连接](#https-抓包与两段-tls-连接)
- [证书公钥、CA 签名与会话密钥](#证书公钥ca-签名与会话密钥)
- [证书链、中间 CA 与根 CA 信任](#证书链中间-ca-与根-ca-信任)
- [“证书不可用”的归因与排查](#证书不可用的归因与排查)
- [URLSession 的职责与请求流程](#urlsession-的职责与请求流程)
- [URLSessionConfiguration 与连接条件](#urlsessionconfiguration-与连接条件)
- [URLSessionDelegate 与重定向决定](#urlsessiondelegate-与重定向决定)
- [URLCache 与 HTTP 缓存规则](#urlcache-与-http-缓存规则)
- [Alamofire：请求与响应模型](#alamofire请求与响应模型)
  - [响应容器、Result 与模型值](#响应容器result-与模型值)
  - [请求信息、HTTP 响应与正文](#请求信息http-响应与正文)

## HTTP 请求方法、安全性与幂等性

HTTP 是应用层的请求与响应协议。客户端在请求中指定目标资源和请求方法。服务端根据请求执行操作，并返回状态码、响应头和可选的响应体。请求方法表达客户端希望对目标资源执行的操作。

### 常见请求方法

| 方法 | 含义 | 安全性 | 幂等性 |
| --- | --- | --- | --- |
| GET | 获取目标资源的表示，例如一份 JSON 数据 | 安全 | 幂等 |
| HEAD | 获取与 GET 对应的响应元信息，不返回响应体 | 安全 | 幂等 |
| POST | 让目标资源按自身规则处理请求内容，例如提交表单或创建记录 | 不安全 | 不保证 |
| PUT | 用请求内容创建或替换指定目标资源的状态 | 不安全 | 幂等 |
| PATCH | 按请求中的修改指令局部修改资源 | 不安全 | 不保证 |
| DELETE | 删除目标 URI 与当前资源功能之间的关联 | 不安全 | 幂等 |
| OPTIONS | 查询目标资源或服务端支持的通信选项 | 安全 | 幂等 |

安全性是请求方法的语义属性：客户端只读取资源，不要求服务端改变业务状态。服务端为安全请求记录访问日志，仍然符合安全性。这里的“安全”只描述操作语义；传输加密由 HTTPS 负责。

幂等性表示：对同一目标重复执行相同请求，服务端的预期效果与执行一次相同。幂等性只约束服务端效果，每次返回的响应内容和状态码可以不同。例如，客户端重复删除同一个资源，第一次返回删除成功，第二次返回资源不存在；资源只被删除了一次。

HTTP 协议不保证 POST 和 PATCH 幂等，具体接口可以自行提供幂等行为，例如服务端按请求标识去重。DELETE 的协议语义是删除 URI 与资源的关联；服务端可以自行决定何时物理删除存储中的数据。

客户端、缓存和代理根据请求方法判断操作语义。例如，缓存在满足缓存条件时复用响应；客户端根据方法是否幂等以及失败情况，决定能否自动重试。所以服务端实现每个方法时，也应遵守这个方法的语义。

### GET 与 POST 的区别

| 对比项 | GET | POST |
| --- | --- | --- |
| 核心语义 | 获取资源的表示 | 让目标资源处理提交的内容 |
| 安全性与幂等性 | 安全且幂等 | 均不保证 |
| 常见参数位置 | URL 路径和查询参数 | 请求体，也可以同时带 URL 查询参数 |
| 请求体 | 没有通用定义的请求体语义，通常不发送 | 常用于承载提交的数据，格式由 Content-Type 和接口约定确定 |
| 响应缓存 | 满足缓存条件时可缓存 | 满足 HTTP 缓存规范的条件时也可缓存，但通用实现支持较少 |

请求体是 HTTP 消息中承载本次提交内容的部分。HTTP 请求头则是描述请求及其内容的一组字段。Content-Type 用于声明请求体的媒体类型，即内容格式，例如 application/json 或 multipart/form-data。

GET 与 POST 的主要区别是操作语义：GET 获取资源，POST 让资源处理提交的内容。两者都可以携带参数。GET 参数的长度上限由客户端、代理和服务端各自决定，HTTP 协议没有规定统一的上限。

POST 把数据放在请求体中，数据本身仍是明文；GET 和 POST 都要靠 HTTPS 加密传输。参数位置决定数据会留在哪里：URL 中的参数可能被记录到浏览历史或访问日志中。

## 请求正文格式与 JSON 字段

Content-Type 声明请求正文的媒体类型，即正文采用的数据格式。application/json 表示 JSON 格式，multipart/form-data 表示由多个表单部分组成的格式。只有 multipart/form-data 格式的正文才包含 boundary 分隔行。

| 对比项 | application/json | multipart/form-data |
| --- | --- | --- |
| 字段名的位置 | JSON 对象的键，例如 title | 各部分 Content-Disposition 中的 name 参数，例如 name="title" |
| 字段值的位置 | JSON 对象中对应键的值 | 各部分头部后的空行之后 |
| 内容如何分隔 | 由 JSON 的括号、冒号和逗号等语法组织 | 由 boundary 对应的分隔行组织 |
| 文件内容如何表示 | 按接口约定转换为字符串等 JSON 数据，例如 Base64 字符串 | 可在文件部分直接携带原始文件字节 |

例如，下面的请求正文提交 title 和 count 两个字段：

```http
Content-Type: application/json
```

```json
{
  "title": "hello",
  "count": 10
}
```

JSON 对象的键 title 和 count 本身就是字段名，所以 JSON 正文不需要 name="title" 这样的参数，也不需要 boundary 分隔行。JSON 中名为 name 或 boundary 的键只是普通接口数据，没有 multipart 的协议含义。

在 multipart/form-data 中，boundary 是用于构造各部分分隔行的标识字符串；name 是每个部分的 Content-Disposition 中用于标识表单字段的参数。两者的具体结构见本文“multipart 文件上传与请求体结构”。

## 使用 JSON 上传文件

客户端可以用 application/json 上传文件：客户端先把文件字节转换成 JSON 能表示的数据，服务端再按相同约定还原。转换是必需的，因为 JSON 没有表示二进制字节的数据类型，图片字节无法直接写进 JSON 文本。

Base64 是一种可逆编码，把字节序列编码成文本；任何人都能解码 Base64，所以 Base64 不提供加密保护。客户端可以把 Base64 文本放入 JSON 字符串。下面示例上传一个文件名为 hello.txt、内容为 hello 的文件；请求头使用 Content-Type: application/json。

```json
{
  "filename": "hello.txt",
  "contentBase64": "aGVsbG8="
}
```

1. 客户端读取文件字节，将字节编码成 Base64 字符串。
2. 客户端把文件名和 Base64 字符串写入 JSON，并发送 JSON 正文。
3. 服务端解析 JSON，读取约定字段，再把 Base64 字符串解码为原始文件字节。
4. 服务端根据接口约定保存或处理文件。

filename 和 contentBase64 是示例接口自己约定的字段名，JSON 没有规定文件字段。服务端代码要按约定读取 contentBase64，自行完成 Base64 解码和文件保存。客户端只发送本地路径或文件地址字符串时，正文中没有文件字节，服务端拿不到文件内容。

对于 n 字节的数据，带标准填充且不插入换行的 Base64 文本长度为 4 × ceil(n / 3) 个字符，ceil 表示向上取整。数据量较大时，编码后的长度约为原来的 4/3，正文还要加上 JSON 字段名等开销。以上计算不考虑之后的压缩。

multipart/form-data 的文件部分直接承载原始文件字节，无须 Base64 编码。客户端使用 JSON 还是 multipart，由服务端接口约定决定。Content-Type 请求头只声明正文格式；正文本身要由客户端按这个格式构造。

## gzip 压缩与抓包正文显示

gzip 是一种无损压缩格式：发送方用 gzip 把原始字节压缩成另一段字节，接收方解压后得到原来的字节。正文的数据格式和压缩方式分属两层：JSON 正文经过 gzip 压缩，解压后仍是原来的 JSON 文本。

| 请求头 | 作用 |
| --- | --- |
| Content-Type: application/json | 声明解码后正文的数据格式为 JSON |
| Content-Encoding: gzip | 声明正文应用了 gzip 内容编码，接收方需要先解压再按媒体类型解析 |
| Content-Length（存在时） | 声明消息正文的字节长度；正文经过 gzip 编码时，对应编码后的长度，不是解压后的字符数 |

客户端发送 gzip 压缩的 JSON 时，按标准同时设置以下两个请求头。服务端需要支持解压 gzip 请求正文：

```http
Content-Type: application/json
Content-Encoding: gzip
```

序列化是把内存中的数据转换成可存储或传输的格式。发送和接收的顺序是：客户端把数据序列化为 JSON 文本 → 客户端把文本编码成 UTF-8 字节 → 客户端用 gzip 压缩字节 → 接收方用 gzip 解压 → 接收方把解压后的文本解析为 JSON。

某些接口用通信双方约定的自定义请求头声明压缩方式。抓包工具按标准的 Content-Encoding 识别压缩，可能识别不了自定义请求头。请求和响应各自有头部和正文：请求正文是否压缩，要看请求的头部；响应头中的 Content-Encoding 只描述响应正文。

十六进制视图把每个字节显示为两位十六进制数，只改变显示方式，不改变数据。gzip 数据常见的起始三个字节为 1F 8B 08，其中前两个字节标识 gzip，第三个字节表示使用 DEFLATE 压缩方法。这个前缀可以帮助识别 gzip 数据；整段数据是否完整有效，要解压后才能确认。

抓包工具中的 JSON、文本和十六进制视图是不同的展示方式。工具先解压 gzip、得到有效的 JSON 文本后，JSON 视图才能显示 JSON 结构；切换视图本身不执行解压。HTTPS 解密只去掉 TLS 加密层，正文中的 gzip 压缩层仍然保留。

## multipart 文件上传与请求体结构

上传文件时，客户端必须发送文件的实际字节。远程服务端无法读取客户端本地路径上的文件。

multipart/form-data 是一种把多个表单项放在同一个请求体中的格式，每个部分承载普通文本或文件数据。文件上传可以用 multipart，也可以用 JSON 等其他格式，由接口约定决定。

### 文件字段、文件名与两层头部

表单字段是表单中的一个命名数据项。文件字段是承载文件数据的表单字段。下文的 file 是示例接口自己约定的字段名，写在请求体内的部分头部中；HTTP 没有规定这个名称。

multipart 的每个部分都有自己的头部，说明这一部分的字段名、文件名和内容类型。部分头部位于 HTTP 请求体内部；HTTP 请求头位于请求体之前，描述整个请求。

| 示例内容 | 所在位置 | 含义 |
| --- | --- | --- |
| Content-Type: multipart/form-data; boundary=Boundary-ABC123 | HTTP 请求头 | 整个请求体采用 multipart 格式，并指定分隔标识 |
| Content-Disposition: form-data; name="file"; filename="photo.jpg" | 请求体中，文件部分的头部 | 该部分对应名为 file 的表单字段，提交的文件名是 photo.jpg |
| Content-Type: image/jpeg | 请求体中，文件部分的头部 | 声明该部分的数据格式是 JPEG 图片 |
| 图片的实际字节 | 请求体中，文件部分的内容 | 服务端接收的文件数据 |

name="file" 中的 file 是 name 参数的值，即字段名；filename="photo.jpg" 是文件名。字段名和文件名互相独立：把字段名改成 avatar，文件名仍是 photo.jpg。Content-Type: image/jpeg 和扩展名 .jpg 只是声明，文件字节本身必须是 JPEG 编码数据。

### boundary 与完整请求体示例

boundary 是用于构造各部分分隔行的标识字符串。客户端在 HTTP 请求头中声明该字符串，并在请求体中使用相同字符串。下面示例的 boundary 值为 Boundary-ABC123。

HTTP 请求头中的 Content-Type：

```http
Content-Type: multipart/form-data; boundary=Boundary-ABC123
```

请求体包含一个普通文本字段 description 和一个文件字段 file：

```text
--Boundary-ABC123
Content-Disposition: form-data; name="description"

我的头像
--Boundary-ABC123
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

[这里是图片的实际字节]
--Boundary-ABC123--
```

方括号中的文字标出图片字节的位置，实际发送时替换为图片字节。示例中的换行显示为普通换行；实际发送时，分隔行和头部行以 CRLF 结尾。CRLF 即回车加换行，对应字节 0x0D、0x0A。

普通分隔行是 -- 加 boundary，例如 --Boundary-ABC123。开头的两个连字符是固定前缀，不属于请求头中声明的 boundary 值。结束分隔行在末尾再加 --，表示 multipart 请求体结束。每个部分的头部与内容之间有一个空行；前一部分的内容结束后，先写 CRLF，再写下一条分隔行。

分隔行不能出现在任何部分的内容中，否则服务端会把这段内容误判为分隔行。客户端通常用包含随机值的字符串作为 boundary，降低冲突概率。请求头声明的 boundary 必须与请求体中使用的 boundary 一致，服务端才能按声明拆分各部分。

### 服务端如何取得文件

1. 服务端读取 HTTP 请求头中的 Content-Type，确定请求体格式与 boundary。
2. multipart 解析器按分隔标记识别请求体中的各个部分。
3. 解析器读取各部分的 Content-Disposition，取得 name 和可选的 filename。
4. 服务端框架将普通字段和文件字段提供给接口代码。接口代码按照约定的字段名取得对应文件，并保存或处理文件字节。

解析后的逻辑关系可以表示为：

```text
普通字段["description"] → "我的头像"
文件字段["file"]        → photo.jpg 的信息和字节内容
```

文件数据来自请求体，字段名标识文件对应哪个接口参数。接口代码只按字段名 file 读取文件时，客户端若发送 name="avatar"，文件字节虽然已经到达服务端，接口仍可能报告缺少 file 字段。服务端也可以遍历所有文件部分，不依赖字段名。

## Swift 构造 multipart 请求与上传

Foundation 的 Data 是用于存放字节的值类型。URLRequest 描述请求地址、方法和请求头。URLSession 负责执行网络传输。使用 multipart 上传时，调用方先按 multipart 格式构造请求体，再交给 URLSession 发送。

### 分隔行如何变成字节

`Data("--\(boundary)\r\n".utf8)` 把一条分隔行及结尾的 CRLF 转换为字节。这个表达式可以拆成以下四步：

```swift
let boundary = "Boundary-ABC123"
let text = "--\(boundary)\r\n"
let bytes = text.utf8
let data = Data(bytes)
```

字符串插值 `\(boundary)` 把 `boundary` 的值插入字符串。开头的 `--` 是 multipart 分隔行的固定前缀。`\r\n` 依次表示回车和换行。字符串的 `utf8` 属性提供 UTF-8 编码的字节视图，`Data(bytes)` 再用这些字节创建 Data 值。

`body.append(data)` 把这条分隔行追加到请求体末尾。分隔行和部分头部是文本，要先编码为字节再追加；文件内容本来就是 Data，直接追加原始字节。

### 单个 JPEG 文件的最小示例

下面的示例假设接口接受 POST 和 multipart/form-data，文件字段名为 file。jpegData 必须已经是 JPEG 编码数据，uploadURL 必须是实际上传接口的 HTTPS 地址。异步 upload(for:from:) 在 iOS 15 及以上可用。

```swift
import Foundation

@available(iOS 15.0, *)
func uploadJPEG(
    _ jpegData: Data,
    to uploadURL: URL
) async throws -> Data {
    let boundary = "Boundary-\(UUID().uuidString)"
    var body = Data()

    body.append(Data("--\(boundary)\r\n".utf8))

    let disposition = "Content-Disposition: form-data; "
        + "name=\"file\"; filename=\"photo.jpg\"\r\n"
    body.append(Data(disposition.utf8))
    body.append(Data("Content-Type: image/jpeg\r\n\r\n".utf8))
    body.append(jpegData)
    body.append(Data("\r\n--\(boundary)--\r\n".utf8))

    var request = URLRequest(url: uploadURL)
    request.httpMethod = "POST"
    request.setValue(
        "multipart/form-data; boundary=\(boundary)",
        forHTTPHeaderField: "Content-Type"
    )

    let (responseData, response) = try await URLSession.shared.upload(
        for: request,
        from: body
    )

    guard let httpResponse = response as? HTTPURLResponse,
          (200..<300).contains(httpResponse.statusCode) else {
        throw URLError(.badServerResponse)
    }

    return responseData
}
```

客户端按“分隔行 → 文件部分头部 → 空行 → 图片字节 → 结束分隔行”的顺序构造 body。`upload(for:from:)` 创建并启动上传任务，上传完成后返回服务端的响应体 `responseData` 和响应对象 `response`。`responseData` 是服务端返回的数据；上传的图片字节在 `body` 中。

示例只检查 HTTP 状态码是否为 2xx。真实接口还可能要求认证请求头，并在响应体中用业务状态字段表示处理结果。调用方需要按接口约定解析响应体，才能确认业务处理成功。

### 大文件的内存边界

示例把完整的 multipart 请求体组装进一个 Data，整个请求体都留在内存中，所以这种写法不适合大文件。上传大文件时，可以分段读取源文件，把部分头部、文件字节和结束分隔行依次写入临时文件，再用 `upload(for:fromFile:)` 发送这个临时文件。

`fromFile` 参数指向的文件就是完整的 HTTP 请求体，URLSession 不会为它生成 boundary 分隔行或部分头部。接口要求 multipart 时，临时文件中必须写好完整的 multipart 格式。上传期间临时文件要保持可读；上传完成或取消后，再删除临时文件。

## 连接、超时与幂等重试

基于 TCP 的 HTTPS 请求依次经过 DNS 解析、TCP 连接、TLS 握手和 HTTP 交换四个阶段。TCP 连接通过三次握手建立；丢包和重传的具体表现取决于发生在哪个阶段。TLS 1.3 已经移除旧式 RSA 密钥传输，按 RSA 密钥传输讲解的握手过程不适用于 TLS 1.3。HTTP/3 使用 QUIC，没有 TCP 连接阶段。

请求超时只表示调用方没有拿到确定结果，服务端可能已经执行了请求。客户端重试同一次业务操作时，沿用原来的操作标识，服务端按操作标识去重；新的业务操作使用新的标识。幂等重试要求客户端和服务端都遵守这一约定。

## TCP 慢启动与快重传

慢启动逐步探测网络能够承受的发送量；快重传根据重复确认推断丢包，在重传定时器超时之前补发数据。以下按经典 TCP 算法说明，具体实现可能使用其他拥塞控制或丢包检测算法。

拥塞窗口 `cwnd` 是发送方对未确认数据量的限制。接收窗口 `rwnd` 是接收方通告的可接收数据量。发送方未确认的数据量同时受 `cwnd` 和 `rwnd` 限制。`RTT` 是发送数据到收到对应确认的往返时间。

**慢启动：**发送方从较小的窗口起步，每收到一个确认新数据的 ACK 就增大窗口。理想条件下（每个数据段都及时得到独立确认），窗口大约每经过一个 RTT 翻倍，例如 `2 → 4 → 8 → 16` 个数据段，其中初始值 2 只是示例。“慢”指起始发送量小；窗口的增长通常接近指数增长。

`ssthresh` 是慢启动阈值。窗口小于阈值时使用慢启动，大于阈值时使用拥塞避免；等于阈值时经典规范允许采用其中任一种。经典拥塞避免阶段近似每个 RTT 增加一个最大数据段的大小。

**快重传：**ACK 是确认报文，ACK 中的累计确认号是接收方下一步期望收到的字节序号。中间某段数据缺失时，接收方每收到一个后续数据段，就再发一个确认号指向缺失位置的 ACK，即重复 ACK。发送方收到 3 个满足判定条件的重复 ACK、且累计确认号一直没有推进时，推断这段数据可能丢失，立即重传这段数据。

例如，发送方收到 `ACK=1001` 后，又收到 3 个确认号同为 1001 的重复 ACK，于是重传从第 1001 字节开始的缺失数据段。相同确认号共出现 4 次：1 次首次确认，加 3 次重复确认。数据段乱序到达也会产生重复 ACK，所以快重传对丢包的判断是推断。

快重传决定何时补发；快恢复负责丢包后的窗口与发送控制。

## UDP 上的可靠传输与 QUIC

UDP 以数据报为单位发送数据。数据报是一次发送形成的消息单元。UDP 本身不保证数据一定到达、按顺序到达或只到达一次，也不自动重传丢失的数据。

应用可以在 UDP 之上自己实现可靠传输：

1. 发送方给数据分配序号或字节位置。
2. 接收方返回 ACK，也就是接收确认。
3. 发送方根据确认和计时结果决定是否补发。
4. 接收方缓存乱序数据、丢弃重复数据，并按协议要求把数据交给应用。

流量控制根据接收方的接收能力限制发送量；拥塞控制根据网络状况调整发送量，减少网络过载。

网络一直不通时，可靠传输也无法完成交付；协议通过超时、关闭连接或错误通知告诉应用交付失败。QUIC 是运行在 UDP 之上的传输协议，提供连接管理、可靠数据流、流量控制、拥塞控制及安全保护。

### 数据包编号与数据位置

QUIC 的流是连接内一条独立编号的有序字节序列。流 ID 标识哪条流，流偏移量标识数据在该流中的字节位置。QUIC 数据包是一次发送的载体；数据包内部包含帧，帧是承载数据片段或控制信息的协议单元。

数据包编号标识一次发送；流 ID 和偏移量标识被传输的数据。编号空间是单独维护编号和确认的一组数据包。在同一个编号空间内，每次发送都使用新的数据包编号，补发旧数据时也使用新编号。下面的例子都在同一个应用数据编号空间中。

假设包 10 包含流 A 的第 0～99 字节，发送方判定包 10 丢失。发送方可以把仍需交付的数据重新装入后续数据包：

| 发送安排 | 包中承载的数据 |
| --- | --- |
| 保持这段数据完整 | 包 15 承载 A 的第 0～99 字节 |
| 拆分数据 | 包 15 承载 A 的第 0～49 字节，包 16 承载 A 的第 50～99 字节 |
| 与其他数据组合 | 包 15 承载 A 的第 0～99 字节，同时承载另一条流的数据；前提是满足包大小与发送限制 |

重新组合后，数据的字节内容和流位置保持不变。发送方只补发仍需交付的信息，不复制旧包的完整结构：旧包中的 ACK 通常不原样重传；某些控制信息改为发送最新状态；旧包中已经通过其他包得到确认的流数据，不再补发。

接收方确认包 15 时，发送方能确定这次确认对应包 15 这次发送，因此能准确计算往返时间、判断丢包。如果包 10 只是迟到，接收方按流 ID 和偏移量识别出重复字节，不会把这些字节重复交给应用。

### TCP 序号与重传的区别

TCP 普通数据段的 `SEQ` 是这个数据段第一个数据字节的序号，标识数据位置，不区分第几次发送。假设一个数据段的 `SEQ=1000`、长度为 100 字节。TCP 重传这段字节时，字节序号仍是 1000～1099；重传时还可以重新分段，新的网络包不必与原来完全相同。

假设此前字节都已连续收到，接收方收到这 100 字节后返回 `ACK=1100`，表示下一步期望第 1100 字节。只看这个累计确认号，发送方无法确定 ACK 对应的是首次发送还是重传。往返时间测量因此存在歧义，TCP 的计时算法需要专门处理这种情况。

### 乱序缓存与流内等待

假设一条流的数据被分成连续的 1、2、3、4、5 五段。这里的段号表示数据在流中的先后位置，与 QUIC 数据包编号无关。若第 2 段缺失，第 1 段可以先交给应用；在接收限制允许时，第 3、4、5 段仍可进入接收方内存，等待缺口补齐。

接收方把已到达的数据存放在对应的字节位置，第 3 段先到达不影响第 2 段之后到达。第 2 段到达后，第 1～5 段的字节变得连续，接收方继续按顺序把第 2～5 段交给应用。所以等待只发生在按顺序交付给应用这一环节；后续数据段照常从网络接收。

接收方的缓存容量有限。接收方通过流量控制规定发送方可以发送的字节范围，发送方必须遵守。发送新字节会占用流量控制额度；补发缺失的原有字节不额外占用额度。

传输层队头阻塞是指某条流缺失的数据迫使其他流已收到的数据也停止交付。QUIC 中一条流出现缺口时，其他流照常交付，所以 QUIC 避免了跨流的传输层队头阻塞。同一条流内仍要等前面的缺口补齐；不同流之间仍可能受到连接共享的拥塞控制、连接级流量控制以及应用自身依赖关系的影响。

### QUIC、TLS 与 HTTPS

HTTPS 表示受到安全传输保护的 HTTP 通信；HTTP/3 是 HTTP 的一个版本；QUIC 是 HTTP/3 使用的传输协议。三者属于不同层次，组合起来使用。

| 常见协议组合 | 从 HTTP 到网络传输的关系 |
| --- | --- |
| HTTPS 使用 HTTP/1.1 或 HTTP/2 | HTTP → TLS → TCP → IP |
| HTTPS 使用 HTTP/3 | HTTP/3 → QUIC → UDP → IP；QUIC 集成 TLS 1.3 的握手与密钥协商 |

在 QUIC 中，TLS 1.3 负责身份验证和密钥协商，QUIC 用协商出的密钥直接保护数据包，应用数据不再封装成传统的 TLS 记录。HTTP/3 同样使用 `https://` 地址，所以只看地址前缀，无法判断连接使用 TCP 还是 QUIC。QUIC 也可以承载 HTTP/3 之外的应用协议。

## 代理连接与 HTTPS 隧道

手机配置代理后，遵循代理设置的网络库把“连接谁”和“最终请求谁”分开：业务 URL 仍是服务端域名，网络库实际连接的是电脑代理的 IP 和端口。IP 定位到电脑，端口定位到电脑上监听这个端口的代理进程。代理进程再根据请求中的目标地址连接服务端，并把服务端的响应转回手机。

```text
手机 App → 电脑代理程序 → 目标服务端
手机 App ← 电脑代理程序 ← 目标服务端
```

客户端通过代理发送 HTTP 请求时，请求中带有目标 URL、Host 等信息，代理据此连接服务端。客户端通过常见的 HTTP 代理发送 HTTPS 请求时，通常先发送 `CONNECT 域名:443`，让代理建立到服务端的隧道，再在隧道中进行 TLS 通信。代理只转发 HTTPS 密文时，不需要解密。要用抓包代理查看明文，通常需要手机信任代理证书，并且代理证书要通过 App 的证书校验；App 启用证书绑定时，可能拒绝代理证书。

代理方式不修改服务端域名的 DNS 解析结果，网络库只是把连接发给代理。使用代理需要满足两个条件：手机能访问代理地址；电脑上的代理监听手机可访问的网卡地址和端口。App 自建连接或使用不遵循系统代理设置的网络栈时，这部分流量可能绕过代理。

## HTTPS 抓包与两段 TLS 连接

HTTPS 使用 TLS 保护 HTTP 通信。TLS 提供身份验证、传输加密和数据完整性校验。在网络上监听只能捕获密文以及 IP、端口等信息；要查看 HTTP 明文，必须具备解密条件，例如下文的抓包代理。

抓包代理解密 HTTPS 需要同时满足三个条件：代理启用 HTTPS 解密；流量经过代理；客户端接受代理证书。这时客户端与代理建立 TLS 连接 A，代理与真实服务器建立 TLS 连接 B。连接 A 和连接 B 各自完成 TLS 握手，各自协商并派生密钥。每条连接中，两个传输方向使用不同的密钥。

```text
客户端 ←─ TLS 连接 A ─→ 抓包代理 ←─ TLS 连接 B ─→ 真实服务器
                            │
                       解密并记录明文
```

请求转发顺序：客户端用连接 A 的发送密钥加密 → 代理用连接 A 对应的接收密钥解密并记录明文 → 代理用连接 B 的发送密钥重新加密 → 服务器用连接 B 对应的接收密钥解密。响应按相反方向转发。

代理参与了两次握手，持有连接 A 和连接 B 的密钥，所以能读取明文；代理不需要真实服务器的私钥。普通 CONNECT 隧道只转发密文，客户端与服务器之间仍是一条完整的 TLS 连接。

**iPhone 配合 Charles 的操作步骤：**

1. 让手机能访问电脑，通常连接同一个 Wi-Fi。在手机当前 Wi-Fi 的 HTTP 代理设置中填写电脑局域网 IP 和 Charles 实际监听的端口，并在 Charles 中允许手机连接。
2. 配置代理后，按照 Charles 的移动设备证书安装提示下载证书描述文件，并在系统设置中完成安装。
3. 打开“设置 → 通用 → 关于本机 → 证书信任设置”，启用对 Charles 根证书的完全信任。安装证书描述文件后，必须在这里手动开启信任，TLS 连接才会信任 Charles 根证书。
4. 在 Charles 中为目标域名和端口启用 SSL Proxying，例如 api.example.com:443。重新建立连接并发起请求，查看请求头、请求体和响应。

证书固定（或公钥固定）是 App 额外做的校验：App 要求服务器证书或公钥与 App 内预设的值匹配。系统信任代理根证书后，证书固定仍可能拒绝代理证书。抓包自己开发的 App 时，需要在调试版本中为证书固定提供相应配置。抓包成功需要两个条件同时成立：网络请求遵循代理设置；客户端接受代理证书。

## 证书公钥、CA 签名与会话密钥

公钥可以公开，私钥由持有者保密。数字签名是持有者用私钥对数据计算出的一段验证信息。验证方用对应的公钥检查签名，确认数据没有被修改，并确认签名者持有这把私钥。CA 是证书签发机构。服务器证书包含域名、公钥、有效期等信息，以及签发者的签名。

以使用证书认证的 TLS 1.3 为例，需要区分两次验证：

| 验证对象 | 签名密钥 | 验证密钥 | 目的 |
|-|-|-|-|
| 服务器证书内容 | 签发该证书的 CA 私钥 | 对应 CA 的公钥 | 验证签发来源和内容完整性，确认 CA 认可域名与公钥的绑定 |
| 本次 TLS 握手相关内容 | 服务器证书公钥对应的私钥 | 服务器证书中的公钥 | 确认当前参与握手的对端持有对应私钥；只有证书副本不够 |

抓包时，Charles 用自己的 CA 私钥为目标域名签发一张证书。客户端预先信任了 Charles 根证书，并且证书路径、域名等检查都通过后，才接受 Charles 签发的目标域名证书。握手时，Charles 用目标域名证书中公钥对应的私钥完成握手签名。

HTTP 请求和响应由本次连接派生的对称密钥加密。在 TLS 1.3 中，证书公钥只用于认证，CA 私钥只用于签发证书，两者都不加密 HTTP 内容。

## 证书链、中间 CA 与根 CA 信任

证书链是按签发关系连接的一组证书。根 CA 证书提供客户端已经信任的起点；中间 CA 证书包含中间 CA 的公钥和上级 CA 的签名；服务器证书包含目标域名和服务器公钥。

```text
客户端已经信任的根 CA 证书
    │ 根 CA 私钥签发
    ▼
中间 CA 证书
    │ 中间 CA 私钥签发
    ▼
服务器证书，例如 api.example.com
```

在只有一级中间 CA 的例子中，服务器证书的签发者是中间 CA，中间 CA 证书的签发者是根 CA。客户端使用中间 CA 公钥验证服务器证书签名，再使用已信任根 CA 公钥验证中间 CA 证书签名。实际的证书链可以有多级中间 CA。除了签名，客户端还要检查有效期、用途、CA 签发权限和目标域名等条件。

客户端信任哪些根 CA 证书，由操作系统、浏览器或应用的预置列表决定，用户或管理员也可以手动添加。客户端只信任本地信任列表中的根证书；服务器发来的根证书和根证书上的自签名，都不会让客户端信任这张根证书。

**缺少中间证书：**服务器证书由中间 CA 签发时，客户端必须用中间 CA 的公钥验证服务器证书的签名，根 CA 的公钥验证不了这个签名。服务器没有提供所需的中间证书，客户端也无法从缓存、网络等来源补齐时，客户端可能无法建立到可信根证书的验证路径。

服务器通常发送服务器证书和必要的中间证书，一般不发送根证书。服务器漏发中间证书时，部分客户端能自行补齐，另一部分不能，所以同一个服务可能出现“部分设备能访问、部分设备失败”。

## “证书不可用”的归因与排查

“证书不可用”提示只说明某次证书验证失败，原因可能在服务端、客户端或抓包代理。普通 HTTPS 中，服务端提供证书，客户端执行验证。客户端报告验证失败后，还要确定是哪一方验证了哪张证书。

| 可能来源 | 检查内容 |
|-|-|
| 服务端配置 | 证书过期、证书不包含实际请求域名、缺少必要的中间证书 |
| 客户端环境或配置 | 设备时间错误、不信任对应 CA、请求域名错误、固定证书或公钥与服务端更新后的配置不匹配 |
| 抓包代理 | 客户端未信任代理根证书、App 拒绝代理证书，或代理验证真实服务器证书失败 |

1. 读取底层错误码、失败域名和 TLS 握手日志，区分过期、域名不匹配、不受信任等失败类型。
2. 检查实际收到的证书，包括签发者、域名、有效期及中间证书，确认收到的是服务器证书还是代理证书。
3. 关闭代理后重试，并比较不同设备或客户端的结果。对照结果用于缩小排查范围；确定根因还要靠第 1、2 步得到的错误码和证书信息。
4. 依据证据修复对应配置。例如，在准确系统时间下确认服务器证书过期，就更新服务端证书；确认未信任调试代理 CA，就修复客户端调试信任配置。

双向 TLS 是客户端和服务端都提供证书供对方验证。启用双向 TLS 时，还要检查服务端是否拒绝了客户端证书。

## URLSession 的职责与请求流程

URLSession 是 Foundation 提供的网络会话对象。URLSession 按配置创建和管理网络任务，负责建立连接、传输数据、接收响应，并把结果交给应用。URLRequest 描述请求地址、方法、请求头和请求体；URLSessionTask 表示一次具体的网络任务。多个请求可以复用同一个 URLSession，不必每次请求都新建。

以 dataTask 为例：应用构造 URLRequest → session 创建尚未启动的任务 → 应用调用 resume() → 系统执行网络传输 → 收到响应并取得数据 → URLSession 调用应用传入的完成回调。网络传输异步进行，调用线程在等待期间可以继续执行其他代码。缓存命中等情况下，URLSession 可能不进行实际的网络传输。

下面的函数接收调用方提供的 URL，并分别判断任务错误、HTTP 状态和响应数据：

```swift
func fetchData(from url: URL) {
    let task = URLSession.shared.dataTask(with: url) { data, response, error in
        if let error {
            print("任务失败：", error)
            return
        }
        guard let response = response as? HTTPURLResponse else {
            return
        }
        guard (200..<300).contains(response.statusCode) else {
            print("HTTP 状态：", response.statusCode)
            return
        }
        guard let data else { return }
        print("收到字节数：", data.count)
    }
    task.resume()
}
```

完成回调的 data 是响应内容字节，response 包含响应信息，error 表示传输或任务执行错误。HTTPURLResponse 是 HTTP 响应信息对象，statusCode 表示服务器返回的 HTTP 状态码。上例把 2xx 作为成功范围，具体接口还可以要求更严格的状态或内容条件。

error 表示传输或任务层面的错误，例如断网、超时。服务器完整返回 404 时，客户端已经成功完成连接、发送请求和接收响应，所以 error 可以是 nil。404 表示服务器没有找到请求的资源，属于 HTTP 层面的失败。因此判断请求是否成功，除了检查 error == nil，还要检查状态码。404 响应中的 data 可能是一段错误说明，应用不应把这段 data 当作图片或成功的 JSON 解析。

完成回调默认不保证在主线程执行。更新 UIKit 界面的代码要放到主线程执行。用 data 创建 UIImage 时，像素解码不一定立即完成，见 [iOS 图片加载、解码与内存](../ios/image-loading-decoding-and-memory.md)。

## URLSessionConfiguration 与连接条件

URLSessionConfiguration 保存一个会话中所有任务共用的规则，例如网络使用限制、超时和缓存配置。URLSession 在创建时复制一份配置；之后修改原配置对象，已经创建的会话不受影响。所以要先改好配置，再创建 URLSession。

```swift
let configuration = URLSessionConfiguration.default
configuration.allowsCellularAccess = false
configuration.waitsForConnectivity = true
let session = URLSession(configuration: configuration)
```

蜂窝网络是手机的 4G、5G 数据网络。`allowsCellularAccess = false` 禁止这个会话使用蜂窝网络。假设手机当前只有 5G 可用：手机本身能上网，但这个会话找不到允许使用的网络。`waitsForConnectivity = true` 让任务先等待，手机连上可用的 Wi-Fi 后再开始建立连接。完全断网时，任务同样会等待。

`waitsForConnectivity` 只作用于连接建立阶段：服务器返回 404、500 时，任务照常结束；已经建立的连接中断后，这个属性也不会让任务自动重试。普通会话中 `waitsForConnectivity` 为 false 时，任务在没有可用连接时可以直接失败。等待期间，任务被取消或达到资源超时时间，等待也会结束。后台会话总是等待可用连接，忽略这个属性。

## URLSessionDelegate 与重定向决定

URLSessionDelegate 是协议，规定自定义代理对象可以实现哪些会话事件方法。应用创建遵循协议的代理对象，在创建 URLSession 时把代理对象传给 delegate 参数；事件发生后，URLSession 调用代理对象上对应的方法。

URLSessionDelegate 主要处理会话级事件和部分认证事件。URLSessionTaskDelegate 继承 URLSessionDelegate，定义任务级事件，包括 HTTP 重定向。URLSessionDataDelegate 定义数据接收事件，URLSessionDownloadDelegate 定义下载事件。

重定向是服务器要求客户端改为访问另一个地址。例如客户端请求旧头像地址，服务器返回 302 和新地址。下面的代理收到 URLSession 按新地址准备好的请求，并决定继续访问：

```swift
final class RedirectDelegate: NSObject, URLSessionTaskDelegate {
    func urlSession(
        _ session: URLSession,
        task: URLSessionTask,
        willPerformHTTPRedirection response: HTTPURLResponse,
        newRequest request: URLRequest,
        completionHandler: @escaping @Sendable (URLRequest?) -> Void
    ) {
        completionHandler(request)
    }
}

let redirectDelegate = RedirectDelegate()
let session = URLSession(
    configuration: .default,
    delegate: redirectDelegate,
    delegateQueue: nil
)
```

执行顺序是：服务器返回重定向响应 → URLSession 调用代理方法并传入新请求 → 代理调用 completionHandler(request) 表示继续 → URLSession 按新请求访问。传入 nil 表示拒绝本次重定向。这里的 `completionHandler` 是 URLSession 交给代理的函数，代理调用这个函数，把“是否继续重定向”的决定告诉 URLSession。

本例使用普通会话。`delegateQueue: nil` 表示由 URLSession 自己创建一个串行操作队列执行代理回调，这个队列不是主队列。URLSession 强引用代理对象，所以自建会话不再使用时要主动结束会话，否则代理对象会一直存活。结束方式按现有任务是否需要完成来选：需要等任务完成时调用 `finishTasksAndInvalidate()`，需要立即取消任务时调用 `invalidateAndCancel()`。

## URLCache 与 HTTP 缓存规则

URLCache 以请求为依据保存和查找 CachedURLResponse。CachedURLResponse 包含响应信息和实际响应字节。对于 JPEG 图片，缓存的字节通常仍是 JPEG 格式；把数据解码为 UIImage 由应用负责。URLCache 判断请求能否命中缓存时，除了比较 URL，还要遵守 HTTP 的匹配规则。

```swift
let cache = URLCache(
    memoryCapacity: 10 * 1024 * 1024,
    diskCapacity: 50 * 1024 * 1024,
    directory: nil
)

let configuration = URLSessionConfiguration.default
configuration.urlCache = cache
configuration.requestCachePolicy = .useProtocolCachePolicy
let session = URLSession(configuration: configuration)
```

memoryCapacity 和 diskCapacity 的单位是字节，分别设置内存和磁盘缓存容量；本例为 10 MiB 和 50 MiB。1 MiB = 1,048,576 字节。`directory: nil` 使用默认缓存目录。内存缓存保存响应对象及其实际字节；磁盘缓存把响应对象和字节保存为文件。缓存条目可能被淘汰，所以即使配置了容量，某条记录也可能被移除。

`requestCachePolicy` 是会话中请求的默认缓存策略。`useProtocolCachePolicy` 是一个枚举值，表示采用网络协议（例如 HTTP）规定的缓存规则；这里的“协议”指网络协议，不是 Swift 中定义方法要求的 protocol。对于 HTTP 和 HTTPS，`useProtocolCachePolicy` 结合请求、响应头和已有缓存判断：没有匹配缓存时访问服务器；缓存仍新鲜且规则允许时直接复用；过期或要求验证时联系服务器，确认能否继续使用旧内容，必要时重新获取。

Cache-Control 是服务器在响应头中提供的缓存指令。新鲜期是响应在满足其他条件时可以直接复用的时间范围。以下以普通 GET 请求为例，并假设应用没有另外禁止存储：

| 响应条件 | 是否允许保存 | 再次使用的条件 |
| --- | --- | --- |
| 200，Cache-Control: max-age=60 | 允许 | 在 60 秒的新鲜期内，满足其他规则时可直接复用 |
| 200，Cache-Control: no-store | 不允许 | 不应保存这份响应用于后续请求 |
| 200，Cache-Control: no-cache | 允许 | 复用前必须联系服务器并成功验证 |
| 404，带有允许缓存的规则 | 也可能允许 | 可以复用“资源不存在”的响应，仍需遵守其有效期等条件 |
| 没有明确的缓存时间 | 不能仅据此判断 | 系统可能根据其他响应信息决定缓存和有效期 |

例如，应用第一次请求头像后，URLCache 保存了允许缓存的响应。在 max-age=60 规定的 60 秒新鲜期内再次发出匹配的请求，可以直接复用缓存，省去网络访问。60 秒后缓存文件可能仍然存在，下次请求时系统会重新验证或重新下载。

no-cache 允许保存响应，但复用前必须向服务器验证；no-store 禁止保存响应。验证时，客户端可以携带服务器此前提供的版本标识，例如 ETag。ETag 是服务器用于标识资源版本的响应字段。服务器确认内容未变时返回 304，客户端继续使用已经缓存的正文，不必重新传输整张图片。没有合适的验证信息时，客户端可能需要重新获取完整响应。

响应允许缓存时，系统可能保存也可能不保存这份响应，下次请求也可能不读取缓存。最终结果由请求方法、响应状态、缓存指令、应用策略和实际容量共同决定：状态码 200 的响应可能不可缓存，404 的响应也可能被缓存。URLCache 与 SDWebImage 自有图片缓存的职责区别见 [SDWebImage 的图片加载与两级缓存](../ios/image-loading-decoding-and-memory.md#sdwebimage-的图片加载与两级缓存)。

## Alamofire：请求与响应模型

Alamofire 是 Swift 网络请求库，适用于获取用户资料、提交表单和上传文件。调用方提供请求地址、参数和结果闭包；库负责发送请求，并提供响应校验和数据转换接口。请求基础见 [URLSession 的职责与请求流程](#urlsession-的职责与请求流程)。

下面假设服务器返回 JSON 对象 `{"id": 1, "name": "小明"}`。`Decodable` 协议表示类型可以从编码数据中还原实例。示例函数接收实际接口地址，调用前需要将 Alamofire 添加到 App 目标。

```swift
import Foundation
import Alamofire

struct User: Decodable {
    let id: Int
    let name: String
}

func loadUser(from url: URL) {
    AF.request(url, method: .get, headers: ["Accept": "application/json"])
        .validate(statusCode: 200..<300)
        .responseDecodable(of: User.self) { response in
            switch response.result {
            case .success(let user):
                print("用户名：\(user.name)")
            case .failure(let error):
                print("请求失败：\(error.localizedDescription)")
            }
        }
}
```

执行顺序是：发起 GET 请求 → 收到响应 → 校验状态码 → 将响应数据转换为 User → 调用结果闭包。`validate(statusCode:)` 接受 200 到 299 的状态码；`responseDecodable` 默认使用 JSONDecoder，字段结构和类型必须符合 User 的声明。网络请求、状态码校验或数据转换失败时，结果进入 failure 分支。

请求异步执行：`loadUser` 返回时，请求可能还没有完成。依赖 user 的代码要放在 `.success` 分支中执行，或由 `.success` 分支触发。服务器返回示例响应时，成功分支打印“用户名：小明”。HTTP 成功和解码成功只说明状态码和数据格式符合要求；接口约定的业务条件是否满足，还要另外检查。

### 响应容器、Result 与模型值

以前例的 `User` 为目标类型，Alamofire 传给结果闭包的 `response` 是响应结果容器。容器同时保存请求、HTTP 响应信息、正文数据和最终结果。`User` 值在 `response.result` 的 `.success` 中，需要先从容器中取出。

| 表达式或变量 | 类型 | 含义 |
| --- | --- | --- |
| `response` | `AFDataResponse<User>` | 响应结果容器 |
| `response.result` | `Result<User, AFError>` | 成功时携带 `User`，失败时携带错误 |
| `.success(let user)` 中的 `user` | `User` | 已经解码得到的模型值 |
| `.failure(let error)` 中的 `error` | `AFError` | Alamofire 定义的错误值 |

`AFDataResponse<User>` 是 `DataResponse<User, AFError>` 的类型别名，即同一种类型的另一种名称。`AFError` 是 Alamofire 用来表示请求、校验或响应数据转换等错误的类型。

成功路径中，Alamofire 先通过解码器得到 `User`，再把模型放入 `.success`，最后把包含这个结果的响应容器传入闭包。`case .success(let user)` 只取出已经解码好的模型，匹配时不会再次解码。枚举关联值与 `Result` 的含义见 [可选类型、状态表达与并发支持](../language/swift-basics.md#可选类型状态表达与并发支持)。

`response.result` 中的 `result` 是容器的属性名，值的类型是 `Result<User, AFError>`。`.success(let result)` 中的 `result` 是调用方声明的局部常量名，值的类型是 `User`。两个 `result` 同名，但不是同一个值；把局部常量命名为 `user` 或 `apiResponse`，更能看出取出的是模型。

### 请求信息、HTTP 响应与正文

下面假设前例请求的地址为 `https://example.com/profile`，方法为 `GET`，并设置请求头 `Accept: application/json`。URL 指定目标地址；请求方法表达本次操作；请求头是一组带名称的字段，描述请求本身以及客户端能接受的响应内容。这里的 `Accept` 表示客户端接受 JSON 格式的响应正文。这个 GET 请求没有请求正文；需要提交数据的请求把数据放在请求正文中。

假设服务器返回下面的内容。为便于阅读，这里使用 HTTP/1.1 的文本形式，省略与示例无关的响应头：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":1,"name":"小明"}
```

HTTP 状态码说明这次 HTTP 请求的结果。响应头是描述响应及其正文的一组字段，例如 `Content-Type: application/json` 声明正文采用 JSON 格式。响应正文承载实际返回的数据，在本例中是用户资料的 JSON 文本。基础协议概念见 [HTTP 请求方法、安全性与幂等性](#http-请求方法安全性与幂等性)。

| 容器属性 | 类型 | 本例保存的内容 |
| --- | --- | --- |
| `response.request` | `URLRequest?` | 请求地址、GET 方法、Accept 请求头等请求信息 |
| `response.response` | `HTTPURLResponse?` | 状态码 200、Content-Type 响应头等响应信息 |
| `response.data` | `Data?` | `{"id":1,"name":"小明"}` 对应的正文字节 |

`response.response` 中，外层 `response` 是闭包参数名，后面的 `.response` 是容器属性。`HTTPURLResponse` 保存状态码和响应头等信息，正文另存于 `data`。这三个属性都是可选值，请求失败时可能为 `nil`。例如，客户端还没收到 HTTP 响应就发生错误时，`response.response` 为 `nil`。

`response.data` 保存尚未转换为 `User` 的原始正文字节，不包含 HTTP 状态码和响应头。解码成功后，容器中同时有 `data` 中的正文字节和 `.success` 中的 `User` 值。

以下代码可放在前例的结果闭包中，用于查看不同层次的信息：

```swift
print(response.request?.url as Any)
print(response.request?.httpMethod as Any)
print(response.request?.value(forHTTPHeaderField: "Accept") as Any)

print(response.response?.statusCode as Any)
print(response.response?.value(forHTTPHeaderField: "Content-Type") as Any)

if let data = response.data,
   let text = String(data: data, encoding: .utf8) {
    print(text)
}
```

`value(forHTTPHeaderField:)` 按字段名获取对应的头字段值；本例读取 `Content-Type` 时得到 `application/json`。示例通过 `if let` 取出正文字符串再打印；UTF-8 解码、JSON 排版和可选字符串的输出差异见 [Data、字符串解码与打印](../language/swift-basics.md#data字符串解码与打印)。

依赖接入见 [Swift 第三方库与依赖接入](../engineering/xcode-and-testing.md#swift-第三方库与依赖接入)。
