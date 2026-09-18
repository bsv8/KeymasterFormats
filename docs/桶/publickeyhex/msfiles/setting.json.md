# `桶/<owner 公钥>/msfiles/setting.json` — MSFile 设置

按 owner 保存的 MSFile 设置:全局金额上限、读取并发和用户供应商列表。文件不存在时全部使用
默认值,只有系统内置官方供应商可用。

## 位置与命名

```
桶根/<owner 公钥>/msfiles/setting.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥,小写;每个 key 一份设置 |
| `msfiles/setting.json` | 固定路径与固定文件名 |

## 文件格式

```jsonc
{
  "format": "keymaster.msfiles-setting",
  "version": 1,
  "priceLimits": {                       // 可选;省略 = 尚未配置,Read 失败关闭
    "seedMaxPriceSatoshis": "5000",      // 规范十进制字符串;"0" = 不限
    "blockMaxPriceSatoshis": "1000"
  },
  "readConcurrency": {                   // 可选;省略字段用推荐值
    "mediaBlockReadConcurrency": 2,
    "globalSeedReadConcurrency": 4,
    "globalBlockReadConcurrency": 8,
    "globalStatConcurrency": 4
  },
  "suppliers": [                         // 可选;只存用户添加的供应商
    {
      "name": "nas",
      "publicKeyHex": "02ab...",
      "addresses": ["/dns4/nas.example.com/tcp/443/tls/ws/p2p/16Uiu2..."],
      "enabled": true
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` / `version` | 字面量 | 是 | 固定 `"keymaster.msfiles-setting"` / `1` |
| `priceLimits` | 对象 | 否 | 单个内容对象的全局最高金额;省略 = 未配置 |
| `priceLimits.seedMaxPriceSatoshis` | 字符串 | 是(在对象内) | 单个 Seed 的最高金额;`"0"` = 不限;uint64 十进制 |
| `priceLimits.blockMaxPriceSatoshis` | 字符串 | 是(在对象内) | 单个 Block 的最高金额;`"0"` = 不限;uint64 十进制 |
| `readConcurrency` | 对象 | 否 | 读取运输层并发;每项缺省 = 推荐值 |
| `readConcurrency.mediaBlockReadConcurrency` | 数字 | 否 | 单个媒体 Session 的 Block 并发,`1~16` |
| `readConcurrency.globalSeedReadConcurrency` | 数字 | 否 | 全局 Seed 并发,`1~8` |
| `readConcurrency.globalBlockReadConcurrency` | 数字 | 否 | 全局 Block 并发,`1~32` |
| `readConcurrency.globalStatConcurrency` | 数字 | 否 | 全局 Stat 并发,`1~16` |
| `suppliers` | 数组 | 否 | 用户供应商;最多 64 个;省略或空数组 = 无用户供应商 |
| `suppliers[].name` | 字符串 | 是 | 显示名称,1~200 字节 UTF-8 |
| `suppliers[].publicKeyHex` | 字符串 | 是 | 供应商压缩公钥,`^(02\|03)[0-9a-f]{64}$` |
| `suppliers[].addresses` | 字符串数组 | 是 | 1~16 条 multiaddr;同一供应商内去重,顺序即拨号顺序 |
| `suppliers[].enabled` | 布尔 | 是 | `false`:保留配置但不参与 Stat/Read |

金额用规范十进制字符串(`"0"` 或非零开头,无前导零),避免 JSON 数字精度问题。

`readConcurrency` 四项要么同时按上表校验,要么整体省略;必须满足
`mediaBlockReadConcurrency <= globalBlockReadConcurrency`。

## 规则

- **文件不存在 = 全部默认**:没有金额上限(Read 失败关闭直到保存)、四项并发取推荐值
  (2/4/8/4)、没有任何用户供应商;不需要为默认值写文件。
- **系统内置官方供应商不落盘**:内置供应商始终存在且启用,由平台常量提供;文件中出现与内置
  供应商同公钥的条目一律忽略,不覆盖内置身份,也不能通过本文件禁用或删除它。
- **供应商身份即公钥**:按 `publicKeyHex` 去重;编辑已有供应商不允许原地更换公钥。
- **修改**:整文件替换(读-改-写);同一文件按《存储规则》处理(Worker 内串行,跨设备最后写入者胜)。
- **切 Key 即切换配置**:本文件按 owner 隔离,切换 active key 后使用目标 Key 自己的设置。
- **明文保存**:供应商地址与金额上限不是秘密,不额外加密;备份/导出随 Key 目录一起走。
- **校验**:本文件自身的未知字段拒绝;类型与取值按上表;单文件不超过 32 KiB。

## 设计取舍(消融记录)

- **一个文件管全部**:金额上限、并发和用户供应商数量都很少,整文件替换最简单;不再使用桶级
  K-V(原 `.keymaster/system/msfile/*`)。
- **内置供应商与用户供应商分开**:内置官方供应商是系统缺省能力,不写进用户文件,避免旧备份、
  手工改动或删 Key 把系统缺省身份弄丢。
- **金额用字符串**:wire 与限额都是 uint64 语义,JSON 数字在 2^53 之后会失真。
- **缺省即默认**:文件不存在时行为唯一确定,第一次使用只需要改要改的字段。
- **不存运行状态**:Stat 缓存、连接、在途 Block、授权确认都不在这里。

## 完整示例

路径:

```
桶/<owner 公钥>/msfiles/setting.json
```

内容:

```json
{
  "format": "keymaster.msfiles-setting",
  "version": 1,
  "priceLimits": {
    "seedMaxPriceSatoshis": "5000",
    "blockMaxPriceSatoshis": "1000"
  },
  "readConcurrency": {
    "mediaBlockReadConcurrency": 2,
    "globalSeedReadConcurrency": 4,
    "globalBlockReadConcurrency": 8,
    "globalStatConcurrency": 4
  },
  "suppliers": [
    {
      "name": "nas",
      "publicKeyHex": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
      "addresses": [
        "/ip4/127.0.0.1/udp/4001/webrtc-direct/certhash/uEiAeCUkerTt3eazN7DQMBrnCCRIPzvRaASapWgxZRWiyyw/p2p/16Uiu2HAmPGLn8pLWrSTqidMuq5P1rQBo9UhRwdAUjNVyjSwurtvH"
      ],
      "enabled": true
    }
  ]
}
```
