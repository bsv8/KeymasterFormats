# keymaster.device-bootstrap.v1

设备引导记录(Device Bootstrap Catalog)。它回答两个问题:

1. 这台设备连过哪些存储桶?
2. 当前启动应该预选哪个桶?

定位:

- 它是**设备本地的连接控制面**,不是业务目录,也不是备份文件。
- 只保存"连接信息",不保存私钥、密码或业务数据。
- 记录本体是明文 JSON;**密文只可能出现在 s3 连接的 `config` 中**。
- 常见存放位置:设备本地存储(例如浏览器 `localStorage`)中键名为 `keymaster.device-bootstrap.v1` 的值;本格式不限定存储介质。

---

## 顶层结构

```jsonc
{
  "format": "keymaster.device-bootstrap",      // 格式标识
  "version": 1,                                // 格式版本
  "selectedRemoteStorageId": "rs_local_7a01",  // 启动预选的连接(可选)
  "connections": [
    { /* local 连接:没有 config */ },
    { /* s3 连接:   config 为密封配置 */ }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字符串字面量 | 是 | 固定为 `"keymaster.device-bootstrap"`,用于识别记录类型 |
| `version` | 数字 | 是 | 固定为 `1` |
| `selectedRemoteStorageId` | 字符串 | 否 | 当前启动预选的连接 ID。必须能在 `connections` 中找到;省略表示尚未选择 |
| `connections` | 数组 | 是 | 本机可用连接,最多 32 条;空数组表示还没连过桶 |

---

## connections[] — 连接条目

一条记录 = "这台设备连接过一个桶"。两种形态并排对照:

### 形态 A:local(只有公开坐标,没有 config)

```jsonc
{
  "remoteStorageId": "rs_local_7a01",
  "displayName": "本机测试桶",              // 可选
  "location": {
    "providerId": "local"                   // local 的对象前缀就是 remoteStorageId
  }
}
```

### 形态 B:s3(公开坐标 + 密封 config)

```jsonc
{
  "remoteStorageId": "rs_s3_9f1c",
  "displayName": "团队 S3 桶",              // 可选
  "location": {                             // 公开坐标(不含凭据)
    "providerId": "s3",
    "endpoint": "https://s3.example.com",
    "region": "auto",
    "bucket": "keymaster-data",
    "prefix": "team"                        // 可选
  },
  "config": {                               // 密封配置(含凭据)
    "keyDerivation": { /* 公开 KDF 参数 */ },
    "cipher": { /* AES-GCM 密文 */ }
  }
}
```

### 字段对照

| 字段 | local 形态 | s3 形态 |
| --- | --- | --- |
| `remoteStorageId` | 必填 | 必填 |
| `displayName` | 可选 | 可选 |
| `location` | 只有 `providerId` | 公开坐标 |
| `config` | **禁止出现** | **必需**,密封 |

字段说明:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `remoteStorageId` | 字符串 | 是 | 稳定的**逻辑**身份。1~128 字符,首字符为字母或数字,其余允许 `A-Za-z0-9._:-`。local 连接同时把它用作对象前缀 |
| `displayName` | 字符串 | 否 | 本机显示名称(可中文),1~128 字符;省略时读取方按坐标生成默认名 |
| `location` | 对象 | 是 | 不含访问凭据的公开坐标,两种变体见下 |
| `config` | 对象 | s3 必需 | 密封配置;local 连接不得出现该字段 |

### location — 公开坐标(两种变体)

`providerId` 是判别字段,两种变体的字段集合互不通用。

**local 变体** — local 没有独立物理坐标:对象前缀就是 `remoteStorageId`。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `providerId` | `"local"` | 是 | 判别字段 |

**s3 变体**

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `providerId` | `"s3"` | 是 | 判别字段 |
| `endpoint` | 字符串 | 是 | 规范化后的 HTTPS 地址。禁止用户名、密码、query、fragment;保存时去掉末尾 `/`;最长 2048 字符 |
| `region` | 字符串 | 是 | S3 签名区域,最长 128 字符(Cloudflare R2 等转换后固定为 `"auto"`) |
| `bucket` | 字符串 | 是 | 物理 S3 桶名。3~63 字符,`^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$` |
| `prefix` | 字符串 | 否 | 对象前缀,最长 1024 字符。不得以 `/` 开头;按 `/` 分段后不得有空段、`.`、`..`;不得包含 `\` |
| `forcePathStyle` | 布尔 | 否 | 是否使用 path-style 请求;省略表示使用默认(虚拟主机风格) |

### location 规范化与唯一性

保存前必须先得到唯一的规范表示:

1. `endpoint` 解析为 URL,只接受 HTTPS;禁止用户名、密码、query 和
   fragment;使用 URL 的标准序列化结果,再去掉末尾全部 `/`。
2. `prefix` 去掉首尾全部 `/`;结果为空时省略该字段。规范化后按 `/` 分段,
   不允许空段、`.`、`..` 或包含 `\`。
3. `forcePathStyle` 为 `false` 时省略;只有 `true` 才写入。省略表示默认的
   虚拟主机风格,因此显式 `false` 与省略必须视为同一位置。
4. `region`、`bucket` 和 `remoteStorageId` 不做大小写折叠或空白裁剪;
   它们必须直接满足本文件的字段规则。
5. 对象键名按 Unicode 码点的字典序递归排序后,用不带空白的
   `JSON.stringify` 得到比较用的规范形式。

唯一性规则分开处理:

- `remoteStorageId` 在整个 `connections` 内必须唯一。
- 两条 S3 连接的规范化 `location` 不得相同,即使它们的
  `remoteStorageId` 不同也不得重复登记同一物理位置。
- local 连接的有效物理目标是 `("local", remoteStorageId)`;不同
  `remoteStorageId` 可以并存,相同 ID 已由第一条规则拒绝。

位置指纹不落盘:需要比较时读取方自行计算。设备引导中也不允许出现
`physicalLocationFingerprint`、创建/更新时间、来源、恢复指针、轮转事务或
Worker profile 等字段;这些字段不是本格式的一部分。

### config — s3 专属的密封配置

只有含凭据的连接才有 `config`。v1 中:

- KDF 固定为 **PBKDF2-HMAC-SHA-256**,密码编码 UTF-8,输出 256 位。
- 加密固定为 **AES-GCM-256**,认证标签 128 位。

算法常量写在规范里,不作为字段保存;算法或长度变更时升 `version`。

```jsonc
{
  "keyDerivation": {
    "iterations": 600000,
    "saltB64Url": "AAECAwQFBgcICQoLDA0ODw"
  },
  "cipher": {
    "ivB64Url": "AAECAwQFBgcICQoL",
    "ciphertextAndTagB64Url": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4v"
  }
}
```

`keyDerivation` — 公开的密钥派生参数(不是密码,也不是密钥):

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `iterations` | 数字 | 是 | 迭代次数,允许 100000 ~ 2000000 |
| `saltB64Url` | 字符串 | 是 | 公开随机盐,无填充 Base64URL;解码后必须是 16 字节(最长 128 字符) |

`cipher` — 用派生密钥加密的配置密文:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `ivB64Url` | 字符串 | 是 | 随机 nonce,无填充 Base64URL;解码后必须是 12 字节 |
| `ciphertextAndTagB64Url` | 字符串 | 是 | 密文与标签拼接,无填充 Base64URL;至少包含 16 字节标签,字符串长度 ≤ 32768 |

Base64URL 在本格式中统一使用**无填充**形式:只允许
`A-Z a-z 0-9 - _`,不得出现 `=`,不得接受无法按 Base64URL 解码的字符串;
编码方必须使用解码后字节重新编码得到的规范形式。`saltB64Url` 同样适用
这条规则,且解码后必须是 **16 字节**;因此规范的 16 字节盐通常是 22 个
字符,12 字节 nonce 通常是 16 个字符。

### v1 的派生、明文编码、AAD 与加密步骤

为了让不同实现生成可以互相解开的 `config`,v1 固定采用以下步骤;这里的
算法和长度不是可选字段,改变它们必须提升 `version`:

```
passwordBytes = UTF-8(用户输入的桶密码)
encryptionKey = PBKDF2-HMAC-SHA-256(
  passwordBytes,
  salt = decodeBase64Url(config.keyDerivation.saltB64Url),
  iterations = config.keyDerivation.iterations,
  outputLength = 32 字节
)
aad = UTF-8(canonicalJson({
  format: "keymaster.device-bootstrap",
  version: 1,
  remoteStorageId,
  location,
  keyDerivation: config.keyDerivation
}))
plaintext = UTF-8(canonicalJson({
  endpoint,
  region,
  bucket,
  accessKeyId,
  secretAccessKey,
  // sessionToken、prefix、forcePathStyle 仅在存在时加入
}))
(ciphertext, tag) = AES-GCM-256(
  key = encryptionKey,
  iv = decodeBase64Url(config.cipher.ivB64Url),
  plaintext,
  aad,
  tagLength = 128 位
)
config.cipher.ciphertextAndTagB64Url = Base64URL(ciphertext || tag)
```

其中 `canonicalJson` 的定义是:对象键名按 Unicode 码点的字典序递归排序,
数组顺序保持不变,再调用不带空白字符的 `JSON.stringify`;字符串按普通
JSON 规则转义,最终按 UTF-8 编码。AAD 中的 `location` 必须已经按本文件
的规范化规则处理。`displayName` 不参与 AAD,因此改显示名称不需要重新
加密;凭据明文也不参与 AAD,它只存在于 `plaintext`。

AAD 同时绑定连接的逻辑 ID、公开坐标和 KDF 参数。读取方必须使用当前记录
中的同一组字段重建 AAD;任何字段被替换后都应使认证失败。`config` 解密
得到的明文必须严格只有“解密后的明文”表中定义的字段,不能把未知字段带入
运行态。

#### 解密后的明文

解密 `cipher` 后得到完整连接配置。**这份明文不落盘,只允许在内存中存在。** 其中坐标字段必须与 `location` 一致,凭据只允许出现在这里:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `endpoint` | 字符串 | 是 | 最终的 S3-compatible HTTPS Endpoint |
| `region` | 字符串 | 是 | S3 签名区域 |
| `bucket` | 字符串 | 是 | 物理 S3 桶名 |
| `accessKeyId` | 字符串 | 是 | S3 Access Key ID(敏感) |
| `secretAccessKey` | 字符串 | 是 | S3 Secret Access Key(敏感) |
| `sessionToken` | 字符串 | 否 | 可选的临时会话令牌(敏感) |
| `prefix` | 字符串 | 否 | 可选的对象前缀 |
| `forcePathStyle` | 布尔 | 否 | 是否使用 path-style 请求 |

解密明文是一个严格的字段白名单,不得包含 `kind`、`providerId`、密码、
其它凭据字段或任何未知字段。明文中的 `endpoint`、`region`、`bucket`、
`prefix`、`forcePathStyle` 必须先按 `location` 的规则规范化,再与公开
`location` 逐字段比较;可选字段的“省略”和规范化后的缺省值必须保持同一
种表示。

坐标在密文里再存一份不是冗余,而是**认证绑定**:本地攻击者可以改公开的 `location`,但改不了密文;解密后两份对不上即判定为篡改,凭据不会被引向伪造的地址。

#### 密码校验发生在哪里

- **local 连接**:设备记录不参与密码校验;密码由桶内记录校验(桶格式另行定义)。
- **s3 连接**:校验点就是"能否解开 `cipher`";密码错误或密文被改动都会因认证标签失败。

#### 设计取舍:为什么 local 没有 config

1. local 的坐标就是 `remoteStorageId`,是公开信息,没有凭据可放。
2. 给非秘密加密封配置会让"有 config 就有凭据"的规则变模糊。
3. 由此得到一条清晰的不变量:**`config` 出现即证明含凭据;没有 `config` 即证明无凭据。**

---

## 限额

| 项 | 上限 |
| --- | --- |
| `connections` 条数 | 32 |
| `displayName` 长度 | 128 字符 |
| 整条记录序列化后大小 | 131072 字节(128 KiB) |
| `cipher.ciphertextAndTagB64Url` 长度 | 32768 个 Base64URL 字符 |

超限即整条记录无效,不允许截断或部分接受。

---

## 校验规则

### 记录级(不需要密码)

1. **字段白名单**:每一层都拒绝未知字段;出现任何未定义字段即失败。
2. **格式与版本**:`format`、`version` 必须完全匹配。
3. **类型与取值**:字符串长度、字符集、枚举取值、必填可选按上文各表校验。
4. **连接变体自洽**:
   - `location.providerId = "local"` 时:连接只允许 `remoteStorageId`、`displayName`、`location`,且 `location` 只允许 `providerId`;
   - `location.providerId = "s3"` 时:`config` 必须存在,且只允许 `keyDerivation`、`cipher`。
5. **唯一性**:
   - `remoteStorageId` 在 `connections` 内唯一;
   - 规范化后的 S3 `location` 在 `connections` 内唯一;
   - local 按 `("local", remoteStorageId)` 判断有效物理目标。
6. **选中项存在**:`selectedRemoteStorageId` 省略或指向已有连接。
7. **整体大小**:序列化后 ≤ 128 KiB。

### 解密后(运行时)

1. 解密 `cipher` 失败(密码错误或密文被篡改)必须作为**认证失败**处理:不得降级为首次初始化,不得清除设备记录。
2. 解密后坐标(`endpoint`、`region`、`bucket`、`prefix`、`forcePathStyle`)必须与 `location` 一致,否则视为篡改。
3. local 连接不触发解密;读取方不得在 local 连接上寻找凭据。

任何一步失败都应视为"整条记录不可用",由上层走重新引导流程,而不是丢弃个别字段继续使用。

---

## 安全边界

| 允许出现 | 禁止出现 |
| --- | --- |
| 逻辑 ID、显示名、公开坐标 | 桶密码、派生密钥 |
| 公开 KDF 参数(迭代次数、盐) | S3 AccessKey / Secret / sessionToken 明文 |
| 密封配置的密文封装与随机 IV | 私钥、助记词、业务数据 |
| 不含凭据的 s3 endpoint / bucket / prefix | 公开 `location` 中出现任何凭据 |

因此本记录本身**不加密**:桶名称、endpoint、使用过哪些 Provider 等信息以明文暴露在设备本地,应视为隐私元数据;凭据只允许封装在 `config` 里。

---

## 持久化与并发

- 设备引导是一个整体 JSON 文件,写入必须以完整记录替换,不得做部分字段或
  部分数组写入。
- 读取允许并发,不因读取加锁。
- 同一个 Worker 对同一个设备引导文件的写入必须串行;写队列属于 Worker
  内存,不能写入记录本身。
- 独立浏览器之间不保证互斥或合并;先读后写产生冲突时,接受最后写入者胜,
  不做跨浏览器 CAS、自动合并或静默丢弃未知连接。
- 记录缺失表示“设备尚未登记连接”;JSON 损坏、字段不符合本规范或版本
  不兼容都表示“记录无效”,不得自动当成缺失、删除或重建。

---

## 完整示例

同时包含 local 与 s3 两种连接,当前预选 local:

```json
{
  "format": "keymaster.device-bootstrap",
  "version": 1,
  "selectedRemoteStorageId": "rs_local_7a01",
  "connections": [
    {
      "remoteStorageId": "rs_local_7a01",
      "displayName": "本机测试桶",
      "location": {
        "providerId": "local"
      }
    },
    {
      "remoteStorageId": "rs_s3_9f1c",
      "displayName": "团队 S3 桶",
      "location": {
        "providerId": "s3",
        "endpoint": "https://s3.example.com",
        "region": "auto",
        "bucket": "keymaster-data",
        "prefix": "team"
      },
      "config": {
        "keyDerivation": {
          "iterations": 600000,
          "saltB64Url": "AAECAwQFBgcICQoLDA0ODw"
        },
        "cipher": {
          "ivB64Url": "AAECAwQFBgcICQoL",
          "ciphertextAndTagB64Url": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4v"
        }
      }
    }
  ]
}
```
