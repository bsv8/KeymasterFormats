# `桶根/keys/<公钥>.keyhold` — 私钥文件

一个 Key 一个文件:文件名是压缩公钥(小写),内容就是一份**完整的 KeyHold v1 文档**。

- 位置:`桶根/keys/`。
- 文件名 = `publicKeyHex`(小写)+ `.keyhold`;内容里的 `publicKeyHex` 必须与文件名一致。
- `keys/` 下存在至少一份可解析的 KeyHold 文档 = 该桶已经有钱包。
- 每把 Key 有自己的密码;导入 / 导出就是文件拷贝:**不需要密码、不重新加密**。
- 列目录 = Key 列表;没有索引文件。
- KeyHold 的字段、加密、校验、限额全部以 KeyHold 规范为准,本文档只规定“放在哪、怎么命名”。
- 原 `keymaster/keys.json` 容器不再使用。

## 文件格式

没有外壳字段:文件本身就是 KeyHold 文档。

```jsonc
{
  "format": "keyhold",
  "version": 1,
  "label": "主 Key",
  "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
  "keyDerivation": { /* KeyHold 定义 */ },
  "cipher": { /* KeyHold 定义 */ }
}
```

## 命名规则

| 项 | 规则 |
| --- | --- |
| 目录 | `桶根/keys/` |
| 文件名 | `<公钥小写>.keyhold` |
| 公钥唯一 | 一个公钥只允许一个文件 |

## 导入 / 导出

- **导出**:复制选中的文件,字节不变;不需要密码。
- **导入**:按 KeyHold 规范校验 → 写入 `keys/<公钥>.keyhold`;**存储不需要密码**,该 Key 首次使用时才输入密码。
- **不导出、不备份桶参数**:桶参数在 `keymaster.device.<ID>` 记录的 `cipher` 里,由启动密码保护(仅 s3)。

## 生命周期与并发

- 新增 / 删除 Key = 增加 / 删除一个文件,不同 Key 互不影响;对同一把 Key 的新增 / 删除 / 使用受该 Key 的应用锁互斥(见《Key 应用锁》)。
- 删除文件与“用户正常删除 Key”无法区分(接受;Key 列表没有整体签名)。
- 单个文件损坏:跳过并提示,不影响其它 Key。

## 校验规则

1. 文件名必须等于小写 `publicKeyHex` + `.keyhold`。
2. 文件内容必须通过 KeyHold 规范校验,且 `publicKeyHex` 与文件名一致。
3. 解锁失败(密码错误 / 密文被改)按**认证失败**处理:不得当成“没有 Key”,不得覆盖或删除文件。
4. 解密后的私钥只允许在内存中,使用完毕立即清零。

## 设计取舍(消融记录)

- **一 Key 一文件**:导入导出 = 拷贝;无容器、无索引;新增 / 删除零冲突。
- **无整体完整性**:每份文档自带认证;单文件删除无法检测,这是接受的代价,并发由各 Key 自己的应用锁兜底。
- **只引用不复制 KeyHold 规则**:避免两份规范漂移。
- **不备份桶参数**:导出物就是单把 Key,不把两个密码域耦合在一起。

## 完整示例

路径:`桶根/keys/023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f.keyhold`

```json
{
  "format": "keyhold",
  "version": 1,
  "label": "主 Key",
  "publicKeyHex": "023f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f",
  "keyDerivation": {
    "algorithm": "pbkdf2-hmac-sha-256",
    "passwordEncoding": "utf-8",
    "iterations": 600000,
    "outputLengthBits": 256,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  },
  "cipher": {
    "algorithm": "aes-gcm",
    "keyLengthBits": 256,
    "ivB64Url": "gT3kQ1sPvJ0mYw12",
    "tagLengthBits": 128,
    "ciphertextAndTagB64Url": "kQ9fLx2mR7dW3cV1nB8sT4hJ6pA0uZ5yE2oI9gK3rM7qX1tCpL4vN8xQ2sD6fG0h"
  }
}
```
