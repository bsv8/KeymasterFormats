# `桶/<owner 公钥>/sat-subscription/setting.json` — SatSubscription 设置

按 owner 保存 SatSubscription 的**本地配置**。本文件只保存用户新增的 Supplier
和当前生效的选择，不保存 SS server 的订阅、账单或运行时状态。

SS server 是订阅关系与扣费记录的唯一事实来源；Keymaster 通过协议查询这些数据，不能把
服务器返回结果复制成桶内的第二份“真值”。

## 位置与命名

```
桶根/<owner 公钥>/sat-subscription/setting.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的小写压缩公钥；路径已经表达 owner 身份 |
| `sat-subscription/setting.json` | 固定目录与固定文件名；一个 owner 一份设置 |

## 文件格式

```jsonc
{
  "format": "keymaster.sat-subscription-setting",
  "version": 1,
  "suppliers": [
    {
      "supplierId": "backup",              // 本地稳定编号；不是远端身份
      "name": "备用供应商",                  // 设置页显示名称
      "supplierPublicKeyHex": "02ab...",    // Noise 认证必须匹配的供应商公钥
      "multiaddrs": ["/dns4/backup.example.com/tcp/443/tls/ws/p2p/16Uiu2..."],
                                               // 按顺序尝试的 libp2p 地址
      "enabled": true                        // 是否允许连接和执行操作
    }
  ],
  "defaultPublishSupplierId": "backup",    // 当前默认发布出口；省略/null = 内置默认
  "receiveSupplierIds": ["backup"]          // 当前额外接收入口；只引用新增 Supplier
}
```

### 字段

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定为 `"keymaster.sat-subscription-setting"` |
| `version` | 数字 | 是 | 固定为 `1` |
| `suppliers` | 对象数组 | 否 | 只保存用户新增的 Supplier；最多 64 个；省略或空数组 = 没有用户新增 Supplier |
| `suppliers[].supplierId` | 字符串 | 是 | 本地编号；`^[a-z][a-z0-9_-]{0,63}$`；按它去重 |
| `suppliers[].name` | 字符串 | 是 | 显示名称；1~128 个字符 |
| `suppliers[].supplierPublicKeyHex` | 字符串 | 是 | 小写压缩公钥；匹配 `^(02\|03)[0-9a-f]{64}$`；它才是远端身份 |
| `suppliers[].multiaddrs` | 字符串数组 | 是 | 1~32 个拨号地址；每条最多 512 个字符；按数组顺序尝试 |
| `suppliers[].enabled` | 布尔 | 是 | `false` 表示保留配置但不建立连接、不执行 SSP/SPI 操作 |
| `defaultPublishSupplierId` | 字符串 \| `null` | 否 | 当前普通 Publish 的默认出口；只能引用 `suppliers[].supplierId`；省略或 `null` 使用内置默认 Supplier |
| `receiveSupplierIds` | 字符串数组 | 否 | 当前启用的额外接收入口；最多 64 个、不能重复；只能引用新增 Supplier |

字段值中的字符串不得包含空字符、控制字符或首尾外的隐含格式；`channel`、订阅状态和
扣费金额不属于本文件。

## 缺省 Supplier 与 active 值

- **内置默认 SS server 不写入文件**：默认 Supplier（例如正式环境的 `bsv8`）由运行时按
  构建网络提供。它的 `supplierId`、公钥、地址和 `enabled` 值都不是本文件内容。
- 文件不存在，或 `suppliers` 为空时，运行时直接使用内置默认 Supplier；不因为默认值而
  创建文件。
- `defaultPublishSupplierId` 只在用户把默认发布出口切换到新增 Supplier 时写入。省略或
  `null` 都表示回到内置默认出口；不能写入内置默认 Supplier 的编号。
- `receiveSupplierIds` 只保存用户新增 Supplier 中当前生效的接收入口。省略表示没有额外
  入口，运行时按内置默认接收规则处理；数组中的每个编号都必须对应一个已保存且启用的
  Supplier。
- `suppliers` 不得出现内置默认 Supplier 的保留编号、公钥或地址；不能通过设置文件覆盖、
  伪装或禁用内置默认 Supplier。
- `enabled`、默认发布选择和接收选择都是**当前值**，不是历史记录；修改设置时覆盖旧值，
  删除用户 Supplier 时同时删除对它的 active 引用。

## 规则

- **只存配置**：本文件不保存 `ownerPublicKeyHex`、`supplierGeneration`、连接对象、连接状态、
  重连队列、在途请求、SPI 余额缓存或其它 Worker 运行态。owner 由目录名确定，generation
  由当前 Worker 会话生成。
- **不存服务器结果**：当前远端订阅列表通过 `SubscriptionsRequest` 查询；订阅/取消订阅的
  扣费结果与历史账单通过 SS server 的账单接口查询。`subscriptions`、`observedChannels`、
  `feeAudit`、`spiInformation`、`collectResults` 不得加入本文件。
- **不存 Channel 状态**：消息去重、ACK、Channel 的 desired/observed 集合属于 Channel/消息
  边界，不属于 SatSubscription 设置；需要持久化时另行定义其格式。
- **供应商身份**：`supplierId` 是本地引用，`supplierPublicKeyHex` 是远端身份；更新已有
  Supplier 时不能只改公钥而保留旧身份语义，身份变化应按删除旧配置再新增处理。
- **引用完整性**：`defaultPublishSupplierId` 和 `receiveSupplierIds` 引用不存在、重复或
  禁用的 Supplier 时，整份设置拒绝保存；读取到非法文件时按默认 Supplier 启动并报告配置错误。
- **字段白名单**：顶层和 Supplier 对象都拒绝未知字段；`format` 与 `version` 必须完全匹配。
- **明文保存**：公钥、地址、名称和启用选择不是私钥或密码；本文件不加密，但备份/导出会
  随 owner 目录一起携带。
- **修改方式**：读-改-写后整文件替换；同一文件在 Worker 内串行，跨浏览器按《存储规则》
  采用最后写入者胜。
- **大小限制**：单文件不超过 32 KiB；超过限制时拒绝写入并保持旧设置。

## 不属于本文件的内容

| 内容 | 事实来源或责任方 |
| --- | --- |
| 当前实际订阅列表 | SS server；通过 `SubscriptionsRequest` 读取 |
| 订阅/发布/ACK 的实际扣费记录 | SS server 账本；通过账单接口读取 |
| 连接状态、重连、在途请求 | Sat Worker / Window runtime 内存 |
| SPI Information、余额和价格 | SS server 实时响应；页面需要时重新查询 |
| Channel 去重与 ACK | Channel/消息模块另行负责 |

## 设计取舍

- **默认 Supplier 与用户 Supplier 分离**：默认 SS server 是产品能力，不复制进每个 owner 的
  设置文件；只有用户主动新增的 Supplier 才占用桶内容。
- **设置与服务端状态分离**：桶只保存“以后怎么连接、当前选谁”；服务器负责“已经订阅了
  什么、已经扣了什么”。刷新订阅或查看账单不会改写本文件。
- **不保存可重建缓存**：连接和远端观察结果可以在 Worker 启动后重新建立或查询；避免一个
  owner 同时拥有过期的本地订阅副本与服务器真值。
- **owner 由路径隔离**：不在 JSON 内重复写 owner 公钥，避免文件被复制到另一个 owner 目录
  后仍看起来合法。

## 完整示例

路径:

```
桶/<owner 公钥>/sat-subscription/setting.json
```

内容:

```json
{
  "format": "keymaster.sat-subscription-setting",
  "version": 1,
  "suppliers": [
    {
      "supplierId": "backup",
      "name": "备用供应商",
      "supplierPublicKeyHex": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
      "multiaddrs": [
        "/dns4/backup.example.com/tcp/443/tls/ws/p2p/16Uiu2HAmPGLn8pLWrSTqidMuq5P1rQBo9UhRwdAUjNVyjSwurtvH"
      ],
      "enabled": true
    }
  ],
  "defaultPublishSupplierId": "backup",
  "receiveSupplierIds": ["backup"]
}
```

示例中没有写入内置默认 SS server；删除 `defaultPublishSupplierId` 后，默认发布回到内置
Supplier；删除 `receiveSupplierIds` 后，不再增加用户自定义接收入口。
