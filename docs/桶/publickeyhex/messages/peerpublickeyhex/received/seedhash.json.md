# `桶/<owner 公钥>/messages/<对端公钥>/received/<seedhash>.json` — 入站加密信封

收到的私密消息的**加密信封精确字节**：线上原始字节，落桶时保持密文；只有 owner 私钥能解密，
解密并验签通过后才写入本目录。

## 位置与命名

```
桶根/<owner 公钥>/messages/<对端公钥>/received/<seedhash>.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥，小写；消息的接收方 |
| `<对端公钥>` | 验签确认后的发件人压缩公钥，小写 |
| `<seedhash>` | 文件内容的 MasterSeed `seed_hash`，64 位小写 hex |

- 文件名不是 `message_id`，而是内容哈希；同一条消息被重复投递（重广播）时内容相同，
  写的是同一个文件，重复写入是幂等覆盖。
- 文件名不带时间；时间写在 [timeindex](../timeindex/timestamp-messageid.json.md) 里。

## 文件内容

文件内容就是收到的加密信封规范 JCS JSON 字节，**不加任何头、包装或额外字段**：

```jsonc
{
  "envelope_version": 1,
  "from_public_key": "02……",   // 发件人自称的压缩公钥
  "kdf_salt": "…………",          // 32 字节 base64url
  "nonce": "…………",             // 12 字节 base64url
  "ciphertext": "…………"         // 密文 + 16 字节 GCM tag，base64url
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `envelope_version` | 数字 | 是 | 固定 `1` |
| `from_public_key` | 字符串 | 是 | 发件人压缩公钥；解密验签后再与目录名核对 |
| `kdf_salt` | 字符串 | 是 | 32 字节 base64url，每条消息新的 HKDF salt |
| `nonce` | 字符串 | 是 | 12 字节 base64url，每条消息新的 AES-GCM nonce |
| `ciphertext` | 字符串 | 是 | 密文；`protocol` / `message_id` / 时间 / 正文 / 签名都在里面 |

注意：文件内**不含** `channel`。解密时频道由路径推导：`channel` = `bsv8.inbox.<owner 公钥>`。

## 规则

- **写入时机**：收到后先用 owner 私钥在 `bsv8.inbox.<owner 公钥>` 上解密并验签；全部通过，
  且信封 `from_public_key` 等于 `<对端公钥>`，才写入本目录。
- **归属失败**：解不开、验签失败、`from_public_key` 与目录不一致、频道不属于当前 owner inbox，
  **直接丢弃，不写入桶内任何位置**；没有对端就不产生文件。
- **幂等**：同一信封字节的 seed_hash 相同，重复投递写同一个文件；不使用条件写。
- **不修改、不删除**：见模块 [README](../../README.md) 的只增规则。
- **每次到达都记时间**：raw 只保留一份，重复投递仍会在 timeindex 追加一条本地时间记录。
- **读取**：用 owner 私钥解密并验签后展示；raw 缺失时条目保留，显示"缺失"；读取失败按损坏处理并提示。
- **大小**：单文件不超过 1 MiB（SSP Wire 上限）。

## 校验规则

1. 文件名匹配 `^[0-9a-f]{64}\.json$`。
2. 文件内容的 MasterSeed `seed_hash` 等于文件名；不等按损坏处理。
3. JSON 可解析，字段白名单匹配，base64url 长度与类型正确。
4. 用 `channel = bsv8.inbox.<owner 公钥>` 解密成功，且信封 `from_public_key` 等于目录名。
5. 解密后验签通过，时间字段在有效期内；否则按无效消息处理。

## 设计取舍（消融记录）

- **只存密文**：第三方拿到桶看不到正文；解密所需的 owner 私钥不落桶。
- **文件名用内容哈希**：重复投递天然幂等；不因重广播产生多份内容。
- **raw 与 index 分离**：内容完整性由 seed_hash 保证，时间顺序由 timeindex 保证；
  索引不参与内容校验，raw 缺失不影响索引作为"收到过"的证据。
- **不用 message_id 命名 raw**：`message_id` 在密文里，必须先解密才能拿到；用内容哈希可先落盘，
  也不暴露消息身份。
- **私钥丢失即不可读**：入站密文的可读性依赖 owner 私钥；这是密文静止的代价，出站签名明文不受影响。

## 完整示例

路径（`<owner 公钥>` = `027b2e4d5a…`，`<对端公钥>` = `023f8a1c9d…`）：

```
桶/027b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a/messages/023f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d/received/3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d.json
```

内容（字段值仅示例）：

```json
{
  "envelope_version": 1,
  "from_public_key": "023f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d",
  "kdf_salt": "IiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiI",
  "nonce": "MzMzMzMzMzMzMzMz",
  "ciphertext": "REREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREREQ"
}
```
