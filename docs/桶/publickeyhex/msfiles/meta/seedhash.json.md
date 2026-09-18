# `桶/<owner 公钥>/msfiles/meta/<seedhash>.json` — MSFile 文件元数据

一个种子一份源文件描述：显示文件名、媒体类型、源文件大小。MasterSeed 种子字节只包含块摘要，
不含文件名、类型和长度，因此这些信息必须作为旁车元数据单独保存。

## 位置与命名

```
桶根/<owner 公钥>/msfiles/meta/<seedhash>.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥，小写；每个 key 一份目录 |
| `msfiles/meta` | 固定目录 |
| `<seedhash>` | 对应种子的 MasterSeed `seed_hash`，64 位小写 hex |

- 同一内容的三处存储共用同一个 `<seedhash>`：
  - 种子：`msfiles/seeds/<seedhash>.ms`；
  - 块：`msfiles/storage/<seedhash>/<块 hash>`；
  - 元数据：`msfiles/meta/<seedhash>.json`。
- 种子是条目的存在开关：只有 `seeds/<seedhash>.ms` 存在时，对应的 `meta` 才进入列表并参与展示。
- 文件名就是内容身份，不另存 `id`；改内容必然改 `seedhash`。

## 文件格式

```jsonc
{
  "format": "keymaster.msfiles-meta",
  "version": 1,
  "seedHashHex": "9d2e4b6a…",            // 与文件名一致
  "fileName": "holiday.mp4",             // 显示名，只取 basename
  "mediaType": "video/mp4",              // 规范化 MIME；未知用 application/octet-stream
  "fileSizeBytes": "1073741824",         // 规范十进制字符串
  "blockCount": 4096,                    // = ceil(fileSizeBytes / 262144)
  "seedSizeBytes": "131072",             // = blockCount × 32
  "storedAt": "2026-09-19T08:00:00.000Z" // 本机写入元数据的时间
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` / `version` | 字面量 | 是 | 固定 `"keymaster.msfiles-meta"` / `1` |
| `seedHashHex` | 字符串 | 是 | `^[0-9a-f]{64}$`，必须与文件名一致 |
| `fileName` | 字符串 | 是 | 显示名；只允许 basename，1~255 UTF-8 字节；不得含 `/`、`\`、NUL/C0/C1 控制字符，不得为 `.` 或 `..` |
| `mediaType` | 字符串 | 是 | 小写 `type/subtype`；未知或浏览器未提供时用 `application/octet-stream`；1~255 字节 |
| `fileSizeBytes` | 字符串 | 是 | 源文件字节数的规范十进制（`"0"` 或非零开头，无前导零），uint64 |
| `blockCount` | 数字 | 是 | 安全整数；等于按 256 KiB 分块的块数，空文件为 `0` |
| `seedSizeBytes` | 字符串 | 是 | 种子文件字节数的规范十进制，等于 `blockCount × 32` |
| `storedAt` | 字符串 | 是 | ISO-8601 UTC 毫秒；本机写入时间，不参与任何计算 |

单个文件不超过 4 KiB；未知字段拒绝。

## 写入顺序与幂等

1. 先写全部块：`msfiles/storage/<seedhash>/<块 hash>`；同名块内容必然相同，允许直接覆盖。
2. 再写种子：`msfiles/seeds/<seedhash>.ms`；种子内容由源文件唯一决定，重复上传同内容等价于覆盖。
3. 最后写元数据：`msfiles/meta/<seedhash>.json`。它最后出现，保证列表里的条目信息完整。
4. 任意一步中断只留下无害残留：孤儿块或缺元数据的种子；重新上传同内容即可补齐，不需要回滚。

- 同一 `seedhash` 再次上传时可以更新 `fileName`、`mediaType`、`storedAt`；`fileSizeBytes`、
  `blockCount`、`seedSizeBytes` 必须与种子和实际内容一致，不一致按损坏处理。
- 实现不做断点续传；没有临时对象、没有事务，也没有跨文件索引。

## 删除

1. 先删 `seeds/<seedhash>.ms`：条目立即从列表消失。
2. 再删 `meta/<seedhash>.json`。
3. 最后删除 `storage/<seedhash>/` 下的全部块。

中断只留下孤儿元数据或孤儿块；两者都不参与列表，也不会被当作有效文件。

## 列出、缺失与损坏

- 列表以 `seeds/` 前缀为真值来源；同一页对每个 `<seedhash>` 尝试读取 `meta/<seedhash>.json`，
  不为缺失元数据的种子编造文件名或类型。
- `seeds/` 有种子、`meta/` 缺失：条目保留，显示未知文件名/类型，并因缺少 `fileSizeBytes`
  无法可靠组装，不提供下载与预览，只允许校验或删除。
- `meta/` 有文件、`seeds/` 缺失：孤儿元数据，忽略，不展示。
- 元数据自身校验失败（字段缺失、文件名不一致、块数关系不成立等）按损坏处理：跳过该元数据，
  按上一条“缺失”语义展示，不影响其它条目。
- 读取种子或块后重算摘要：块 SHA-256 必须等于块文件名，种子 SHA-256（即 `seed_hash`）必须等于
  `<seedhash>`；不符按损坏处理。

## 大小与并发

- 元数据不是秘密，明文保存；其中没有私钥、凭据或可执行内容。
- 不同 `seedhash` 是不同文件，互不冲突；同一文件在 Worker 内串行，跨设备最后写入者胜
  （见《存储规则》）。
- 种子与块的字节格式、分块和校验规则以 [MasterSeed](https://github.com/bsv8/MasterSeed)
  的 `keymaster-seed-v1` 为准；本文件只描述旁车元数据。

## 设计取舍（消融记录）

- **单独 `meta/` 目录**：列表只需前缀 `seeds/`，不会扫到块对象；元数据与字节分开，
  删元数据不影响内容寻址，删种子不影响元数据文件本身的幂等写入。
- **只存描述，不存摘要清单**：块 hash 可由种子文件解析，不重复保存；元数据只补种子格式
  明确不包含的字段（文件名、类型、源长度、写入时间）。
- **字符串存字节数**：避免 JSON 数字在 2^53 之后失真；与 wire 和金额字段的处理一致。
- **最后写元数据**：它是“列表可完整展示”的标记；先写种子保证内容可用，再写元数据保证展示完整。
- **无索引、无事务、无续传**：与《存储规则》一致；单条失败不影响其它条目，重传即幂等修复。

## 完整示例

路径（`<owner 公钥>` = `027b2e4d5a…`，`<seedhash>` = `9d2e4b6a…`）：

```
桶/027b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a7b2e4d5a/msfiles/meta/9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a.json
```

内容（字段值仅示例）：

```json
{
  "format": "keymaster.msfiles-meta",
  "version": 1,
  "seedHashHex": "9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a9d2e4b6a",
  "fileName": "holiday.mp4",
  "mediaType": "video/mp4",
  "fileSizeBytes": "1073741824",
  "blockCount": 4096,
  "seedSizeBytes": "131072",
  "storedAt": "2026-09-19T08:00:00.000Z"
}
```
