# TCP Framing · TCP 帧协议

[首页](../README.md#zh) · [协议目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

WhatsApp 的 TCP 长连接使用应用层长度前缀划分帧：**前 3 字节是大端无符号长度，随后紧跟指定长度的二进制内容**。握手完成后，接收流程为“按长度取帧 → 认证解密 → 处理标志与可选压缩 → Protocol Tree Node 解析”。

### 帧结构

```text
| b0 | b1 | b2 | frame_body：L 字节 |

L = ((b0 & 0xFF) << 16)
  | ((b1 & 0xFF) << 8)
  |  (b2 & 0xFF)
完整帧长度 = 3 + L
```

三个字节全部用于长度，不携带消息类型或压缩标志。`& 0xFF` 按无符号字节取值，避免 Java 等语言的有符号 `byte` 影响计算。字段可表示 `0～0xFFFFFF`，即最多 16,777,215 字节；实际处理还受内容规则与缓冲限制约束。

握手后的帧体包含密文和 **16 字节 GCM 标签**，二者都计入 `L`，三字节头本身不计入。例如：

```text
00 01 20 | 密文 272 字节 | GCM 标签 16 字节
L = 0x120 = 272 + 16 = 288，完整帧共 291 字节
```

### 解密与节点解析

使用当前方向的 Noise 传输密钥和计数器，对完整帧体做 AES-256-GCM 认证解密，得到明文包 `P`：

```text
未压缩：P = 00 || binary_node
已压缩：P = 02 || zlib(binary_node)
L = len(P) + 16
```

首字节是加密明文内部的 `flags`。`flags & 0x02` 非零时，对它后面的数据做 zlib 解压，得到节点字节；压缩格式是带 zlib 包装的 DEFLATE。`flags & 0x01` 表示节点分片，服务器到客户端的节点读取路径会拒绝该形式。

[Protocol Tree Node](protocol-tree-node.md#zh) 将节点字节中的列表、token 字典、属性、JID 和子节点还原为树结构；XML 是树的可读展示形式。节点内的 `enc` 等载荷仍可能经过 Signal 端到端加密。

```text
接收：完整帧 → Noise 认证解密 → flags / 可选 zlib → 节点解析 → 业务分发
发送：Node → 二进制编码 → flags / 可选 zlib → Noise 加密 → 长度头 + 密文及标签
```

### TCP 收帧与异常

TCP 是字节流：循环读满 3 字节，再循环读满 `L` 字节。使用接收缓冲时，连续取出完整帧，保留不足一帧的尾部。TCP 拆分或合并传输数据与节点分片标志无关。

帧中途 EOF 表示截断；GCM 认证失败应终止当前加密会话。认证解密得到的空明文会被跳过，只有标志字节的明文会报错。长度为零的帧不能自行解释为心跳。

### 握手与连接前导

握手阶段同样使用三字节长度头，但帧体是 Protobuf 编码的 `HandshakeMessage`，按 Noise 握手状态处理，不进入节点解析器。

客户端首个握手帧前有一次性的 `WA` 连接前导，带路由信息时还会先写入 `ED` 前导。这些前导不计入后面握手帧的 `L`，也不在每帧前重复。具体字节见 [连接封装](noise.md#zh-wire)，nonce 与 AAD 规则见 [Noise 加密状态](noise.md#zh-state)。

### 相关内容

[Noise 握手与加密传输](noise.md#zh) · [Protocol Tree Node · 二进制节点与编解码](protocol-tree-node.md#zh) · [通信协议与数据格式](README.md#zh) · [端到端加密与密钥管理](../encryption/README.md#zh)

<a id="en"></a>

## English

WhatsApp's persistent TCP connection uses application-level length prefixes: **three bytes of unsigned big-endian length followed immediately by that many binary bytes**. After the handshake, receiving proceeds through frame extraction, authenticated decryption, flags and optional decompression, then Protocol Tree Node parsing.

### Frame Layout

```text
| b0 | b1 | b2 | frame_body: L bytes |

L = ((b0 & 0xFF) << 16)
  | ((b1 & 0xFF) << 8)
  |  (b2 & 0xFF)
Total frame size = 3 + L
```

All three bytes encode length, with no message type or compression flag. Masking with `0xFF` treats each byte as unsigned, avoiding signed-byte issues in languages such as Java. The field represents `0–0xFFFFFF`, up to 16,777,215 bytes; content rules and buffer limits further constrain processing.

After the handshake, the body contains ciphertext and a **16-byte GCM tag**. Both count toward `L`; the three-byte prefix does not. For example:

```text
00 01 20 | ciphertext: 272 bytes | GCM tag: 16 bytes
L = 0x120 = 272 + 16 = 288; total frame size = 291 bytes
```

### Decryption and Node Parsing

Authenticate and decrypt the complete body with AES-256-GCM using the current direction's Noise transport key and counter. This yields plaintext packet `P`:

```text
Uncompressed: P = 00 || binary_node
Compressed:   P = 02 || zlib(binary_node)
L = len(P) + 16
```

The first plaintext byte contains `flags`. When `flags & 0x02` is nonzero, zlib-decompress the following bytes to obtain the node bytes, using zlib-wrapped DEFLATE. `flags & 0x01` indicates stanza fragmentation; the server-to-client node-reading path rejects that form.

[Protocol Tree Node](protocol-tree-node.md#en) reconstructs a tree from lists, token dictionaries, attributes, JIDs, and child nodes. XML is a readable representation of that tree. Payloads such as `enc` may still contain Signal end-to-end ciphertext.

```text
Receive: complete frame → Noise authenticated decryption → flags / optional zlib → node parsing → dispatch
Send:    Node → binary encoding → flags / optional zlib → Noise encryption → length + ciphertext and tag
```

### TCP Frame Reading and Errors

TCP is a byte stream: read exactly three bytes in a loop, then exactly `L` body bytes. A buffered receiver extracts complete frames and retains incomplete trailing data. TCP splitting or combining data is independent of the stanza fragmentation flag.

EOF within a frame means truncation; GCM authentication failure terminates the current cryptographic session. Authenticated empty plaintext is skipped, while flags-only plaintext is an error. A zero-length frame must not be assumed to be a heartbeat.

### Handshake and Connection Prefixes

Handshake frames use the same three-byte length prefix, but their body is a Protobuf `HandshakeMessage`. Process it according to Noise handshake state rather than passing it to a node decoder.

The client writes a `WA` connection prefix once, before its first handshake frame. When routing information is supplied, an `ED` prefix precedes it. These prefixes are excluded from the following handshake frame's `L` and do not recur before each frame. See [connection framing](noise.md#en-wire) for their bytes and [Noise cryptographic state](noise.md#en-state) for nonce and AAD rules.

### Related Topics

[Noise Handshakes and Encrypted Transport](noise.md#en) · [Protocol Tree Node and Binary Encoding](protocol-tree-node.md#en) · [Communication Protocols and Data Formats](README.md#en) · [End-to-End Encryption and Key Management](../encryption/README.md#en)
