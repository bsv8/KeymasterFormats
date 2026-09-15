# `桶/<owner 公钥>/p2pkh/test/height/<height>.json` — 我的交易在区块里的位置

格式与 [main 文档](../../main/height/height-num.json.md)完全相同,只有网络不同:

- 目录是 `p2pkh/test`(主网是 `p2pkh/main`),两套数据不互通;
- 网络只影响地址显示与 provider 选择,不影响文件格式。

命名规则、文件格式、写入/读取/校验、设计取舍均以 main 文档为准。
