# 「小暖」Demo 实施方案 v1.0

> 日期：2026-10-23
> 原始需求见 [spec-v0.1.md](spec-v0.1.md)。本方案是核对各家官方最新文档后的定稿；与需求不一致的地方以本方案为准，第 2 节逐条列出改动及原因。

---

## 0. 目标

用最低成本，在 5–10 个真实家庭里测出两个数字：

- **老人接听率**，以及第 2 周和第 1 周的对比
- **子女日报打开率**

不影响这两个数字的功能一律不做。

---

## 1. 总体架构

```
老人 iPhone（iOS 26+）
  AlarmKit 主闹钟 + 备用闹钟
  SwiftUI 六个页面 / AVAudioEngine（回声消除、外放）/ Keychain 存 elder_token
        │  WSS /ws/call：上行 PCM16 16kHz，下行 PCM16 24kHz（二进制帧）
        │  HTTPS /api/elder/*
        ▼
香港服务器（单机，Docker Compose）
  Caddy：HTTPS / WSS，自动申请 Let's Encrypt 证书
  FastAPI 应用
    ├─ 通话中转（relay）+ 语音抽象层 ──WSS──▶ 阿里 qwen3.8-omni-flash-realtime（华北2 北京）
    │                                  └──▶ 智谱 glm-realtime-flash
    ├─ 业务 API / 管理 API / manage.py
    ├─ 调度器：每分钟未接检测、每晚 21:00 汇总
    ├─ 总结 + 风险判断 ──HTTPS──▶ 智谱 GLM-4.7-Flash（免费文本模型）
    ├─ H5 日报（Jinja2）+ 打开上报 ◀── 子女在微信里打开 /r/{report_token}
    └─ 通知（可插拔）──▶ 管理员：邮件为主，Server酱可选
  SQLite（WAL 模式）
```

**原则（沿用需求）**

- API Key 只在服务器环境变量里，app 只连我们自己的后端。
- 对话经后端中转，后端天然拿到双方转写并落库。
- 默认不保存录音，只存转写和总结。

---

## 2. 相对需求文档的改动

| # | 项 | 需求原文 | 定稿 | 原因 |
|---|---|---|---|---|
| 1 | 阿里模型 | qwen3-omni-flash-realtime | **qwen3.8-omni-flash-realtime**，地域华北2（北京） | 官方当前推荐；旧版只剩快照 `-2025-09-15`。北京地域才有新人免费额度 |
| 2 | 智谱模型 | air / flash | **glm-realtime-flash** 起步，air 备选 | 已不再免费；flash 0.18 元/分钟 |
| 3 | 音频格式 | PCM16 16kHz | **上行 16kHz、下行 24kHz**，均为 PCM16 单声道；WebSocket 二进制帧传输 | 两家模型都是这个格式；二进制帧比 base64 省带宽 |
| 4 | 闹钟 | 每日重复 + 取消当天备用 | **主闹钟 = 每周 7 天重复；备用 = 未来 7 天的单次闹钟，滚动补齐** | AlarmKit 不能取消重复闹钟中的某一次；"每天设次日"一旦某天没打开 app 就会断链 |
| 5 | 从闹钟接听 | 先进仿来电页，再点接听 | **点闹钟上的"接听"直接接通**；从其他入口在问候时段内打开 app，才显示仿来电页 | 避免老人接两次；也兜住"滑动停止后又自己打开 app"的情况 |
| 6 | 未接判定 | T+30 无已完成会话 | T+30 **无有效接听，且无进行中的会话** | 防止 T+25 才接起、仍在通话时被误判为未接 |
| 7 | 接听口径 | 未定义 | **有效接听 = completed，或 partial 且时长 ≥ 60 秒** | 统计接听率前必须定 |
| 8 | 时长控制 | 靠 prompt 写"3 分钟左右" | **服务端计时**：3:00 注入"可以收尾"，5:30 注入"请现在道别"，8:00 强制结束 | 模型不知道已经聊了多久 |
| 9 | 结束通话 | 小暖主动道别 | 服务端识别道别（工具调用或固定道别句，M1 选更稳的一种），**等这段语音发完**再挂断 | 需求没说怎么判断 |
| 10 | 风险告警 | 会后判断 | 通话中**实时扫描**老人转写，命中关键词立即告警；会后仍按两层判断 | 紧急情况不应等挂断后才知道 |
| 11 | 连接失败 | 重试 1 次 | 同一家重试 1 次，**再失败切另一家**；两家都失败才显示"等会儿再打来" | 提高接通率；会话记录实际使用的供应商 |
| 12 | 总结模型 | 待定 | **GLM-4.7-Flash（免费）**，可配置 | 零成本 |
| 13 | 管理员通知 | Server酱 / 邮件 | **邮件为主**，Server酱可选 | 邮件免费、无条数限制 |
| 14 | 日报打开统计 | 记录每次打开 | **页面 JS 上报 + UA 过滤 + 按天去重** | 防止链接预检、安全扫描把打开数抬高 |
| 15 | 知情同意朗读 | 小暖语音朗读 | **预先生成音频文件打包进 app** | 不需要实时语音合成 |
| 16 | 部署时机 | M7 | **M3 之前**就部署到香港服务器 | 之后的验收都在真实网络下做 |
| 17 | iOS 工程 | — | 用 **XcodeGen**（`project.yml`）生成工程 | 代码在 Linux 下写，你在 Mac 上一条命令生成 Xcode 工程，避免手改 pbxproj |
| 18 | 供应商接入 | — | 两家都直接用 **WebSocket 原生协议**（Python `websockets`），不用官方 SDK | 官方 SDK 一个是线程模型，一个封装了 aiohttp；统一 asyncio 更好控制打断和超时 |

---

## 3. 关键设计

### 3.1 app ↔ 后端通话协议（与供应商无关）

- **连接**：`wss://<域名>/ws/call?source=alarm|manual`，请求头带 `Authorization: Bearer <elder_token>`
- **上行**
  - 二进制帧：麦克风音频，PCM16 / 16kHz / 单声道，每帧 100ms（3200 字节）
  - 文本帧：`{"type":"hangup"}`
- **下行**
  - 二进制帧：小暖语音，PCM16 / 24kHz / 单声道
  - 文本帧：
    - `{"type":"ready","session_id":"..."}`：已接通
    - `{"type":"interrupt"}`：老人开口了，app **立即清空播放队列**
    - `{"type":"end","reason":"goodbye|timeout|hangup|error"}`
    - `{"type":"error","message":"..."}`

### 3.2 语音抽象层

```python
class VoiceProvider(Protocol):
    name: str
    async def connect(self, system_prompt: str, voice: str) -> None: ...
    async def send_audio(self, pcm16_16k: bytes) -> None: ...
    async def start_greeting(self) -> None: ...          # 让小暖先开口
    async def inject_hint(self, text: str) -> None: ...  # 计时提示等
    async def interrupt(self) -> None: ...               # 取消当前回答
    def events(self) -> AsyncIterator[VoiceEvent]: ...
    async def close(self) -> None: ...
```

- `VoiceEvent` 类型：`audio_out`（PCM16 24k）、`speech_started`、`speech_stopped`、`user_transcript`、`ai_transcript`、`response_done`、`end_requested`、`error`
- 实现：`QwenOmniRealtimeProvider`、`GLMRealtimeProvider`，另有一个 `FakeProvider` 专供测试（不需要 Key）
- 每次会话记录：供应商、首字延迟（`speech_stopped` 到第一个 `audio_out`）、总时长

### 3.3 一通电话的时间线

1. 鉴权 → 读老人档案和最近 3 天的总结 → 渲染 `prompts/xiaonuan_system.md`
2. 连接供应商（失败就重试，再失败切另一家）→ 给 app 发 `ready` → `start_greeting`，小暖先开口
3. 双向转发音频，转写实时落库
   - 收到 `speech_started`：通知 app `interrupt`，并让供应商取消当前回答
   - 老人的转写命中风险关键词：立即通知管理员
4. 3:00 注入收尾提示，5:30 注入道别提示，8:00 强制结束
5. 识别到道别 → 等这段语音发完 → 发 `end` → 关闭连接
6. 写入会话状态 → 异步生成总结 → 风险判断 → 需要时告警 → 更新日报

### 3.4 闹钟（AlarmKit）

- **主闹钟**：`Alarm.Schedule.Relative`，时间 T，每周 7 天重复，ID 固定
- **备用闹钟**：`Alarm.Schedule.fixed`，未来 7 天每天 T+10 各一个，ID 由日期生成（UUIDv5）
- **app 每次启动或回到前台**：确认主闹钟还在；补齐未来 7 天的备用闹钟；清理过期的
- **有效通话结束后**：取消当天的备用闹钟
- **闹钟按钮**："接听"作为次要按钮（`.custom` + App Intent 打开 app 并直接接通）；停止按钮的文案和交互待 spike 确认
- **不在客户端检测未接**：AlarmKit 无法可靠感知"划掉"（stopIntent 在划掉时可能不会被调用），未接只靠后端超时判断

### 3.5 未接检测与统计口径

- **每分钟检查**：当前时间 ≥ T+30，且今天没有有效接听、没有进行中的会话、还没标记过 → 记为 missed → 通知管理员 → 日报显示"今天还没聊"
- **有效接听** = completed，或 partial 且时长 ≥ 60 秒
- **接听率** = 有效接听天数 / 应问候天数（从完成知情同意的第二天算起）
- **日报打开率** = 有打开记录的天数 / 有日报的天数（打开以 JS 上报为准，同一天去重）

### 3.6 仓库结构（计划）

```
docs/                 需求、方案、验收记录
backend/
  xiaonuan/           FastAPI 应用（relay、voice/、summary、risk、scheduler、notify、report）
  prompts/            xiaonuan_system.md 等 prompt 模板
  config/             settings.yaml、risk_keywords.yaml
  scripts/            cli_talk.py（Mac 麦克风对话）、smoke_call.py、manage.py、stats.py
  tests/
ios/                  project.yml（XcodeGen）+ SwiftUI 源码
spikes/alarmkit/      AlarmKit 真机验证用的最小 app
deploy/               docker-compose.yml、Caddyfile、部署脚本
```

---

## 4. 怎么确保跑通

1. **先打掉两个最大的未知**：AlarmKit 真机验证（M0）和两家语音实测（M1），两项都过了再往下做。
2. **自动化测试不依赖 Key**：后端各模块用 `FakeProvider` 和录好的 WAV 文件测试；GitHub Actions 每次推送自动跑 lint 和测试。
3. **冒烟脚本** `scripts/smoke_call.py`：把一段录音当作"老人"推进 `/ws/call`，检查能收到语音、转写落库、生成总结。每次部署后跑一次。
4. **app 隐藏调试页**（待机页长按 5 秒进入）：1 分钟后响一次测试闹钟、立即通话、查看配置和闹钟列表、查看最近的错误。远程支持靠它。
5. **早部署**：M3 之前就上香港服务器，之后所有验收都走真实网络。
6. **内测 3 天**：正式试点前，你自己（或自家老人）完整用 3 天，跑通"响铃 → 接听 → 总结 → 日报 → 未接 → 告警"全流程。
7. **运行监控**：每晚 21:00 的汇总同时当健康检查；供应商报错、总结失败、通知失败都会单独通知你。

---

## 5. 需要你准备的

**现在（M0 / M1 用）**

- [ ] 阿里云**中国站**账号并实名 → 开通百炼 → 控制台右上角地域切到**华北2（北京）** → 创建 API Key，记下工作空间 ID
- [ ] 智谱开放平台账号并实名 → 创建 API Key → 充值 20–50 元（GLM-Realtime 收费）
- [ ] 把 Key 配到云端开发环境：环境设置 → Edit → 环境变量，变量名 `DASHSCOPE_API_KEY`、`DASHSCOPE_WORKSPACE_ID`、`ZHIPU_API_KEY`，新开的会话才会生效。**不要把 Key 贴在聊天里。**
- [ ] Mac：Python 3.11+、Xcode 26、`brew install xcodegen`、一副耳机（命令行测试没有回声消除）
- [ ] 一台 iOS 26 的 iPhone（你自己的）+ Apple 开发者账号
- [ ] TataAlarm：把仓库加到会话里，或者把 AlarmKit 相关代码放进本仓库

**M3 之前**

- [ ] 香港轻量服务器（2 核 2G、Ubuntu 即可）+ 一个域名，解析到这台服务器

**M5 之前**

- [ ] 决定日报怎么每天送到子女手里（见第 7 节）

**M6 之前**

- [ ] 通知邮箱的 SMTP 授权码，或 Server酱 SendKey

**试点之前**

- [ ] 招募 5–10 个家庭：老人用 **iPhone 11 及以后**的机型，并已升级到 iOS 26；记下所在城市和方言
- [ ] 小暖的名字、头像、音色（先用占位）

---

## 6. 执行顺序

| 阶段 | 我做 | 你做 | 验收标准 |
|---|---|---|---|
| **M0 准备**（与 M1 并行） | 仓库骨架、配置、语音抽象层 + FakeProvider、测试和 CI；AlarmKit spike app | 准备账号和 Key；在 iPhone 上跑 spike | spike 给出结论：锁屏、静音时能响；"接听"能打开 app 并开麦；"滑动停止"的实际表现 |
| **M1** | 接入 Qwen 和 GLM；Mac 命令行对话脚本（显示首字延迟和双方转写） | 戴耳机在 Mac 上分别和两家聊 | 两家都能流畅聊天；记录延迟和转写质量，决定是否需要统一转写 |
| **M2** | prompt 模板化、转写落库、计时提示、道别识别、总结 JSON、风险规则、实时告警 | 检查总结结果 | 一次命令行对话结束后，能看到正确的结构化总结 |
| **M3** | iOS 六个页面 + 通话；部署脚本 | 买服务器和域名、部署、在 iPhone 上通话 | iPhone 上点接听能和小暖流畅对话 |
| **M4** | 主闹钟 + 备用闹钟 + 闹钟重建 | 真机测试 | 锁屏、静音状态下到点响铃，点接听直接接通 |
| **M5** | H5 日报 + 7 天趋势 + 打开上报 | 在微信里打开 | 子女在微信里能看到当天日报 |
| **M6** | 未接检测、通知、每晚汇总、统计脚本 | 故意不接 | T+30 分钟收到通知 |
| **M7** | TestFlight 配置 | 内测 3 天 → 正式试点 | 一个真实家庭完整跑通一天 |

每个里程碑完成后都会停下来，等你验收再继续。

---

## 7. 待你拍板（不影响 M0 / M1 开工）

1. **日报怎么送到子女**：建议 Demo 阶段由系统每天生成"转发文案 + 链接"发给你，你转发到各家庭群；以后再换成服务号推送。
2. **测试家庭的城市和方言**：默认用普通话音色。阿里 3.8 能听懂上海话，但不能说上海话。
3. **技术栈**：默认按需求文档（Python 3.11+、FastAPI、SQLite、APScheduler、Jinja2）。

---

## 8. 成本估算（前期）

估算前提：开发调试约 200 通电话，试点 10 个家庭 × 4 周 ≈ 280 通，两家语音模型各打一半。

| 项目 | 估算 |
|---|---|
| 阿里实时语音（每通约 0.1–0.3 元） | 25–70 元 |
| 智谱实时语音（每通约 0.54 元） | 约 130 元 |
| 通话总结（GLM-4.7-Flash） | 0 |
| 香港轻量服务器 | 50–100 元（促销价约 25 元/月起，续费可能更贵） |
| 域名 | 几十元/年 |
| HTTPS 证书、邮件通知 | 0 |
| Apple 开发者账号 | 688 元/年（已有则为 0） |
| **合计** | **不含苹果账号约 300 元，含苹果账号约 1000 元** |

---

## 9. 已知风险

| 风险 | 应对 |
|---|---|
| iOS 26.1 起闹钟停止按钮改为"滑动停止"，老人可能当成"滑动接听"而误关闹钟 | M0 真机验证；有备用闹钟兜底；问候时段内打开 app 仍显示仿来电页；绑定时当面教一次 |
| 锁屏时点"接听"可能需要先解锁 | M0 真机验证；绑定时帮老人设好面容 ID |
| GLM 只有约 2 分钟的通话记忆，3 分钟的电话后半段可能重复提问 | M1 实测；必要时在中途注入"已经聊过的话题" |
| GLM 转写质量文档自己都说不保证，影响总结 | M1 对比两家的转写；必要时中转层统一用一个识别模型 |
| 老人说话慢、停顿多：静音阈值短了会抢话，长了延迟高 | M1 调参，试用 Qwen 的语义判断模式（`semantic_vad`） |
| 未备案域名在微信里可能出现拦截或提示页 | 拿到域名后先发链接实测 |
| 老人手机机型太旧（iPhone XR、XS 不支持 iOS 26） | 招募时先确认机型 |
