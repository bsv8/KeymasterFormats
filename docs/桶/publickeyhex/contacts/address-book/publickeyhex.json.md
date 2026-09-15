# `桶/<owner 公钥>/contacts/address-book/<联系人公钥>.json` — 联系人文件

一个联系人一个文件,文件名就是联系人公钥(小写压缩格式);按 owner(当前 active key)隔离;明文 JSON。

```
桶根/<owner 公钥>/contacts/address-book/<联系人公钥>.json
```

- `contacts/address-book` 是中央声明的模块/用途;owner 下只允许登记过的模块名。
- 列目录 = 全部联系人;删除 = 删文件。

## 文件格式

```jsonc
{
  "format": "keymaster.contact",
  "version": 1,
  "publicKeyHex": "02ab...",     // 与文件名一致,自描述 + 防改名
  "name": "小明",
  "note": "线下认识的",           // 可选,≤1024 字符
  "tags": ["朋友"],               // ≤32 个,每个 1~64 字符
  "createdAt": "2026-09-16T03:00:00.000Z",
  "updatedAt": "2026-09-16T03:00:00.000Z"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` / `version` | 字面量 | 是 | 固定 `"keymaster.contact"` / `1` |
| `publicKeyHex` | 字符串 | 是 | `^(02\|03)[0-9a-f]{64}$`,必须与文件名一致 |
| `name` | 字符串 | 是 | 显示名称,1~128 字符 |
| `note` | 字符串 | 否 | 备注;空字符串视为省略 |
| `tags` | 字符串数组 | 是 | 无标签用 `[]` |
| `createdAt` / `updatedAt` | 字符串 | 是 | ISO-8601 UTC;`updatedAt >= createdAt` |

单个文件不超过 16 KiB。不保存 `id`:文件名就是唯一身份。

## 规则

- **命名**:文件名 = 小写 `publicKeyHex` + `.json`。
- **新增**:文件已存在则拒绝,不覆盖。
- **更新/删除**:只动当前这一个文件。
- **损坏**:文件名与字段不一致、格式不匹配的文件跳过并提示,不影响其它联系人;没有索引,也没有跨文件校验。
- **并发**:不同联系人 = 不同文件,零冲突;同一联系人在同一 Worker 内串行,跨设备最后写入者胜(见《存储规则》)。

## 设计取舍

- 一人一文件:新增/删除不冲突,删除最简单,不需要索引与孤儿回收。
- 不存 `id`:canonical 身份本来就是公钥,文件名可直接承担。
- 保留 `publicKeyHex` 字段:读取不必信任文件名,两者交叉校验。
- 明文、无整体完整性:联系人不是秘密,接受单文件删除/篡改不可检测。

## 完整示例

路径:`桶/<owner 公钥>/contacts/address-book/023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f.json`

```json
{
  "format": "keymaster.contact",
  "version": 1,
  "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
  "name": "小明",
  "note": "线下认识的",
  "tags": ["朋友", "本地"],
  "createdAt": "2026-09-16T03:00:00.000Z",
  "updatedAt": "2026-09-16T03:00:00.000Z"
}
```
