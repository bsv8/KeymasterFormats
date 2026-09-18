# keymaster.device.v1 — 设备桶记录

设备本地保存"这台设备连过哪些桶":**一个桶一条记录**,键名 `keymaster.device.<ID>`。它不是一条总记录,没有"连接列表"外壳。

它只回答一个问题:这台设备登记了哪些桶。当前选中的是哪个桶由浏览器 session 记录(见《浏览器session》)。

定位:

- 它是**设备本地的连接控制面**,不是业务目录,也不是备份文件。
- 只保存"连接信息",不保存私钥、密码或业务数据。
- 记录本体是明文 JSON;**密文只可能出现在 s3 记录的 `cipher` 中**。
- 常见存放位置:设备本地存储(例如浏览器 `localStorage`)中键名形如 `keymaster.device.rs_s3_9f1c` 的值;本格式不限定存储介质。

---

## 键名

| 项 | 规则 |
| --- | --- |
| 键名 | `keymaster.device.<ID>` |
| `<ID>` | 就是连接的 `remoteStorageId`:1~128 字符,`^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$` |
| 枚举 | 扫描前缀 `keymaster.device.`;没有匹配键 = 尚未登记任何连接 |
| 唯一性 | 存储键天然唯一;一个 `<ID>` 只允许一条记录 |

`remoteStorageId` 不做大小写折叠或空白裁剪;local 连接同时把它用作对象前缀。

---

## 记录结构

`format` / `version` 固定;`displayName` 可选。

### 形态 A:local(只有公开坐标,没有 cipher)

```jsonc
{
  "format": "keymaster.device",
  "version": 1,
  "displayName": "本机测试桶",              // 可选
  "location": {
    "providerId": "local"                   // local 的对象前缀就是键名里的 <ID>
  }
}
```

### 形态 B:s3(公开坐标 + 凭据密文 + 可选能力缓存)

```jsonc
{
  "format": "keymaster.device",
  "version": 1,
  "displayName": "团队 S3 桶",              // 可选
  "location": {                             // 公开坐标(不含凭据)
    "providerId": "s3",
    "endpoint": "https://s3.example.com",
    "region": "auto",
    "bucket": "keymaster-data",
    "prefix": "team"                        // 可选
  },
  "cipher": {                               // 凭据密文(启动密码加密)
    "algorithm": "aes-gcm",
    "keyLengthBits": 256,
    "ivB64Url": "gT3kQ1sPvJ0mYw12",
    "tagLengthBits": 128,
    "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM7qX1tCpL4vN8xQ2sD6fG0h"
  },
  "capabilities": {                         // 可选:条件写能力缓存
    "conditionalWrites": "native"
  }
}
```

### 字段对照

| 字段 | local 形态 | s3 形态 |
| --- | --- | --- |
| `format` / `version` | 必填 | 必填 |
| `displayName` | 可选 | 可选 |
| `location` | 只有 `providerId` | 公开坐标 |
| `cipher` | **禁止出现** | **必需**,凭据密文 |
| `capabilities` | **禁止出现** | 可选,条件写能力缓存 |

字段说明:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字符串字面量 | 是 | 固定为 `"keymaster.device"`,用于识别记录类型 |
| `version` | 数字 | 是 | 固定为 `1` |
| `displayName` | 字符串 | 否 | 本机显示名称(可中文),1~128 字符;省略时读取方按坐标生成默认名 |
| `location` | 对象 | 是 | 不含访问凭据的公开坐标,两种变体见下 |
| `cipher` | 对象 | s3 必需 | 用启动密码派生的 key 加密的凭据密文;local 记录不得出现该字段 |
| `capabilities` | 对象 | 否 | 仅 s3:已探测到的条件写能力缓存,见下;local 记录不得出现该字段 |

不复存 `remoteStorageId`:键名里的 `<ID>` 是唯一真值。

### location — 公开坐标(两种变体)

`providerId` 是判别字段,两种变体的字段集合互不通用。

**local 变体** — local 没有独立物理坐标:对象前缀就是键名里的 `<ID>`。

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
4. `region`、`bucket` 和 `<ID>` 不做大小写折叠或空白裁剪;
   它们必须直接满足本文件的字段规则。
5. 对象键名按 Unicode 码点的字典序递归排序后,用不带空白的
   `JSON.stringify` 得到比较用的规范形式。

唯一性规则:

- 同一个 `<ID>` 只允许一条记录(存储键天然保证)。
- 两条 S3 记录的规范化 `location` 不得相同:保存前扫描全部
  `keymaster.device.*`,即使 `<ID>` 不同也不得重复登记同一物理位置。
- local 连接的有效物理目标是 `("local", <ID>)`。

位置指纹不落盘:需要比较时读取方自行计算。记录中也不允许出现
`physicalLocationFingerprint`、创建/更新时间、来源、恢复指针、轮转事务或
Worker profile 等字段;这些字段不是本格式的一部分。

### capabilities — s3 专属的条件写能力缓存

`s3` 记录可以缓存一次探测得到的条件写能力,避免每次连接/读取桶时重复对远端发送写探针。

```jsonc
{
  "conditionalWrites": "native"       // 仅 "native" | "best-effort"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `conditionalWrites` | 字符串枚举 | 是 | `"native"`:服务端原生执行 `If-None-Match` / `If-Match`(原子);`"best-effort"`:服务端忽略条件头,改用读 ETag 后写入模拟(非原子,接受竞态窗口) |

语义与规则:

- **仅 s3 形态允许**;local 记录出现该字段即整条记录无效(local 没有可缓存的能力结论)。
- **字段缺失 = 尚未探测**:读取方在需要时执行一次条件写探测,成功后按本记录格式写回。
- **不设过期时间、不做自动重探**:一旦写入即长期有效;探测结果只通过显式"重新探测"动作更新。
- **更新以完整记录替换**:更新 `capabilities` 必须保留 `location`、`cipher`、`displayName` 等原字段,且遵循"新增桶"以外的整记录替换规则。
- **连接配置变更即失效**:`location` 或 `cipher` 指向的连接发生变化时,原 `capabilities` 必须被丢弃(重建记录不写入),由下一次探测重新产生。
- 缓存被篡改的后果由读取方承担:伪造成 `"best-effort"` 只会导致多余的读后写入;伪造成 `"native"` 会让条件写失去原子保护。本格式不做额外认证(与 `location` 的取舍一致)。
- 该字段不是秘密,允许明文出现;它不属于密钥、密码或业务数据。

### cipher — s3 专属的凭据密文

只有含凭据的连接才有 `cipher`。v1 的加密方法**与 KeyHold 完全一致**:算法与长度完整写入 JSON,不依赖隐藏 profile 或代码约定;字段与规则如有出入,以 KeyHold 规范为准。

启动密码的 `keyDerivation` **不在本记录里**,它在 session(见《浏览器session》):一个浏览器一把密码,所有 s3 桶用同一个派生 key。

```jsonc
{
  "algorithm": "aes-gcm",
  "keyLengthBits": 256,
  "ivB64Url": "gT3kQ1sPvJ0mYw12",
  "tagLengthBits": 128,
  "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM7qX1tCpL4vN8xQ2sD6fG0h"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `algorithm` | 字符串 | 是 | 固定 `"aes-gcm"` |
| `keyLengthBits` | 数字 | 是 | 固定 `256` |
| `ivB64Url` | 字符串 | 是 | 随机 nonce,无填充 Base64URL;解码后恰好 12 字节 |
| `tagLengthBits` | 数字 | 是 | 固定 `128` |
| `ciphertextAndTagB64Url` | 字符串 | 是 | 密文与标签拼接,无填充 Base64URL;解码后 ≥ 16 字节(末尾是标签),字符串长度 ≤ 32768 |

Base64URL 在本格式中统一使用**无填充**形式:只允许
`A-Z a-z 0-9 - _`,不得出现 `=`、`+`、`/`、空白或换行;
解码失败或解码后重新编码与输入不一致,都按非法处理。

### 派生与加密步骤(v1,与 KeyHold 相同)

格式层没有默认参数:KDF 参数由 session 显式提供(见《浏览器session》),`cipher` 的算法与长度必须完整写入本记录。v1 固定采用以下步骤;算法和长度不是可选字段,改变它们必须提升 `version`:

```
passwordBytes = UTF-8(用户输入的启动密码)
encryptionKey = PBKDF2-HMAC-SHA-256(
  passwordBytes,
  salt = decodeBase64Url(session.keyDerivation.saltB64Url),
  iterations = session.keyDerivation.iterations,
  outputLengthBits = session.keyDerivation.outputLengthBits
)
plaintext = UTF-8(JSON({
  endpoint,
  region,
  bucket,
  accessKeyId,
  secretAccessKey,
  // sessionToken、prefix、forcePathStyle 仅在存在时加入
}))
(ciphertext, tag) = AES-256-GCM(
  key = encryptionKey,
  iv = decodeBase64Url(cipher.ivB64Url),
  plaintext,
  tagLengthBits = cipher.tagLengthBits
)
cipher.ciphertextAndTagB64Url = Base64URL(ciphertext || tag)
```

- **不使用 AAD**(与 KeyHold 相同):不附加认证数据,也不要求 canonical JSON;
  JSON 的字段顺序、缩进和换行不影响语义。
- 密码必须是非空 UTF-8 字符串;不 trim、不变更大小写、不做 Unicode normalization。
- KDF 参数(salt、iterations)由 session 提供;每次加密必须生成新的 12 字节随机 IV,同一个派生 key 下 IV 必须唯一。
- PBKDF2 输出的 32 字节直接作为 AES-256-GCM key,不再做第二次 KDF。

`displayName` 不参与加密,改显示名称不需要重新加密;凭据只存在于 `plaintext`。

`cipher` 解密得到的明文必须严格只有“解密后的明文”表中定义的字段,不能把未知字段带入运行态。

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

坐标在密文里再存一份是**事后校验**:本地攻击者可以改公开的 `location`,
但改不了密文;解密后两份对不上即判定为篡改,凭据不会被引向伪造的地址。
代价是本格式没有 AAD,把记录换挂到别的键名下无法被密码学发现(接受)。

#### 密码校验发生在哪里

- **local 记录**:没有 `cipher`,本记录不参与密码校验。
- **s3 记录**:**启动密码**的校验点就是能否解开 `cipher`(密码 + session 的 `keyDerivation`);密码错误、密文损坏、明文不合白名单或坐标比对失败统一按**认证失败**处理,不区分失败发生在哪一步(与 KeyHold 的 `unlock_failed` 相同)。

> Key 自己的密码与本记录无关:私钥密码属于 `keys/<公钥>.keyhold` 里的 KeyHold 文档(见该格式文档)。

#### 设计取舍:为什么 local 没有 cipher

1. local 的坐标就是键名里的 `<ID>`,是公开信息,没有凭据可放。
2. 给非秘密套一层密文会让"有 cipher 就有凭据"的规则变模糊。
3. 由此得到一条清晰的不变量:**`cipher` 出现即证明含凭据;没有 `cipher` 即证明无凭据。**

---

## 限额

| 项 | 上限 |
| --- | --- |
| 已登记桶数(`keymaster.device.*` 键数) | 32 |
| `displayName` 长度 | 128 字符 |
| 单条记录序列化后大小 | 131072 字节(128 KiB) |
| `cipher.ciphertextAndTagB64Url` 长度 | 32768 个 Base64URL 字符 |

超限即该条记录无效,不允许截断或部分接受。

---

## 校验规则

### 记录级(不需要密码)

1. **键名**:必须形如 `keymaster.device.<ID>`,`<ID>` 匹配
   `^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$`;前缀下的其它键视为无效。
2. **字段白名单**:每一层都拒绝未知字段;出现任何未定义字段即失败。
3. **格式与版本**:`format`、`version` 必须完全匹配。
4. **类型与取值**:字符串长度、字符集、枚举取值、必填可选按上文各表校验。
5. **变体自洽**:
   - `location.providerId = "local"` 时:记录只允许 `format`、`version`、
     `displayName`、`location`,且 `location` 只允许 `providerId`;
     `capabilities` 不得出现。
   - `location.providerId = "s3"` 时:`cipher` 必须存在,且只允许
     `algorithm`、`keyLengthBits`、`ivB64Url`、`tagLengthBits`、`ciphertextAndTagB64Url`;
     `capabilities` 可选,出现时只允许 `conditionalWrites`,取值只能是
     `"native"` 或 `"best-effort"`。
6. **跨记录唯一**:扫描全部 `keymaster.device.*`,规范化后的 S3
   `location` 不得重复;local 按 `("local", <ID>)` 判断有效物理目标。
7. **整体大小**:单条序列化后 ≤ 128 KiB;桶数 ≤ 32。

### 解密后(运行时)

1. 解密 `cipher` 失败(密码错误或密文被篡改)必须作为**认证失败**处理:
   不得降级为首次初始化,不得清除设备记录。
2. 解密后坐标(`endpoint`、`region`、`bucket`、`prefix`、`forcePathStyle`)
   必须与 `location` 一致,否则视为篡改。
3. local 记录不触发解密;读取方不得在 local 记录上寻找凭据。

单条记录任何一步失败都视为**该桶不可用**:跳过并提示,不影响其它桶;
不得自动删除或重建。

---

## 安全边界

| 允许出现 | 禁止出现 |
| --- | --- |
| 逻辑 ID、显示名、公开坐标 | 启动密码(会话密码)、Key 密码、派生密钥 |
| 条件写能力缓存(`capabilities`) | 任何探测过程产生的远端临时对象内容或凭据 |
| 不含凭据的 s3 endpoint / bucket / prefix | 公开 `location` 中出现任何凭据 |
| 密文封装与随机 IV | 私钥、助记词、业务数据;S3 AccessKey / Secret / sessionToken 明文 |

因此本记录本身**不加密**:桶名称、endpoint、使用过哪些 Provider 等信息以明文暴露在设备本地,应视为隐私元数据;凭据只允许封装在 `cipher` 里。

---

## 持久化与并发

- 一个桶一条记录,写入必须以完整记录替换,不得做部分字段写入。
- 新增 s3 桶要求 session 已有 `keyDerivation`(见《浏览器session》);没有就先创建。
- 新增桶:键不存在才写入,禁止覆盖已有键(否则会丢掉原凭据)。
- 删除桶 = 删除该键,不得留下"空壳"。
- 读取允许并发,不因读取加锁。
- 同一个 Worker 对同一个键的写入必须串行;写队列属于 Worker
  内存,不能写入记录本身。
- 独立浏览器之间不保证互斥或合并;先读后写产生冲突时,接受最后写入者胜。
- 没有任何 `keymaster.device.*` 键表示"设备尚未登记连接";单个键 JSON
  损坏、字段不符合本规范或版本不兼容都表示"该记录无效",不得自动删除或
  重建。
- 旧键 `keymaster.device-bootstrap.v1` 已废弃:不再读写,也不迁移。

---

## 完整示例

当前设备登记了两个桶,分别占用两个存储键:

键名:`keymaster.device.rs_local_7a01`

```json
{
  "format": "keymaster.device",
  "version": 1,
  "displayName": "本机测试桶",
  "location": {
    "providerId": "local"
  }
}
```

键名:`keymaster.device.rs_s3_9f1c`

```json
{
  "format": "keymaster.device",
  "version": 1,
  "displayName": "团队 S3 桶",
  "location": {
    "providerId": "s3",
    "endpoint": "https://s3.example.com",
    "region": "auto",
    "bucket": "keymaster-data",
    "prefix": "team"
  },
  "cipher": {
    "algorithm": "aes-gcm",
    "keyLengthBits": 256,
    "ivB64Url": "AAECAwQFBgcICQoL",
    "tagLengthBits": 128,
    "ciphertextAndTagB64Url": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4v"
  },
  "capabilities": {
    "conditionalWrites": "native"
  }
}
```
