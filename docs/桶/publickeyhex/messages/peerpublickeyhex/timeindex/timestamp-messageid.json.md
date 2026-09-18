# `桶/<owner 公钥>/messages/<对端公钥>/timeindex/<时间戳>-<message_id>.json` — 本地时间索引

每个会话一份**只增的本地时间索引**：记录本机在这条消息上观察到的时间，
以及它对应的 raw 文件。列出目录就是时间线，不需要读取 raw。

## 位置与命名

```
桶根/<owner 公钥>/messages/<对端公钥>/timeindex/<时间戳>-<message_id>.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥，小写 |
| `<对端公钥>` | 会话对端压缩公钥，小写 |
| `<时间戳>` | 13 位补零毫秒 Unix 时间，UTC；本地时间 |
| `<message_id>` | 32 字节 base64url 无填充（43 字符） |

- 时间戳补零到 13 位，保证按文件名字典序排列就是时间顺序。
- 文件名同时带 `message_id`：同一毫秒的两条消息不会互相覆盖，也方便去重。
- 时间与 `message_id` 在文件内重复一次，读取时不信任文件名，做交叉校验。

## 文件格式

```jsonc
{
  "format": "keymaster.message-index",
  "version": 1,
  "kind": "sent" | "received",
  "timestamp": 1757900000000,
  "rawHash": "…………64 位小写 hex",
  "messageId": "…………43 位 base64url"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.message-index"` |
| `version` | 数字 | 是 | 固定 `1` |
| `kind` | 枚举 | 是 | `sent` = 出站（raw 在 `sent/`）；`received` = 入站（raw 在 `received/`） |
| `timestamp` | 数字 | 是 | 本机观察时间，毫秒；见下 |
| `rawHash` | 字符串 | 是 | 对应 raw 文件名的 MasterSeed `seed_hash` |
| `messageId` | 字符串 | 是 | 消息的 `message_id`；与文件名一致 |

`timestamp` 的语义只取本机时间，不使用对方在消息里声明的 `issued_at_ms`：

- `kind = "sent"`：发布被本地接受的时间；
- `kind = "received"`：本机收到该消息的时间。

## 规则

- **写入时机**：先写 raw，再追加本文件。崩溃只会留下"有 raw、无 index"。
- **只增**：每次到达（包括重复投递）都追加一条；不修改、不删除、不合并。
- **不再重建**：索引是"我收到/我发出"的本地证据，不接受从 raw 反向重建，也不因 raw 缺失而删除。
- **raw 缺失**：索引保留，内容显示"缺失"；不补写、不扫描。
- **读取去重**：同一条消息可能有多条索引（重复投递、多设备各自观察）；按 `messageId` 去重，
  取最早 `timestamp` 作为"首次收到"。
- **重复投递**：raw 只保留一份（内容寻址），索引仍追加新的一条本地时间记录。
- **不跨文件校验**：一条坏索引跳过并提示，不影响其它条目和 raw。
- **大小**：单文件不超过 1 KiB。

## 校验规则

1. 文件名匹配 `^\d{13}-[A-Za-z0-9_-]{43}\.json$`。
2. `format` / `version` 匹配；字段白名单匹配。
3. `timestamp` 为 13 位非负整数，且与文件名前缀一致。
4. `messageId` 与文件名中的 `message_id` 一致。
5. `rawHash` 匹配 `^[0-9a-f]{64}$`；对应文件存在时重算校验（缺失不算错误，按"缺失"展示）。

## 设计取舍（消融记录）

- **索引是证据，不是缓存**：收到时间无法从 raw 反推，所以不重建、不删除、不清理旧条目。
- **一条一文件**：避免 head 式聚合写；并发只受"同一条"影响，重复写同内容幂等。
- **时间放文件名**：列目录即排序，读取时不用先把所有 raw 读出来排序。
- **`kind` 区分方向**：出站和入站共用一条会话时间线；删除、备份仍以对端目录为单位。
- **`message_id` 进文件名**：同毫秒不覆盖，并承担去重主键；时间戳仍是排序主键。
- **不存供应商标识和频道**：它们属于链路诊断，不是消息证据；需要时另行记录。

## 完整示例

路径（`<owner 公钥>` = `027b2e4d5a…`，`<对端公钥>` = `023f8a1c9d…`）：

```
桶/027b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a/messages/023f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d/timeindex/1757900000000-ERERERERERERERERERERERERERERERERERERERERERE.json
```

内容（字段值仅示例）：

```json
{
  "format": "keymaster.message-index",
  "version": 1,
  "kind": "received",
  "timestamp": 1757900000000,
  "rawHash": "3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d3f8a1c9d",
  "messageId": "ERERERERERERERERERERERERERERERERERERERERERE"
}
```
