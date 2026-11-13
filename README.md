# dsh-wechat-neo

把**个人微信**接进 [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness) 的桥接插件：私聊随手回，群里被 @ 才说话。

与 `dsh-wechat-pad` 等实现的区别：**不依赖任何外部协议端 / 云服务**。插件会自动拉起一个本地 sidecar 子进程运行开源协议库 [wechaty/puppet-neo](https://github.com/wechaty/puppet-neo)（底层 `wechat-protocol` crate），插件通过本机 HTTP 与它通信并接收 webhook；扫码登录你自己的微信号即可。

> ⚠️ 使用非官方协议登录个人微信，**违反微信软件许可及服务协议，有被封号的风险**（轻则 1 天，重则永久）。请只在你自己的号、且你愿意承担后果时使用。本项目不提供账号、不提供绕过风控的手段。
>
> ⚠️ 当前为实验版本（0.1.x）：**私聊 + 群聊文本**已实测可用；**图片/文件收发暂不支持**（见「已知限制」）。

## 架构

```
你的微信 ⇄ wechat-protocol（Rust，经 puppet-neo 的原生 addon）
              │  in-process
              ▼
        dsh-wechat-neo（本插件）⇄ 本地 HTTP ⇄ wpp-shim sidecar ⇄ puppet-neo
              │  Harness 内部通道（connection + typertGateway）
              ▼
        DSH 会话 / agent
```

- 入站：`puppet-neo` 的 message 事件 → 归一化成消息 → 交给内置 bridge → 唤起/续接对应 DSH 会话。
- 出站：agent 回复 → `puppet.messageSendText` 发回原聊天（私聊/群聊文本）。
- 每聊天一个独立 DSH 会话；群聊默认只在被 @ 时回复。

## 前置条件

本插件复用并扩展了 `wechaty/puppet-neo`，而上游 **没有发布到 npm、也没有 macOS-Intel 的原生包**。所以要先自己构建协议层：

1. 取 `wechaty/puppet-neo` 源码，并应用本仓库 [`patches/`](patches/) 下的三个补丁：
   - `native_src_lib.rs.patch`：原生层透出群消息（`room`）、解析群发送者前缀；
   - `src_puppet-neo.ts.patch`：允许向 `@chatroom` 发送文本、把 `roomId` 写进消息载荷；
   - `src_native-platforms.json.patch`：增加 `darwin-x64` 构建目标（Intel Mac 必需）。
2. 构建原生 addon 与 TS：
   ```bash
   cd puppet-neo
   # 需要 Node >= 26、Rust >= 1.93、Xcode CLT（macOS）/ C 工具链（Linux）
   node scripts/build-native.mjs      # 会在 native/<platform>/neo.node 生成 addon
   npm install && npm run build       # 生成 dist/
   ```
   > Apple Silicon 可直接用上游 CI 的 darwin-arm64 预编译包；Intel Mac 需按上面加 `darwin-x64` 目标自行编译。
3. 在本机把协议层装进 DSH profile（示例）：
   ```bash
   dsh plugin --profile web add /absolute/path/to/puppet-neo
   ```

## 安装本插件

```bash
# 本地目录安装（开发）
dsh plugin --profile web add /absolute/path/to/dsh-wechat-neo
# 或发布到 npm 后
dsh plugin --profile web add dsh-wechat-neo
```

装完重载/重启 DSH，打开 **设置 → 微信**（面板地址 `/wechat-pad`）：

1. 点「扫码登录」，用手机微信扫码并确认；
2. 登录成功后插件会自动记录 `selfWxid`，私聊即可对话；
3. 群聊：把该号拉进群，@ 它才会回（可在设置里改「群里每句都回」）。

## 配置（设置页 / `settings.json`）

| 项 | 默认 | 说明 |
| --- | --- | --- |
| 启用自动回复 | 开 | 关掉只记录不回复 |
| 群里怎么说话 | 只回 @ 我的 | mention / all / off |
| 私聊自动回复 | 开 | |
| 自动加入新群 | 关 | 打开后不在白名单的群也会回 |
| 连发合并窗口 | 800ms | 同一个人连发合并成一次提问 |
| 单条回复上限 / 分段长度 | 4000 / 1500 | |
| 等待模型超时 / 超时兜底话 | 120s | |
| 限速（单会话/全局） | 5 / 10 每分钟 | |
| 说话风格提示 | | 拼在每条 prompt 前 |
| 私聊白名单 `dmWhitelist` | 空=全部放行 | 只回复这些 wxid/微信号/昵称 |
| `sidecarScript` | 内置默认路径 | wpp-shim.mjs 的绝对路径 |

## 已知限制

- **图片 / 文件 / 语音收发未支持**：上游 `wechat-protocol` 的 CDN 上传下载 API 为 `pub(crate)`，嵌入方拿不到；`send_image` 还要求三份 JPEG（原图 ≤64KB）。要做需 fork 协议库把相关接口公开。
- 会话是**内存态**：进程/机器重启需重新扫码。
- 群消息发送者若为自定义微信号（非 `wxid_` 开头），显示名可能为空（正文前缀已由本仓库补丁处理）。
- 非官方协议，**封号风险自负**，建议先用小号。

## 致谢与来源

- 本插件的会话/设置/限速/合并等桥接逻辑 fork 自 [QIC-9821/dsh-wechat-pad](https://github.com/QIC-9821/dsh-wechat-pad)（MIT）。
- 微信协议能力来自 [wechaty/puppet-neo](https://github.com/wechaty/puppet-neo)（Apache-2.0）及其依赖 crate [`wechat-protocol`](https://crates.io/crates/wechat-protocol)（2.1.0）。
- `patches/` 下为本仓库对 puppet-neo 的最小改动。

## License

MIT（见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)）。
