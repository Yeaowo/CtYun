# 天翼云电脑（CtYun）客户端保活项目分析报告

## 1. 项目简介

**CtYun** 是一个使用 C# 编写的控制台应用程序，主要用于自动登录天翼云电脑（或云手机）并建立 WebSocket 连接，实现设备的保活（Keep-Alive）功能。
该项目针对天翼云桌面进行协议级模拟，无需启动庞大的官方客户端，便可通过控制台程序（或 Docker 容器）在后台自动维护设备的活跃状态。

此项目主要包含以下几个核心模块：
- **用户认证与设备绑定：** 通过 HTTP API 进行登录、获取验证码、调用第三方 OCR API（`ddddocr` 包装）自动识别验证码。
- **设备列表与状态获取：** 请求云端接口拉取名下的云电脑/云手机列表，并触发开机/连接指令。
- **WebSocket 协议保活：** 模拟原生客户端的 WebSocket 握手与心跳机制（含加密报文）确保会话存活。
- **加密处理：** 包含自定义的 RSA-OAEP / MGF1 及 MD5、SHA256 等散列加密逻辑，应对保活过程中的“挑战-响应”校验。

---

## 2. 整体架构与运行流程

整个程序的入口点在 `Program.cs`，代码采用了现代 C# 顶层语句（Top-level statements）风格编写。整个执行流程可以分为四大阶段：

### 2.1 凭证获取与初始化 (Initialization)
1. **环境变量/交互式输入：** 程序首先通过 `ResolveCredentials` 获取用户的 `账号(APP_USER)`、`密码(APP_PASSWORD)` 和 `设备ID(DEVICECODE)`。优先从环境变量中读取（Docker 环境），若无则采用控制台交互式输入。
2. **API 客户端初始化：** 实例化 `CtYunApi`，在其构造函数中初始化 `HttpClient`，并设置模拟浏览器与天翼云客户端特有的 Headers（如 `ctg-devicetype`、`ctg-version`、`ctg-devicecode`）。

### 2.2 登录认证阶段 (Authentication Sequence)
在 `PerformLoginSequence` 中：
1. **获取挑战码：** 请求 `genChallengeData` 接口获取登录时所需的 `challengeId` 和 `challengeCode`。
2. **自动识别验证码：** 请求天翼云的验证码图片 API（`captcha`），并将其 Base64 传递给第三方 OCR 服务（`orc.1999111.xyz/ocr`），解析出文本。
3. **密码加密与登录请求：**
   - 客户端对密码进行了两层 SHA256 加密。结合 `challengeCode` 再次 Hash：
     - `password` 参数 = `SHA256(明文密码 + challengeCode)`
     - `sha256Password` 参数 = `SHA256(SHA256(明文密码) + challengeCode)`
   - 将账号、密码密文、验证码等作为表单提交至 `/api/auth/client/login` 完成登录。
4. **风控校验（设备绑定）：** 如果账号首次在新设备登录，服务端返回的 `LoginInfo.BondedDevice` 会为 `false`。此时客户端会再次请求短信验证码，并提示用户手动输入完成设备绑定 (`BindingDeviceAsync`)。

### 2.3 获取设备并触发连接 (Device Discovery & Connection)
1. **获取桌面列表：** 调用 `GetLlientListAsync` 从 `/api/desktop/client/pageDesktop` 接口拉取用户下的云桌面与云手机列表。
2. **设备开机与连接凭证获取：** 遍历设备列表，对于每台设备调用 `ConnectAsync`。如果设备未开机，云端会自动下发开机指令。若连接成功，会返回 `DesktopInfo`，里面包含了连接 WebSocket 服务器所需的域名、证书以及 Token 等信息。

### 2.4 WebSocket 保活循环 (Keep-Alive Worker)
程序为每台存活的设备分配了一个独立的异步任务 `KeepAliveWorkerWithForcedReset`。这里的保活逻辑被设计为“每 60 秒强制重连一次”的短周期轮询：

1. **建立 WebSocket 连接：** 使用 `ClientWebSocket` 连接到设备的专有代理节点（如 `wss://{ClinkLvsOutHost}/clinkProxy/{DesktopId}/MAIN`），需要设置子协议 `binary`。
2. **握手与初始化报文：**
   - 先发送一段 **JSON 格式的握手消息** (`ConnecMessage`)，内含 SSL 参数、客户端证书、密钥等，完成鉴权。
   - 等待 500ms，发送预设的固化二进制报头 (`initialPayload`: `REDQ...`)。
3. **心跳与加密挑战响应 (`ReceiveLoop`)：**
   - 在 WebSocket 保持打开期间，程序监听服务端发来的二进制报文。
   - **保活校验报文 (`REDQ`)：** 如果收到以 `REDQ` (十六进制 `52454451`) 开头的数据，程序将其传入 `Encryption.Execute()` 进行解密/加密计算（基于 RSA-OAEP 和 MGF1 掩码）。生成加密签名后回复给服务端，完成该轮的活跃状态挑战。
   - **CLINK 协议指令：** 程序还将二进制报文解析为自定义的 `SendInfo` 结构体：
     - `Type == 103` (CLINK_MSG_MAIN_INIT)：程序会发送用户的基础信息 (`Type == 118`)。
     - `Type == 4`：通常属于 PING/PONG 机制（在现有代码中部分被注释掉了）。
4. **重连机制：** 当前连接超过 60 秒（由 `sessionCts.CancelAfter` 触发）后，取消当前任务，关闭 WebSocket。由于外层的 `while (!globalToken.IsCancellationRequested)`，程序会立刻再次执行以上流程，实现了持续不间断地重连保活。

---

## 3. 核心机制解析

### 3.1 协议报文结构 (`SendInfo.cs`)
自定义的二进制通讯报文结构非常简单：
- `Type` (2 Byte, ushort)：消息类型。
- `Size` (4 Byte, int)：数据长度。
- `Data` (N Byte)：消息体（Payload）。
在实际组装中，有时还会插入 `msgLength` 头字段（由 `isBuildMsg` 控制），体现了典型的 `Type-Length-Value` (TLV) 设计模式。

### 3.2 加密签名机制 (`Encryption.cs`)
天翼云电脑的保活协议为了防止恶意刷号，在 WebSocket 中加入了基于非对称加密的“挑战-响应”验证：
- 当接收到服务端下发的 `REDQ` 校验流时，客户端需要从流中截取公钥 N 和指数 E。
- 使用标准的 `RSA-OAEP`（最佳非对称加密填充）逻辑。
- 其中手动实现了掩码生成函数 `MGF1`（基于 SHA-1）。
- 最终利用 `BigInteger.ModPow` 进行大数模幂运算完成 RSA 的加密。
- 将加密结果添加 `AuthMechanism` 标志后封包回复。
（*注：这里的 RSA 实现采用纯 C# 数组与大数运算手写完成，而未使用 .NET 自带的 `RSA.Create()`，可能是为了兼容特定的字节序或非常规的 Padding 格式。*）

### 3.3 HTTP 接口签名 (`ctg-signaturestr`)
为了防御重放攻击，所有 API 请求 (GetAsync, PostAsync) 都会调用 `ApplySignature` 生成请求头签名：
- 提取变量：`设备类型(deviceType)`、`时间戳(timestamp)`、`租户ID(tenantId)`、`用户ID(userId)`、`客户端版本(version)` 以及登录返回的 `secretKey`。
- 拼接字符串：`{deviceType}{timestamp}{tenantId}{timestamp}{userId}{version}{secretKey}`。
- 将字符串进行 MD5 计算后，转为十六进制小写作为签名 `ctg-signaturestr`。

---

## 4. 安全与维护建议

对于开发和二次维护此项目的开发人员，有以下几点需要注意：

1. **强依赖第三方 OCR：** 登录流程严重依赖于开源的 ddddocr 搭建的接口 (`orc.1999111.xyz/ocr`)。如果该公共 API 宕机或改变接口结构，客户端将无法自动登录。**建议：** 支持本地部署 OCR 容器，或将 OCR API 地址放入配置文件中。
2. **明文保存凭据：** 设备码 `DeviceCode.txt` 虽为明文但不敏感。如果在 Docker 中运行并注入 `APP_PASSWORD`，需要防范环境变量泄漏带来的安全风险。
3. **强制 60 秒重连：** 当前代码中的 `sessionCts.CancelAfter(TimeSpan.FromMinutes(60))` 策略略显粗暴（无论是否健康均强制断开重连）。虽然能有效防范假死，但如果设备过多，频繁重新建立 TLS 与 WSS 连接会带来额外的网络开销和可被溯源的特征。
4. **硬编码的参数：** API 版本号 `103020001`，App 版本 `3.2.0` 等均写死在代码中。如果官方强制升级客户端，可能需要同步抓包更新这些参数才能继续使用。

## 5. 总结
CtYun 项目是一个极简但非常高效的天翼云控制台客户端。作者通过逆向官方 API，成功还原了复杂的登录密码散列、HTTP 头签名认证、以及 WebSocket 保活心跳中的自定义 RSA-OAEP 加密校验算法。整体代码逻辑清晰，依赖极少，非常适合在无头环境（Linux/Docker）下作为守护进程运行。