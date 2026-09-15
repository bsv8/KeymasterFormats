# `桶/<owner 公钥>/p2pkh/test/tx/<txid>.json` — P2PKH 交易原始字节

格式与 [main 文档](../../main/tx/txhex.json.md)完全相同,只有网络不同:

- 目录是 `p2pkh/test`(主网是 `p2pkh/main`),两套数据不互通;
- 网络只影响地址显示与 provider 选择;P2PKH 脚本匹配用 HASH160,与网络无关。

命名规则、文件格式、写入/读取/校验、设计取舍均以 main 文档为准。
