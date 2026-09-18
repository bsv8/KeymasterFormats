# `桶/<owner 公钥>/app.<三方 app 公钥>/settings.json` — 三方 App 设置

按 owner 与三方 App 保存的 Keymaster 侧设置:本地观察到的 App 记录和按 App 的 MSFile
额度覆盖。`app.<三方 app 公钥>` 目录同时是三方 App 在桶内的家目录,`storage/` 子目录留给
App 自己使用(见 [README](./README.md))。

## 位置与命名

```
桶根/<owner 公钥>/app.<三方 app 公钥>/settings.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥,小写 |
| `app.<三方 app 公钥>` | `app.` + App 身份的发布者公钥(publisherPublicKeyHex),小写压缩公钥 |
| `settings.json` | 固定文件名;Keymaster 管理,App 自己不应改写 |

一个 publisher 可以有多个 App;App 之间按 `apps` 对象里的 `appId` 区分,不按目录拆分。

## 文件格式

```jsonc
{
  "format": "keymaster.app-settings",
  "version": 1,
  "publisherPublicKeyHex": "02ab...",     // 必须与目录名一致
  "apps": {
    "player.example": {                   // 键 = appId
      "name": "Player",                   // App 自称名称(本地观察)
      "firstSeenAt": "2026-09-19T03:00:00.000Z",
      "lastSeenAt": "2026-09-19T03:05:00.000Z",
      "msfiles": {                        // 可选;省略 = 全部继承全局
        "seedMaxPriceSatoshis": "250",    // 可选;"0" = 不限
        "blockMaxPriceSatoshis": "60"     // 可选
      }
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` / `version` | 字面量 | 是 | 固定 `"keymaster.app-settings"` / `1` |
| `publisherPublicKeyHex` | 字符串 | 是 | `^(02\|03)[0-9a-f]{64}$`,必须与目录名一致 |
| `apps` | 对象 | 是 | 键 = `appId`,值见下;没有记录时用 `{}` |
| `apps.<appId>.name` | 字符串 | 是 | App 自称的显示名称,1~256 字符 |
| `apps.<appId>.firstSeenAt` | 字符串 | 是 | 本机首次调用时间,ISO-8601 UTC;写入后不变 |
| `apps.<appId>.lastSeenAt` | 字符串 | 是 | 本机最近调用时间,ISO-8601 UTC;`>= firstSeenAt` |
| `apps.<appId>.msfiles` | 对象 | 否 | MSFile 的按 App 覆盖;省略 = 全部继承全局 |
| `apps.<appId>.msfiles.seedMaxPriceSatoshis` | 字符串 | 否 | 覆盖全局 Seed 上限;`"0"` = 不限;省略 = 继承 |
| `apps.<appId>.msfiles.blockMaxPriceSatoshis` | 字符串 | 否 | 覆盖全局 Block 上限;`"0"` = 不限;省略 = 继承 |

`appId` 规则同 Connect App 身份:`^[a-z0-9](?:[a-z0-9._-]{0,61}[a-z0-9])?$`,最长 63 字符。

## 规则

- **文件不存在 = 无记录**:任何 App 都继承全局 MSFile 金额上限,没有任何 override。
- **目录名 = publisher 身份**:`publisherPublicKeyHex` 与目录名交叉校验,不一致按损坏处理。
- **`name`/`firstSeenAt`/`lastSeenAt` 是本地观察**:来自本机实际发生的 Connect 调用,不是远端
  声明的真值;同一 App 在其它设备的调用不合并。
- **覆盖是可选字段**:`msfiles` 里只写要覆盖的字段;删除字段 = 恢复继承全局额度;`"0"` 是显式不限,
  与"省略=继承"不同,不能混用。
- **未知段保留**:`apps.<appId>` 下其它模块的段(未来扩展)在读-改-写时必须原样保留;本文件自身的
  未知顶层字段拒绝。
- **修改**:整文件替换(读-改-写);同一文件按《存储规则》处理(Worker 内串行,跨设备最后写入者胜)。
- **明文保存**:不加密;文件名与内容都不含秘密。
- **不自动清理**:没有 App 删除/过期逻辑;要清空记录就删除对应 `appId` 或整个文件。
- **校验**:类型与取值按上表;单文件不超过 16 KiB。

## 设计取舍(消融记录)

- **一 publisher 一目录**:App 身份 = publisher 公钥 + appId,目录按 publisher 聚合,桶不会被
  每个 appId 各开一个目录撑大;App 的 `storage/` 也只需要一个家目录。
- **settings.json 与 storage/ 分离**:`settings.json` 由 Keymaster 管理(授权、观察、覆盖额度),
  `storage/` 由 App 自由使用;两者权限与生命周期不同。
- **只记调用观察,不记远端许可**:没有服务端下发的授权账本;本机调用过才有记录,换设备各自记录。
- **缺省即继承**:不写文件 / 不写字段就是继承全局,避免把全局额度复制出一份可能过期的副本。

## 完整示例

路径:

```
桶/<owner 公钥>/app.03a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90/settings.json
```

内容:

```json
{
  "format": "keymaster.app-settings",
  "version": 1,
  "publisherPublicKeyHex": "03a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "apps": {
    "player.example": {
      "name": "Player",
      "firstSeenAt": "2026-09-19T03:00:00.000Z",
      "lastSeenAt": "2026-09-19T03:05:00.000Z",
      "msfiles": {
        "seedMaxPriceSatoshis": "250",
        "blockMaxPriceSatoshis": "60"
      }
    },
    "viewer.example": {
      "name": "Viewer",
      "firstSeenAt": "2026-09-19T04:00:00.000Z",
      "lastSeenAt": "2026-09-19T04:00:00.000Z"
    }
  }
}
```
