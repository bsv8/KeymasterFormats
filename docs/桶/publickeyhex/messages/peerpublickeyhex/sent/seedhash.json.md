# `桶/<owner 公钥>/messages/<对端公钥>/sent/<seedhash>.json` — 出站签名明文

自己发出的私密消息的**签名明文精确字节**：作者证据，明文可读、可用 owner 公钥验签，
不依赖解密，也在 owner 私钥丢失后仍可阅读。

## 位置与命名

```
桶根/<owner 公钥>/messages/<对端公钥>/sent/<seedhash>.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥，小写；消息的作者 |
| `<对端公钥>` | 收件人压缩公钥，小写；消息的接收方 |
| `<seedhash>` | 文件内容的 MasterSeed `seed_hash`，64 位小写 hex |

- 文件名不是 `message_id`，而是内容哈希：同名文件必然同内容；重复发送同一消息是幂等覆盖。
- 文件名不带时间；时间是本机观察，写在
  [timeindex](../timeindex/timestamp-messageid.json.md) 里。

## 文件内容

文件内容就是 ChannelProtocol 已签名私密消息的规范 JCS JSON 字节，**不加任何头、包装或额外字段**：

```jsonc
{
  "protocol": "bsv8.message.v1",
  "message_id": "…………43 位 base64url",
  "issued_at_ms": 1757900000000,
  "expires_at_ms": 1757986400000,
  "body": { "type": "deliver", "content": { "type": "text", "contentType": "text/plain", "body": "你好", "clientMessageId": "……", "createdAtMs": 1757900000000 } },
  "signature": "…………"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `protocol` | 字符串 | 是 | 子协议名；普通私信为 `bsv8.message.v1` |
| `message_id` | 字符串 | 是 | 32 字节 base64url 无填充；消息身份 |
| `issued_at_ms` / `expires_at_ms` | 数字 | 是 | 作者声明的签发/过期时间；过期上限由子协议决定 |
| `body` | JSON | 是 | 子协议正文；应用消息为 `deliver` / `ack` |
| `signature` | 字符串 | 是 | 对 `(channel, from_public_key, protocol, message_id, issued_at_ms, expires_at_ms, body)` 的签名 |

注意：文件内**不含** `channel` 和 `from_public_key`（ChannelProtocol 规范如此）。
校验时由路径补全：

- `channel` = `bsv8.inbox.<对端公钥>`；
- `from_public_key` = `<owner 公钥>`。

## 规则

- **写入时机**：发布被本地接受后写入；不写入仅尝试发送、尚未被接受的消息。
- **幂等**：同一签名明文的 seed_hash 相同，重复写等价于覆盖同内容；不使用条件写。
- **不修改**：文件一旦写入不再改写（内容寻址保证同名同内容）。
- **不删除**：见模块 [README](../../README.md) 的只增规则。
- **读取**：解析 JSON，按路径补出 `channel` / `from_public_key` 后用 owner 公钥验签；
  验签失败按损坏文件跳过并提示，不展示正文。
- **大小**：单文件不超过 1 MiB（SSP Wire 上限）。

## 校验规则

1. 文件名匹配 `^[0-9a-f]{64}\.json$`。
2. 文件内容的 MasterSeed `seed_hash` 等于文件名；不等按损坏处理。
3. JSON 可解析且字段白名单匹配；`protocol` 与 `body` 按子协议校验。
4. 用路径推导出的 `channel` / `from_public_key` 验签通过。
5. 时间字段为非负整数且 `issued_at_ms <= expires_at_ms`。

## 设计取舍（消融记录）

- **存签名明文，不存发布信封**：owner 用 `open()` 解不开自己发出的信封（它要求私钥等于频道里的
  收件人）；签名明文无需解密即可读、可验签。发布信封与它只差随机 salt/nonce 和密文，不构成内容证据。
- **文件名用内容哈希**：天然去重、跨设备幂等，不需要 CAS，也不会因时间戳不同而重复存同一内容。
- **不存应用投影**：正文、对端、时间都不额外落盘；能由签名明文和路径重建。
- **明文静止**：出站消息在桶里是明文，换取"私钥丢失后仍可阅读和验证"；入站消息仍是密文。

## 完整示例

路径（`<owner 公钥>` = `027b2e4d5a…`，`<对端公钥>` = `023f8a1c9d…`）：

```
桶/027b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a/messages/023f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d/sent/9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a.json
```

内容（字段值仅示例）：

```json
{
  "protocol": "bsv8.message.v1",
  "message_id": "ERERERERERERERERERERERERERERERERERERERERERE",
  "issued_at_ms": 1757900000000,
  "expires_at_ms": 1757986400000,
  "body": { "type": "deliver", "content": { "type": "text", "contentType": "text/plain", "body": "你好", "clientMessageId": "km-msg-001", "createdAtMs": 1757900000000 } },
  "signature": "VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV"
}
```
