# `桶/<owner 公钥>/p2pkh/main/tx/<txid>.json` — P2PKH 交易原始字节

这个 key 在 mainnet 上的**所有交易原始字节**。桶里只保存 raw tx;UTXO、余额、花费关系由 Worker 在内存里解析,区块顺序来自 provider 同步。

- 一个交易一个文件,文件名就是 txid:已知 txid 可直接取文件。
- 新增交易写的是新文件,跨设备并发不冲突(见《存储规则》)。
- `main` 与 `test` 是两个独立目录,数据不互通。

## 位置与命名

```
桶根/<owner 公钥>/p2pkh/main/tx/<txid>.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥,小写;代表"谁的钱包" |
| `main` | 网络;测试网用 `test` |
| `<txid>` | 交易哈希,64 位小写 hex |

文件名不带区块高度:高度是链上信息,由 provider 同步提供;放进文件名反而要先知道高度才能取文件。

## 文件格式

```jsonc
{
  "format": "keymaster.p2pkh-tx",
  "version": 1,
  "txid": "ab12...",     // 与文件名一致,并与 rawTxHex 解析结果核对
  "rawTxHex": "0100..."  // 唯一真值
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.p2pkh-tx"` |
| `version` | 数字 | 是 | 固定 `1` |
| `txid` | 字符串 | 是 | `^[0-9a-f]{64}$`;与文件名、`rawTxHex` 解析结果一致 |
| `rawTxHex` | 字符串 | 是 | 完整原始交易字节,小写 hex,不含 `0x` 前缀 |

不保存地址、输入、输出、UTXO 状态、区块信息;需要时现场解析或由 provider 提供。

## 写入规则

- **确认后写入**:`tx/<txid>.json`;同一交易不重复写,文件名就是唯一身份。
- **未确认交易只在 Worker 内存**,不落盘。
- **reorg**:交易被回滚 → 删除或按新状态改写,由 Worker 依 provider 结果执行。

## 读取与重建(Worker 内存)

1. 列举 `tx/` 目录得到全部交易文件;
2. 解析每笔 `rawTxHex` → 输入 outpoint、输出 value/script;
3. 用 owner 公钥推导 P2PKH script(20 字节 HASH160,与网络无关)→ 标记属于自己的输出;
4. 回放:自己的输出 − 被后续交易花费的 = 可用 UTXO;
5. 区块高度、确认数和排序由 provider 同步提供;余额与交易列表在内存计算。

## 校验规则

1. 文件名必须匹配 `^[0-9a-f]{64}\.json$`。
2. `format`/`version` 匹配;`rawTxHex` 合法,且解析出的 txid 等于文件内 `txid` 与文件名。
3. 单个文件不合法:跳过并提示,不影响其它交易。

## 设计取舍(消融记录)

- **只存 raw tx**:原实现的 8 类记录(地址资源、交易事实、UTXO 投影、同步游标、本地交易、本地输出、输入占用、协议提交)全部可由 raw tx + 公钥重建,一律删除。
- **索引不落盘**:排序、UTXO、余额都在 Worker 内存,省掉 head/values 与 CAS 合并。
- **文件名只用 txid**:不需要先知道区块高度就能定位文件。
- **区块信息不落盘**:排序与确认数来自 provider 同步;离线启动时没有区块高度可展示。
- **未确认交易不落盘**:刷新后只能重新从 provider 同步,期间没有"本地已广播"的记账依据;若要防双花必须先落盘。
- **代价**:启动要列举并解析全部文件;历史越大越慢(没有持久化同步游标)。

## 完整示例

路径:

```
桶/<owner 公钥>/p2pkh/main/tx/9f1c8d2a4b6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f708.json
```

内容:

```json
{
  "format": "keymaster.p2pkh-tx",
  "version": 1,
  "txid": "9f1c8d2a4b6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f708",
  "rawTxHex": "0100000001a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f9000000006a47304402203f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f902201a2b3c4d5e6f708192a3b4c5d6e7f890a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4101000000ffffffff02e8030000000000001976a9143f8a1c9d2e4b6a708192a3b4c5d6e7f890a1b2c3d488ac88130000000000001976a9145c7b9a2e4f6081c3d5e7f9012a4b6c8d0e2f4a6b88ac00000000"
}
```
