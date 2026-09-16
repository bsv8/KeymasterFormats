# `keymaster/keys.json` — 私钥文件(KeymasterHold 机制)

桶内唯一保存私钥的文件,是整个钱包的私钥真值。

- 位置固定:桶根下 `keymaster/keys.json`。
- 文件存在 = 该桶已经有钱包;不存在 = 首次初始化。
- **密码机制完全采用 KeymasterHold v1**:派生、AAD 结构、完整性、序列化、限额都遵循它;文档标识保持本文件自己的 `keymaster.keys`。
- **导出不需要解锁**:认证域固定为 `keymaster-hold`,导出时从本文件取出字段、组装成 `keymaster-hold` 文档即可,不重新加密、不重算标签。
- 导出物仍然是密文;规范不支持“无密码导出明文”。

## 顶层结构

```jsonc
{
  "format": "keymaster.keys",                  // 格式标识
  "version": 1,
  "keyDerivation": {                           // 公开 KDF 参数
    "algorithm": "pbkdf2-hmac-sha-256",
    "passwordEncoding": "utf-8",
    "iterations": 600000,
    "outputLengthBits": 256,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  },
  "storage": {                                 // 桶连接配置密文(见下)
    "cipher": { /* CipherEnvelope */ }
  },
  "keys": [
    {
      "label": "主 Key",
      "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
      "cipher": { /* CipherEnvelope */ }
    }
  ],
  "integrity": {
    "algorithm": "hmac-sha-256",
    "tagB64Url": "0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.keys"` |
| `version` | 数字 | 是 | 固定 `1` |
| `keyDerivation` | 对象 | 是 | 公开派生参数,不含密码或密钥 |
| `storage` | 对象 | 是 | 桶连接配置的密文,让文件可独立导出/冷导入 |
| `keys` | 数组 | 是 | 私钥记录,0~10000 条;顺序保留,公钥不重复 |
| `integrity` | 对象 | 是 | 整档认证标签 |

## keyDerivation — 公开 KDF 参数

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `algorithm` | 字面量 | 是 | `"pbkdf2-hmac-sha-256"` |
| `passwordEncoding` | 字面量 | 是 | `"utf-8"` |
| `iterations` | 数字 | 是 | 600000 ~ 2000000 |
| `outputLengthBits` | 字面量 | 是 | `256` |
| `saltB64Url` | 字符串 | 是 | 16 字节随机盐,无填充 Base64URL |

## keys[] — 私钥记录

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `label` | 字符串 | 是 | 显示名称;非空,UTF-8 不超过 1024 字节 |
| `publicKeyHex` | 字符串 | 是 | 66 位小写 hex 压缩公钥,`02`/`03` 开头且是有效曲线点;文件内唯一 |
| `cipher` | 对象 | 是 | 32 字节私钥的密文 |

**CipherEnvelope**(`storage` 与每条 Key 共用):

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `algorithm` | 字面量 | 是 | `"aes-gcm"` |
| `keyLengthBits` | 字面量 | 是 | `256` |
| `ivB64Url` | 字符串 | 是 | 12 字节随机 IV,无填充 Base64URL |
| `tagLengthBits` | 字面量 | 是 | `128` |
| `ciphertextAndTagB64Url` | 字符串 | 是 | 密文 + 16 字节标签;Key 记录解码后固定 48 字节 |

## integrity — 整档认证

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `algorithm` | 字面量 | 是 | `"hmac-sha-256"` |
| `tagB64Url` | 字符串 | 是 | 32 字节 HMAC 标签 |

它同时承担两个职责:防静默删除/重排 `keys` 记录,以及密码校验(解不开即密码错误)。

## 密码套件与派生(KeymasterHold v1)

```
M        = PBKDF2-HMAC-SHA-256(UTF8(密码), salt, iterations, 32 字节)
PRK      = HKDF-Extract-SHA-256(salt, M)
Kstorage = HKDF-Expand-SHA-256(PRK, "keymaster-hold/v1/storage", 32)
Kkey(P)  = HKDF-Expand-SHA-256(PRK, "keymaster-hold/v1/key/" + P, 32)
Kmac     = HKDF-Expand-SHA-256(PRK, "keymaster-hold/v1/integrity", 32)
```

- `P` 为记录的小写 `publicKeyHex`;用途字符串是 v1 固定规则,不写入文件。
- 每条记录用 AES-256-GCM 加密,IV 12 字节、标签 16 字节,加密前输入为 JCS 规范化 JSON。
- **认证域固定**:下列对象里的 `format` 一律取 `"keymaster-hold"`,**不是**本文件外层的 `"keymaster.keys"`;`keymaster.keys` 只是容器标识,不参与认证。
- **AAD**(UTF8(JCS(…))):
  - storage:`{ format: "keymaster-hold", version, keyDerivation, recordType: "storage" }`
  - key:`{ format: "keymaster-hold", version, keyDerivation, recordType: "key", label, publicKeyHex }`
- `integrity.tagB64Url = base64url(HMAC-SHA-256(Kmac, UTF8(JCS(认证对象))))`;
  认证对象是 `{ format: "keymaster-hold", version, keyDerivation, storage, keys, integrity: { algorithm } }`,不含标签自身。
- 规范化用 JCS(RFC 8785);序列化输出无 BOM、无结尾换行;Base64URL 无填充。

## 导出成 KeymasterHold 文档(无需解锁)

因为认证域固定为 `keymaster-hold`,本文件里的字段本身就已经是一个合法的 KeymasterHold 文档,只是外层标识不同:

1. 取出 `version`、`keyDerivation`、`storage`、`keys`、`integrity`;
2. 组装 `{ "format": "keymaster-hold", "version": …, "keyDerivation": …, "storage": …, "keys": …, "integrity": … }`;
3. 按 JCS 序列化(无 BOM、无结尾换行)。

全程不需要密码,也不重新加密或重算标签;结果直接通过 KeymasterHold 的完整性与解密校验。反向导入时,把外层 `format` 换回 `keymaster.keys` 存放即可。

只有外层 `format` 标识不受认证保护;改动它不影响任何校验,只改变文件被识别的方式。

## storage 的明文结构

只存在于加密前输入和解密后输出:

- local 桶:`{ "kind": "local" }`
- s3 桶:`{ kind: "s3", endpoint, region, bucket, accessKeyId, secretAccessKey, sessionToken?, prefix?, forcePathStyle? }`

它与 device-bootstrap 里那份密封配置**用途不同**:device-bootstrap 的用于“解锁前建立连接”,这里的用于“导出文件自包含、可冷导入”。

## 位置与生命周期

| 阶段 | 形态 |
| --- | --- |
| 持久化 | 桶内 `keymaster/keys.json`,整文件替换;并发写按《存储规则》 |
| 运行态 | 明文私钥只允许在内存;锁钱包、切换 Key、失败时立即清零 |
| 导出(无解锁) | 取字段组装成 `keymaster-hold` 文档(见上一节) |
| 导入 | 结构校验 → 密码派生 → 校验 integrity → 逐条解密;任一步失败按认证失败处理 |

本文件是私钥的唯一持久化真值:公钥索引、地址、交易历史都可以由它推导或重建。

## 限额(KeymasterHold v1)

| 项 | 上限 |
| --- | --- |
| JSON 文件原始 UTF-8 长度 | 16 MiB |
| `keys` 条数 | 0 ~ 10000 |
| JSON 嵌套深度 | 16 层 |
| `storage` 明文字节 | 64 KiB |
| 单个配置字符串 UTF-8 长度 | 16 KiB |
| `label` UTF-8 长度 | 1 ~ 1024 字节 |
| 密码 UTF-8 长度 | 1 ~ 1024 字节 |

超限即整个文件无效,不允许截断或部分接受。

## 校验规则

### 结构校验(先于密码派生)

1. 每层字段白名单,拒绝未知字段;格式与版本必须匹配。
2. `iterations` 在 600000 ~ 2000000;`saltB64Url` 解码为 16 字节。
3. Base64URL 无填充且规范;IV 解码 12 字节;Key 密文解码 48 字节;storage 密文加标签不超过 65,552 字节。
4. `publicKeyHex` 格式合法、是有效曲线点,且文件内唯一。
5. `label` 非空且不超过 1024 字节。
6. 条数、嵌套深度、文件大小不超限。

### 解密后(运行时)

1. `integrity` 校验失败或解密失败,必须作为**认证失败**处理:不得当成“该桶没有 Key”,不得覆盖或重建文件。
2. 任何一条记录解密失败,整个文件视为损坏;不得跳过坏记录继续使用。
3. 由明文私钥重算的压缩公钥必须等于记录里的 `publicKeyHex`。
4. 明文私钥只允许出现在内存中,使用完毕立即清零。

## 安全边界

| 允许出现 | 禁止出现 |
| --- | --- |
| 公钥、标签、算法标识、随机盐、迭代次数 | 私钥明文、密码、派生密钥 |
| 随机 IV、密文与 GCM 标签、HMAC 标签 | 任何可用于重建私钥的明文材料 |

文件外壳本身不加密:公钥和标签明文暴露是刻意的;秘密全部封装在 `cipher` 与 `integrity` 里。

## 设计取舍(消融记录)

- **密码机制采用 KeymasterHold v1,文档标识用 `keymaster.keys`**:派生、AAD 结构、完整性规则直接复用,不另立一套密码机制。
- **认证域固定为 `keymaster-hold`**:这是“无解锁导出”的关键——导出只是换一个外层标识,不需要密码,也不需要重算任何密文或标签。
- **代价**:字段比自定义最小版多——算法常量、`storage`、每条 Key 独立子密钥都回来了。
- **不设独立密码校验记录**:整档 HMAC 校验点是密码校验点。
- **保留 `publicKeyHex` 明文**:公钥非秘密,且能在解锁前列出 Key。
- **保留整档 `integrity`**:没有它,攻击者可以静默删除或重排 `keys` 记录。

## 完整示例

一条 Key、local 桶:

```json
{
  "format": "keymaster.keys",
  "version": 1,
  "keyDerivation": {
    "algorithm": "pbkdf2-hmac-sha-256",
    "passwordEncoding": "utf-8",
    "iterations": 600000,
    "outputLengthBits": 256,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  },
  "storage": {
    "cipher": {
      "algorithm": "aes-gcm",
      "keyLengthBits": 256,
      "ivB64Url": "gT3kQ1sPvJ0mYw12",
      "tagLengthBits": 128,
      "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM"
    }
  },
  "keys": [
    {
      "label": "主 Key",
      "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
      "cipher": {
        "algorithm": "aes-gcm",
        "keyLengthBits": 256,
        "ivB64Url": "R8dW4nB9sT2hJ6pA",
        "tagLengthBits": 128,
        "ciphertextAndTagB64Url": "mQ2xP7vK4cY8bN1sF5hJ9tR3wE6uZ0iL4oG7kD2qV8nX1cM5aB6dT0hY3jK9pW2z"
      }
    }
  ],
  "integrity": {
    "algorithm": "hmac-sha-256",
    "tagB64Url": "0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5"
  }
}
```
