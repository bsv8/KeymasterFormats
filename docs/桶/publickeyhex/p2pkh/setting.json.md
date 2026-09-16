# `桶/<owner 公钥>/p2pkh/setting.json` — P2PKH 设置

按 owner 保存的 P2PKH 偏好、provider 选择与配置,一个文件同时管 main 和 test 两个网络。文件不存在时全部使用默认值。

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
  },
  "providers": {                        // 可选;每个网络的 provider 选择
    "main": {
      "syncProviderId": "woc",          // 可选;null 或省略 = 未选
      "broadcastProviderId": "woc"
    },
    "test": {
      "syncProviderId": "woc",
      "broadcastProviderId": "woc"
    }
  },
  "providerConfigs": {                  // 可选;按 provider id 分组
    "woc": {
      "endpoint": "https://api.whatsonchain.com/v1/bsv",
      "requestsPerSecond": 3,
      "apiKey": "..."                   // 仅当该 provider 需要;敏感值
    }
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
| `providers` | 对象 | 否 | 每个网络的 provider 选择;只允许 `main`、`test` 两个键,省略 = 都未选 |
| `providers.<network>.syncProviderId` | 字符串 \| `null` | 否 | 确认数据(交易与高度)的 provider id;`null` 或省略 = 未选 |
| `providers.<network>.broadcastProviderId` | 字符串 \| `null` | 否 | 广播 provider id;`null` 或省略 = 未选 |
| `providerConfigs` | 对象 | 否 | 按 provider id 分组;每个值是 JSON 对象,字段由该 provider 自己定义 |
| `providerConfigs.<providerId>` | 对象 | 否 | provider 自有字段(例:`endpoint`、`requestsPerSecond`);**可能含 API key / token 等敏感凭据,明文保存** |

provider id 规则:`1~64` 字符,`^[a-z][a-z0-9._-]*$`。

## 规则

- **文件不存在 = 全部默认**:`includeTestnet = false`、费率 500/1000/2000、provider 未选,不需要为默认值写文件。
- **修改**:整文件替换(读-改-写);同一文件按《存储规则》处理(Worker 内串行,跨设备最后写入者胜)。
- **provider 可用性在运行时判断**:文件里可以保留一个当前不可用的 provider id(表示"我选过它");不可用时该网络的同步/广播按阻塞处理,不回退、不改写选择。
- **providerConfigs 由 provider 自己解释**:formats 只校验它是 JSON 对象;字段含义、默认值与取值范围属于对应 provider 的规范。
- **敏感值明文**:`apiKey` 之类的凭据直接写在文件里,文件本身不加密,备份/导出会带上。
- **关闭 testnet 不删除数据**:`tx/` 与 `height/` 里的 testnet 文件保留,重新打开即恢复展示。
- **校验**:本文件自身的未知字段拒绝;类型与取值按上表(`providerConfigs` 内部字段除外,由 provider 解释);单文件不超过 4 KiB。

## 设计取舍(消融记录)

- **不放大数据**:交易历史、UTXO 索引、同步游标都不在这里。
- **不按网络拆文件**:main/test 共用一份设置,偏好一次改完。
- **provider 选择与配置随 key 保存**:原来的 Coordinator 桶级 K-V 不再承载这些字段;`providerConfigs` 的字段由各 provider 定义,formats 不枚举,新增 provider 不需要改规范。
- **凭据明文**:不额外加密,拿到桶即可读;API key 一般可以吊销,接受。
- **不保存 generation**:同步代是运行时状态,不落盘。
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
  },
  "providers": {
    "main": {
      "syncProviderId": "woc",
      "broadcastProviderId": "woc"
    },
    "test": {
      "syncProviderId": "woc",
      "broadcastProviderId": "junglebus"
    }
  },
  "providerConfigs": {
    "woc": {
      "endpoint": "https://api.whatsonchain.com/v1/bsv",
      "requestsPerSecond": 3
    },
    "junglebus": {
      "endpoint": "https://junglebus.gorillapool.io",
      "timeoutMs": 10000,
      "maxRetries": 3
    }
  }
}
```
