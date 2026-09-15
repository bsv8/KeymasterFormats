# `桶/<owner 公钥>/p2pkh/setting.json` — P2PKH 设置

按 owner 保存的 P2PKH 偏好,一个文件同时管 main 和 test 两个网络。文件不存在时全部使用默认值。

## 位置与命名

```
桶根/<owner 公钥>/p2pkh/setting.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥,小写;每个 key 一份设置 |
| `p2pkh/setting.json` | 固定路径与固定文件名,不按网络拆分 |

## 文件格式

```jsonc
{
  "format": "keymaster.p2pkh-setting",
  "version": 1,
  "includeTestnet": false,              // 是否把 testnet 货币纳入钱包
  "feeRateSatoshisPerKb": {             // 可选;省略的档位用默认值
    "low": 500,
    "medium": 1000,
    "high": 2000
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.p2pkh-setting"` |
| `version` | 数字 | 是 | 固定 `1` |
| `includeTestnet` | 布尔 | 是 | `true`:testnet 货币进入资产/同步范围;`false`:只显示 main。默认 `false` |
| `feeRateSatoshisPerKb` | 对象 | 否 | 三档费率(sats/kB),可只写部分 |
| `feeRateSatoshisPerKb.low` | 数字 | 否 | 默认 `500`;正整数 |
| `feeRateSatoshisPerKb.medium` | 数字 | 否 | 默认 `1000`;正整数 |
| `feeRateSatoshisPerKb.high` | 数字 | 否 | 默认 `2000`;正整数 |

## 规则

- **文件不存在 = 全部默认**:`includeTestnet = false`、费率 500/1000/2000,不需要为默认值写文件。
- **修改**:整文件替换(读-改-写);同一文件按《存储规则》处理(Worker 内串行,跨设备最后写入者胜)。
- **关闭 testnet 不删除数据**:`tx/` 与 `height/` 里的 testnet 文件保留,重新打开即恢复展示。
- **校验**:未知字段拒绝;类型与取值按上表;单文件不超过 4 KiB。

## 设计取舍(消融记录)

- **只放产品偏好**:交易历史、UTXO 索引、同步游标都不在这里。
- **不按网络拆分**:这是"钱包级偏好",main/test 共用一份。
- **provider 选择与 provider 凭据不放这里**:选哪个区块浏览器、API key 属于设备级选择,不随 key 保存。
- **缺省即默认**:文件不存在时行为唯一确定,避免首次使用必须先写文件。
- **费率可部分覆盖**:只改一档不影响其它档,方便以后加档位。

## 完整示例

路径:

```
桶/<owner 公钥>/p2pkh/setting.json
```

内容:

```json
{
  "format": "keymaster.p2pkh-setting",
  "version": 1,
  "includeTestnet": true,
  "feeRateSatoshisPerKb": {
    "low": 500,
    "medium": 1500,
    "high": 2000
  }
}
```
