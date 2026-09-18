# `桶/<owner 公钥>/p2p/setting.json` — P2P 设置

按 owner 保存的 P2P(WebRTC)连接偏好,目前只有 STUN 服务器列表。文件不存在或列表为空时使用默认值。

## 位置与命名

```
桶根/<owner 公钥>/p2p/setting.json
```

| 部分 | 规则 |
| --- | --- |
| `<owner 公钥>` | 当前 active key 的压缩公钥,小写;每个 key 一份设置 |
| `p2p/setting.json` | 固定路径与固定文件名 |

## 文件格式

```jsonc
{
  "format": "keymaster.p2p-setting",
  "version": 1,
  "stunServers": [                      // 可选;省略或空数组 = 默认
    "stun:stun.l.google.com:19302"
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | 字面量 | 是 | 固定 `"keymaster.p2p-setting"` |
| `version` | 数字 | 是 | 固定 `1` |
| `stunServers` | 字符串数组 | 否 | STUN 服务器 URL;最多 16 条;省略或空数组 = 默认 `["stun:stun.l.google.com:19302"]` |

单条 URL 规则:

- 必须以 `stun:` 开头,形如 `stun:host[:port]`;
- `host` 非空,字符限 `[A-Za-z0-9._-]`;
- `port`(可选)必须是 `1~65535` 的整数;
- 不含空白与控制字符,总长不超过 256;
- `turn:` / `turns:` 一律拒绝——不支持 TURN / 中继。

## 规则

- **文件不存在 = 默认**:默认只启用 `stun:stun.l.google.com:19302`,不需要为默认值写文件。
- **空列表 = 默认**:`stunServers` 省略或为空数组时回落默认;不允许保存成"没有任何 STUN"。
- **去重**:重复的 URL 只保留第一次出现。
- **整组校验**:任一条不合法 → 整组拒绝,不写入、不改内存态;读取到不合法文件按默认处理,不阻断启动。
- **修改**:整文件替换(读-改-写);同一文件按《存储规则》处理(Worker 内串行,跨设备最后写入者胜)。
- **只存配置,不存状态**:连接历史、ICE 结果、媒体参数不在这里。
- **校验**:本文件自身的未知字段拒绝;类型与取值按上表;单文件不超过 8 KiB。

## 设计取舍(消融记录)

- **只支持 STUN,不接 TURN**:`turn:` / `turns:` 直接拒绝,不做中继;避免在文件里承载中继凭据,也不引入"必须自建 TURN"的运维面。
- **不放大数据**:会话历史、连接质量、编解码参数都不在这里。
- **随 Key 保存**:与其他模块设置一致,切 Key 即切换配置;备份/导出随 Key 目录一起走。
- **明文保存**:STUN 地址是公开基础设施地址,不含凭据,不额外加密。
- **缺省即默认**:文件不存在时行为唯一确定,避免首次使用必须先写文件。
- **数量与长度设限**:最多 16 条、单条 ≤ 256 字符,避免文件被撑大。

## 完整示例

路径:

```
桶/<owner 公钥>/p2p/setting.json
```

内容:

```json
{
  "format": "keymaster.p2p-setting",
  "version": 1,
  "stunServers": [
    "stun:stun.l.google.com:19302",
    "stun:stun.a.example.com:3478"
  ]
}
```
