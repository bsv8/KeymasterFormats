# `keymaster/keys.json` — 私钥文件

桶内唯一保存私钥的文件,是整个钱包的私钥真值。

- 位置固定:桶根下 `keymaster/keys.json`。
- 文件存在 = 该桶已经有钱包;不存在 = 首次初始化。
- 外壳是明文 JSON:只有算法常量、公开 KDF 参数、公钥和标签;私钥只以 AES-GCM 密文出现。
- 公钥索引、地址等都可以从本文件重建,因此不单独保存。

---

## 顶层结构

```jsonc
{
  "format": "keymaster.keys",                  // 格式标识
  "version": 1,                                // 格式版本
  "keyDerivation": {                           // 公开 KDF 参数
    "iterations": 600000,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  },
  "keys": [
    {
      "label": "主 Key",
      "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
      "cipher": {
        "ivB64Url": "gT3kQ1sPvJ0mYw12",
        "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM7qX1tCpL4vN8xQ2sD6fG0h"
      }
    }
  ],
  "integrity": {
    "tagB64Url": "0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字符串字面量 | 是 | 固定为 `"keymaster.keys"` |
| `version` | 数字 | 是 | 固定为 `1`;算法或结构变更时升版本 |
| `keyDerivation` | 对象 | 是 | 公开的密钥派生参数,不含密码或密钥 |
| `keys` | 数组 | 是 | 私钥记录列表,最多 10000 条;允许为空数组 |
| `integrity` | 对象 | 是 | 整档完整性标签 |

## keyDerivation — 公开 KDF 参数

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `iterations` | 数字 | 是 | PBKDF2 迭代次数,允许 100000 ~ 2000000 |
| `saltB64Url` | 字符串 | 是 | 随机盐,Base64URL;解码后必须是 16 字节 |

## keys[] — 私钥记录

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `label` | 字符串 | 是 | 显示名称,1~128 字符 |
| `publicKeyHex` | 字符串 | 是 | 压缩公钥,`^(02\|03)[0-9a-f]{64}$`;文件内必须唯一 |
| `cipher` | 对象 | 是 | 32 字节私钥的密文 |

`cipher`:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `ivB64Url` | 字符串 | 是 | 随机 nonce,Base64URL;解码后必须是 12 字节 |
| `ciphertextAndTagB64Url` | 字符串 | 是 | 密文 + 认证标签,Base64URL;解码后必须是 48 字节(32 字节私钥 + 16 字节标签) |

公钥以明文保存:它不是秘密,而且能在解锁前列出 Key、在解锁后与私钥重算结果交叉校验。

## integrity — 整档完整性

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `tagB64Url` | 字符串 | 是 | HMAC-SHA-256 标签,Base64URL;解码后必须是 32 字节 |

它同时承担两个职责:

1. **结构完整性**:防止静默删除、重排或替换 `keys` 记录;
2. **密码校验**:标签解不开即密码错误,因此不需要单独的密码校验记录。

---

## 固定算法与派生(v1)

算法和长度写进规范正文,不作为字段保存;变更时必须升 `version`。

```
master       = PBKDF2-HMAC-SHA-256(密码 UTF-8, salt, iterations, 输出 32 字节)
prk          = HKDF-Extract(salt, master)               // = HMAC-SHA-256(key = salt, data = master)
recordKey    = HKDF-Expand(prk, "keymaster.keys/v1/record",    32)
integrityKey = HKDF-Expand(prk, "keymaster.keys/v1/integrity", 32)
```

- 每条记录:`AES-GCM-256(recordKey, iv, 明文 = 32 字节 secp256k1 私钥)`,标签 128 位。
- 每条记录的 **AAD** = 规范化 JSON 的 `{ format, version, keyDerivation, publicKeyHex }`。
  - 这样密文与它的公钥和 KDF 参数绑定,不能移到另一条记录或另一份文件上复用。
  - `label` 故意不在 AAD 里:它的完整性由整档 `integrity` 覆盖,改名不需要重新加密私钥。
- `integrity.tagB64Url` = `HMAC-SHA-256(integrityKey, 规范化 JSON { format, version, keyDerivation, keys })`,标签自身不参与计算。
- 规范化:对象的键名按字典序递归排序后 `JSON.stringify`,不留空白。

---

## 位置与生命周期

| 阶段 | 形态 |
| --- | --- |
| 持久化 | 桶内 `keymaster/keys.json`,整文件替换写入;并发写规则见《存储规则》(同 Worker 内串行,跨端最后写入者胜) |
| 运行态 | 解密后的私钥只允许存在于内存;锁钱包、切换 Key、操作失败时立即清零 |
| 导出 | 直接复制本文件或其中一条记录,仍是密文;不需要解密 |

本文件是私钥的唯一持久化真值:公钥索引、地址、生命周期日志都可以由它重建,不应在别处保存私钥材料的副本。

---

## 限额

| 项 | 上限 |
| --- | --- |
| `keys` 条数 | 10000 |
| `label` 长度 | 128 字符 |
| 整个文件大小 | 16 MiB |

超限即整个文件无效,不允许截断或部分接受。

---

## 校验规则

### 记录级(不需要密码)

1. **字段白名单**:每一层都拒绝未知字段。
2. **格式与版本**:`format`、`version` 必须完全匹配。
3. **参数范围**:`iterations` 在范围内;`saltB64Url` 解码为 16 字节。
4. **记录**:`label` 长度合法;`publicKeyHex` 格式合法且在文件内唯一。
5. **密文**:`ivB64Url` 解码为 12 字节;`ciphertextAndTagB64Url` 解码为 48 字节。
6. **完整性字段**:`tagB64Url` 解码为 32 字节。
7. **大小限制**:条数与文件大小不超限。

### 解密后(运行时)

1. `integrity` 校验失败或记录解密失败,必须作为**认证失败**处理:不得当成"该桶没有 Key",不得覆盖或重建文件。
2. 任何一条记录解密失败,整个文件视为损坏;不得跳过坏记录继续使用其余部分。
3. 由明文私钥重新计算的压缩公钥必须等于记录里的 `publicKeyHex`;不一致即拒绝该文件。
4. 明文私钥只允许出现在内存中,使用完毕立即清零。

---

## 安全边界

| 允许出现 | 禁止出现 |
| --- | --- |
| 公钥、标签、随机盐、迭代次数 | 私钥明文、密码、派生密钥 |
| 随机 IV、密文与 GCM 标签、HMAC 标签 | 任何可用于重建私钥的材料 |

文件外壳本身不加密:公钥和标签以明文暴露是刻意的;秘密全部封装在 `cipher` 与 `integrity` 里。

---

## 设计取舍(消融记录)

- **不设独立密码校验记录**:整档 HMAC 就是校验点;解不开即密码错误。
- **不设算法字段**:v1 常量写进正文;算法变更升版本。
- **不设地址、网络、能力、时间戳**:地址可由公钥推导;其余不属于私钥本身。
- **保留 `publicKeyHex` 明文**:公钥非秘密,且能避免"先解密才能列出 Key"。
- **保留整档 `integrity`**:没有它,攻击者可以静默删除或重排 `keys` 记录而不被发现。
- **`label` 不参与 AAD**:完整性已由整档 HMAC 覆盖,改名无需重新加密私钥。

---

## 完整示例

两条 Key:

```json
{
  "format": "keymaster.keys",
  "version": 1,
  "keyDerivation": {
    "iterations": 600000,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  },
  "keys": [
    {
      "label": "主 Key",
      "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
      "cipher": {
        "ivB64Url": "gT3kQ1sPvJ0mYw12",
        "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM7qX1tCpL4vN8xQ2sD6fG0h"
      }
    },
    {
      "label": "交易 Key",
      "publicKeyHex": "035c7b9a2e4f6081c3d5e7f9012a4b6c8d0e2f4a6b8c0d1e3f4a5b6c7d8e9f0a1b",
      "cipher": {
        "ivB64Url": "R8dW4nB9sT2hJ6pA",
        "ciphertextAndTagB64Url": "mQ2xP7vK4cY8bN1sF5hJ9tR3wE6uZ0iL4oG7kD2qV8nX1cM5aB6dT0hY3jK9pW2z"
      }
    }
  ],
  "integrity": {
    "tagB64Url": "0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5"
  }
}
```
