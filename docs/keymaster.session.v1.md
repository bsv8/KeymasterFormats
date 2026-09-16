# keymaster.session.v1 — 浏览器 session

记录浏览器的稳定身份、当前上下文(active 桶 + active key)和启动密码的派生参数。一个浏览器一条记录,直接存 JSON 文本。

## 存储

| 项 | 值 |
| --- | --- |
| 位置 | 设备本地存储(例如浏览器 `localStorage`) |
| 键名 | `keymaster.session` |
| 值 | UTF-8 JSON 文本(不额外编码) |

## 记录格式

```jsonc
{
  "format": "keymaster.session",
  "version": 1,
  "sessionId": "3f8a1c9d2e4b6a708192a3b4c5d6e7f8",
  "activeBucketId": "rs_s3_9f1c",     // 可选;active 桶的 remoteStorageId
  "activeKey": "023f8a...",           // 可选;必须与 activeBucketId 同时出现
  "keyDerivation": {                  // 可选;启动密码(会话密码)的 KDF 参数
    "algorithm": "pbkdf2-hmac-sha-256",
    "passwordEncoding": "utf-8",
    "iterations": 600000,
    "outputLengthBits": 256,
    "saltB64Url": "pQ7xW1sT4hJ6pA0uZ5yE2o"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.session"` |
| `version` | 数字 | 是 | 固定 `1` |
| `sessionId` | 字符串 | 是 | 32 位小写 hex 随机数;Key 应用锁归属只认这个字段 |
| `activeBucketId` | 字符串 | 否 | active 桶的 `remoteStorageId`(见《设备桶记录》),匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$` |
| `activeKey` | 字符串 | 否 | active key 的压缩公钥,66 位小写 hex(`02`/`03` 开头);`activeBucketId` 缺失时禁止出现 |
| `keyDerivation` | 对象 | 否 | 启动密码(会话密码)的 KDF 参数;没有 s3 桶时省略 |

`keyDerivation` 的字段与规则与 KeyHold 相同:

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `algorithm` | 字符串 | 是 | 固定 `"pbkdf2-hmac-sha-256"` |
| `passwordEncoding` | 字符串 | 是 | 固定 `"utf-8"` |
| `iterations` | 数字 | 是 | `1 ~ 2147483647`;推荐 `600000`;必须执行原值,不静默降低 |
| `outputLengthBits` | 数字 | 是 | 固定 `256` |
| `saltB64Url` | 字符串 | 是 | 无填充 Base64URL;解码后恰好 16 字节 |

## 启动密码(会话密码)

- **一个浏览器一把密码,所有 s3 桶共用**:`keymaster.device.<ID>` 记录的 `cipher` 都用这把密码派生的同一个 key 加密;每条记录各自生成随机 IV。
- 密码本身**永不落盘**:session 只保存公开的 KDF 参数;密码由用户每次输入。
- 密码必须是非空 UTF-8 字符串;不 trim、不变更大小写、不做 Unicode normalization(与 KeyHold 相同)。
- 密码是否正确,由"能否解开 s3 记录的 `cipher`,且明文通过白名单与坐标比对"确定;失败统一按认证失败处理,不区分原因。
- **没有 s3 桶时不需要密码**:local 记录没有 `cipher`,也不参与密码校验。
- **第一次需要一个 s3 桶时**生成 `keyDerivation`:16 字节随机 salt,推荐 `iterations = 600000`。
- **更换密码**:先解出全部 s3 明文,再用新参数重新密封所有 `cipher`,最后更新 session 的 `keyDerivation`;不允许只改 session 或只改一部分记录。

## 生成与生命周期

- 本浏览器**第一次完成初始化**(写入 `keymaster.device.<ID>` 记录)时生成 `sessionId` 并写入;
- `sessionId` 之后永不改变:刷新、锁钱包、重启浏览器都是同一个值;
- `activeBucketId` 在选定 / 切换连接成功后写入;值变化时同时清除 `activeKey`(两者成对维护),与当前值相同则保持不动;
- `activeKey` 在选定 / 切换 Key 成功后写入;
- 锁定钱包、关闭会话都不清除;
- 删除 Key 时若它正是 `activeKey`,清除该字段;删除桶记录时清除 `activeBucketId`,若删的正是 active 桶则同时清除 `activeKey`;
- 记录被清除 = 视为一台“新浏览器”,下次初始化时重新生成。

session 与 `keymaster.device.*` 必须放在同一介质里:两者一起被清除时密文不会变成孤儿;只剩 device 记录而没有 session(或反过来)都无法使用,只允许重新引导。

## 校验

1. 值必须是 UTF-8 JSON 对象。
2. 字段白名单;`format` / `version` 完全匹配;坏 JSON、未知字段,或 `sessionId` 缺失 / 不匹配 `^[0-9a-f]{32}$` → 按“新浏览器”处理。
3. `activeBucketId` 必须匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$`;缺失或非法 → 当作未选桶,并忽略 `activeKey`。
4. `activeKey` 必须匹配 `^(02|03)[0-9a-f]{64}$`;缺失或非法只忽略这个字段。
5. `keyDerivation` 若存在,按 KeyHold 规则校验:算法常量、迭代次数范围、salt 解码后 16 字节、Base64URL 规范形式;非法 → 当作没有启动密码。
6. 启动 / 切换时核对:`activeBucketId` 必须能在 `keymaster.device.<ID>` 里找到(见《设备桶记录》),`activeKey` 必须在该桶 `keys/` 里;对不上都视为未选定。

## 用途

- **sessionId**:作为 **Key 应用锁**的持有者标识(见《Key 应用锁》)——锁文件里的 `holder` 只取这个字段;多标签页共享同一份记录,天然算同一个持有者;
- **activeBucketId**:启动时预选连接,不用每次重新挑桶;
- **activeKey**:浏览器当前使用的 Key,每浏览器一个;进入钱包时用它预选 Key,启动 / 刷新后恢复上次选择;
- **keyDerivation**:配合用户输入的启动密码,解 `keymaster.device.<ID>` 的 `cipher`(见《设备桶记录》);
- 用于提示与诊断:“该 Key 正被另一个浏览器使用”。
