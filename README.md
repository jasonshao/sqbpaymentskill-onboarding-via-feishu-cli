# feishu-sqb-onboarding

> 把"收钱吧支付接入立项"流程搬进飞书工作台的 lark-cli skill。一句自然语言，30 秒产出**方案文档 + 进度多维表 + 代码骨架 zip + IM 任务派发卡片**。

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![lark-cli](https://img.shields.io/badge/lark--cli-%E2%89%A51.0.15-orange)](https://github.com/larksuite/cli)
[![End-to-end verified](https://img.shields.io/badge/E2E-live--tested-green)](#工程质量保证)

> **Built on top of [WoSai/sqb-payment-skills](https://github.com/WoSai/sqb-payment-skills) (Apache-2.0)** — 上游提供 8 个收钱吧业务接口 SKILL + 4 个共享模块 SKILL + Java/Python 参考实现。本仓**零修改**复用其内容，只做飞书工作台的最后一公里编排。

---

## 60 秒看懂

**问题** ｜ 收钱吧 ISV / 商户每接一次支付，开发者要在 5 个工具之间手动跨：翻文档、建进度表、写方案、复制代码骨架、群里 @ 对接负责人。**典型耗时 30+ 分钟，且每次都重复**。

**方案** ｜ 把这套流程压缩成一个 lark-cli AI Agent Skill。开发者对 Claude Code / Cursor / Codex 说一句话，AI 读本 [`SKILL.md`](feishu-sqb-onboarding/SKILL.md) → 调 6 条 `lark-cli` 命令 → **30 秒产出 4 件套**：飞书云文档、多维表、代码骨架 zip、IM 派发卡片。

**与 lark-cli 生态的关系** ｜ 严格对齐官方 [`skills/lark-workflow-meeting-summary`](https://github.com/larksuite/cli/tree/main/skills/lark-workflow-meeting-summary) / [`lark-workflow-standup-report`](https://github.com/larksuite/cli/tree/main/skills/lark-workflow-standup-report) 的 workflow skill 范式 —— markdown 主体 + frontmatter + Step-by-step CLI 编排。**不引入运行时 bot、不开 WebSocket、不需 cloudflared / ngrok**。

**关键差异化** ｜ 多数 lark-cli skill 是单点能力（"创建文档"、"查日程"）。本 skill 是**跨 4 个产品域（docs / base / drive / im）的工作流编排器**，且把"勾选哪些场景 → 命中哪些上游 SKILL"做成数据驱动（[`scene-skill-map.json`](feishu-sqb-onboarding/data/scene-skill-map.json)），可被复用为同范式（"X 业务接入立项"）的模板。

---

## Demo（实际跑出来长什么样）

完整端到端样例：[`feishu-sqb-onboarding/examples/餐饮-付款码-Java.md`](feishu-sqb-onboarding/examples/餐饮-付款码-Java.md)（含每步真实 stdout JSON）。下面是浓缩版：

### 输入

> "立项一下：深圳金店火锅，接付款码 + 退款，Java，5/31 上线，对接人 @张工"

### AI agent 干的事（按 [`SKILL.md`](feishu-sqb-onboarding/SKILL.md) 的 6 步）

```bash
# Step 3 — 创建方案云文档
$ lark-cli docs +create --title "《深圳金店火锅》收钱吧支付接入方案" --markdown ./proposal.md
{ "ok": true, "data": { "doc_url": "https://www.feishu.cn/docx/...", "doc_id": "..." } }

# Step 4 — 创建多维表 + 主表 1 行 + 子表 5 行（每个 SKILL 接口 1 行）
$ lark-cli base +base-create   --name "《深圳金店火锅》收钱吧接入" --time-zone "Asia/Shanghai"
$ lark-cli base +table-create  --base-token <token> --name "接入登记" --fields '[…9 字段…]'
$ lark-cli base +record-batch-create --json '{"fields":[…],"rows":[[…]]}'
$ lark-cli base +table-create  --base-token <token> --name "接入任务" --fields '[…5 字段…]'
$ lark-cli base +record-batch-create --json '{"fields":[…],"rows":[[…5 行…]]}'

# Step 5 — 打 zip + 上传 + 回填主行
$ lark-cli drive +upload --file ./skeleton.zip --name "《深圳金店火锅》收钱吧接入代码骨架.zip"
$ lark-cli base +record-batch-update --json '{"record_id_list":[…],"patch":{"代码骨架":"<zip_url>"}}'

# Step 6 — 派发 IM 卡片给 owner
$ lark-cli im +messages-send --user-id ou_xxx --msg-type interactive \
    --content "$(cat ./dispatch-card.json)" --idempotency-key "sqb-onboarding-…"
{ "ok": true, "data": { "message_id": "om_…" } }
```

### Owner 飞书 IM 收到的卡片

```
┌────────────────────────────────────────────────────────────┐
│ 📦 收钱吧接入立项 · 深圳金店火锅                              │
│ 餐饮 · Java · 涵盖 付款码、退款                              │
├────────────────────────────────────────────────────────────┤
│ 接入主体：深圳金店火锅      预计上线：2026-05-31（剩 30 天）  │
│ 对接负责人：@张工           接口任务：5 个 SKILL 已生成       │
├────────────────────────────────────────────────────────────┤
│ 下一步：1. 进度表领认任务 / 2. 下载 zip 起跑 / 3. SKILL.md   │
│         喂给 Claude Code 续写                                │
├────────────────────────────────────────────────────────────┤
│  [📄 方案文档]   [📊 进度表]   [📥 代码骨架]                   │
└────────────────────────────────────────────────────────────┘
```

> 想看真实运行结果？开发期用测试主体跑过 1 次完整真链路，下方截图为当时形态（工件已出于隐私销毁）。
<img width="415" height="117" alt="Clipboard_Screenshot_1777650780" src="https://github.com/user-attachments/assets/117cb30f-6ca3-4b15-ab5c-5357f75d7049" />

---

## 它解决什么

现状：收钱吧 ISV / 商户接入支付，开发者要在 5 个工具间手动跨：

| # | 步骤 | 时间 | 工具 |
|---|---|---|---|
| 1 | 翻文档梳理要接的接口 | 30 min+ | 飞书 wiki / 钉钉 / 企业微信文档 |
| 2 | 建一份进度跟踪表 | 10 min | Excel / 飞书表格 |
| 3 | 在飞书云文档里写方案 / 范围说明 | 20 min | 飞书云文档 |
| 4 | 复制 sqb-payment-skills 的 reference 代码、改商户名 | 10 min | IDE + 浏览器 |
| 5 | 在群里 @ 对接负责人交接 | 手动 | 飞书 IM |

**总耗时 70+ 分钟，且每次都重复。**

接入后：开发者对 AI agent 说一句话，**30 秒产出**：

| 产物 | 落点 | 内含 |
|---|---|---|
| **方案云文档** | 飞书 Drive | 抬头（含主体名 / 行业话术）+ 接入范围表 + 每个命中 SKILL 的描述 + 4 周节奏建议 + 5 条风险提醒 + owner 信息 |
| **进度多维表** | 飞书 Base | 主表 9 字段（接入登记）+ 子表 5 字段（每个 SKILL 接口 1 行任务） |
| **代码骨架 zip** | 飞书 Drive | `reference/<语言>/` 完整目录（仅相关场景）+ 自动渲染的 `KICKOFF.md`（3-step 起跑指南） |
| **IM 任务派发卡片** | 飞书 IM | 三按钮 interactive card：方案文档 / 进度表 / zip，@ 对接负责人，含 idempotency-key 防重复 |

---

## 快速上手（3 分钟可复现）

### 1. 装 lark-cli + 完成认证（一次性）

```bash
npm install -g @larksuite/cli
lark-cli auth login --recommend
lark-cli auth status   # 应回 tokenStatus: valid
```

> 需要的 scope：`docx:document:create`、`base:app:create` `base:table:create` `base:field:create` `base:record:create` `base:record:update`、`drive:file:upload`、`im:message.send_as_user`、`contact:user:search`。`--recommend` 一键全勾。

### 2. 拉本仓 + submodule

```bash
git clone https://github.com/jasonshao/sqbpaymentskill-onboarding-via-feishu-cli.git
cd sqbpaymentskill-onboarding-via-feishu-cli
git submodule update --init --recursive
```

### 3. 喂给 AI agent

把 [`feishu-sqb-onboarding/`](feishu-sqb-onboarding/) 整个目录拷到你的 AI agent 的 skill 目录，或在终端对它说：

> "读 `feishu-sqb-onboarding/SKILL.md`，按里面工作流帮我立项收钱吧接入：演示主体、餐饮、付款码 + 退款、Java、2026-06-01 上线、对接人是我自己"

AI 自动抽字段、调 lark-cli、产出 4 件套。**30 秒后**飞书 Drive / Base / IM 三处同步亮起。

---

## 架构

```
你的一句话
   │
   ▼
AI agent (Claude Code / Cursor / Codex) 读 SKILL.md
   │     ┌─ 抽 6 字段（不全则反问，每次只问一个）
   │     ├─ 读 data/scene-skill-map.json，把场景集合 → SKILL 接口集合
   │     └─ 读 templates/*.hbs，按 6 字段渲染产物
   ▼
6 个 lark-cli 命令（顺序串行，单步失败立即停手）
 ├─► docs +create               方案云文档      ──► doc_url
 ├─► base +base-create                          ──► base_token, base_url
 ├─► base +table-create + +record-batch-create   主表 9 字段 / 1 行
 ├─► base +table-create + +record-batch-create   子表 5 字段 / N 行
 ├─► drive +upload                              代码骨架 zip ──► zip_url
 ├─► base +record-batch-update                   回填主行 zip 链接
 └─► im +messages-send --msg-type interactive    @owner 派发卡片
   │
   ▼
飞书工作台：Drive / Base / IM 同步亮起 + owner 收到 @
```

**没有运行时 bot、没有 WebSocket、没有 cloudflared / ngrok**——AI agent 在你已装的 lark-cli 二进制上调 API，飞书工作空间是 server，本仓只是"指挥棒"。

---

## 设计取舍（为什么这么做）

| 决策 | 选定 | 否决的对面 | 理由 |
|---|---|---|---|
| 运行时形态 | **markdown skill**（agent 读 + 调 lark-cli） | 长连接 bot（@larksuiteoapi/node-sdk） | 锚定 lark-cli skill 生态；bot 形态需要解决公网回调（cloudflared/ngrok）、断线重连等运维问题，演示脆弱、复现门槛高 |
| 上游集成 | **submodule pin tag + 零修改** | fork 后改 / 直接 import 包 | submodule 让上游 SKILL.md 路径稳定可引用，pin commit 锁版本；零修改保住 Apache-2.0 派生边界，不与上游争夺更新权 |
| 场景到 SKILL 映射 | **数据驱动 JSON** ([scene-skill-map.json](feishu-sqb-onboarding/data/scene-skill-map.json)) | 硬编码 if-else | 加场景 / 改依赖时只改 JSON，不动 SKILL.md；同一份数据被 Step 1 解析、Step 2 模板、Step 4 子表行三处复用 |
| 输入字段数 | **6 个** | 12 个 / 多轮问答 | 6 个 = 方案文档 + Bitable + zip + 派发卡片所需的最小集合；多于 6 个会让 AI 反问回合超过 1 轮，体验变差 |
| 答疑能力 | **不做长连接侦听** | 主题线程 / 全群 fuse.js | AI agent 本身就是 RAG，让 owner 直接把 SKILL.md 路径喂给 Claude/Cursor 续写更轻量；多一个 bot 反而引入复杂度和误触发风险 |
| 产物语言覆盖 | **Java + Python**（其他语言只出文档） | 全语言 zip | 上游 reference 只有 Java/Python；其他语言由 AI 据 SKILL.md 现场推导，不假装包了所有人 |

---

## 与 lark-cli skill 体系的对齐

本 skill 严格遵循 lark-cli 官方 [skills/](https://github.com/larksuite/cli/tree/main/skills) 目录的写作约定：

| 约定 | 本 skill 的实现 |
|---|---|
| `SKILL.md` 主入口 + YAML frontmatter（`name` / `version` / `description` / `metadata.requires.bins`） | ✓ [SKILL.md:1-9](feishu-sqb-onboarding/SKILL.md) |
| 顶部 `**CRITICAL**` 块声明前置依赖 | ✓ 显式要求先 Read templates / data / submodule |
| "适用场景" 列触发词 + "前置条件" 列 scope | ✓ [SKILL.md "适用场景" / "前置条件"](feishu-sqb-onboarding/SKILL.md) |
| 工作流图（ASCII art）+ 编号 Step + 每步 lark-cli 命令 + 输出捕获约定 | ✓ Step 1–6 完整 |
| 错误处理 / 兜底表 + 权限表 + "参考"链接区 | ✓ 三段齐备 |
| 跨 skill 引用通过相对路径 + 仓内 SKILL.md 链接 | ✓ 引用 [`vendor/sqb-payment-skills/sqb-api-skills/<skill>/SKILL.md`](vendor/sqb-payment-skills) × 12 篇 + lark-cli 自身的 lark-base / lark-doc / lark-im skill |

最相似的官方参考：
- [`lark-workflow-meeting-summary`](https://github.com/larksuite/cli/tree/main/skills/lark-workflow-meeting-summary) —— "拉会议数据 → 调多个 API → 综合产出文档"
- [`lark-workflow-standup-report`](https://github.com/larksuite/cli/tree/main/skills/lark-workflow-standup-report) —— "calendar +agenda + task +get-my-tasks → AI 汇总"

本 skill 把这种 workflow 范式从"读取并汇总"扩展到"**接 6 字段输入 → 跨 4 产品域产出 4 件套**"。

---

## 工程质量保证

| 维度 | 证据 |
|---|---|
| **Schema 真验证** | 写 SKILL.md 时假设的 lark-cli 字段 schema 用 `--dry-run` + 真链路双轨验证。**捕到 5 处真实坑**已修：①Bitable 字段不是 `field_name`+数字 type 而是 shortcut 形式；② `record-batch-create` 是 `{fields:[],rows:[[]]}` 而不是 `{records:[{fields:{}}]}`；③ `--format json` 不是子命令 flag；④ `--markdown @path` 强制相对路径；⑤ `im --content` 不接受 `@file`/`-`，只能 `"$(cat ...)"` inline。详见 commit [`4ec9645 fix: D2.2 dry-run schema corrections`](https://github.com/jasonshao/sqbpaymentskill-onboarding-via-feishu-cli/commit/4ec9645) |
| **真链路 e2e** | 全部 6 个 lark-cli 命令在真测试租户跑通：1 docx + 1 bitable（含 2 表 6 行）+ 1 zip + 1 IM 卡片。从 `lark-cli auth status` 到 IM `message_id` 落地，**端到端约 18 秒**（详见 examples/[餐饮-付款码-Java.md](feishu-sqb-onboarding/examples/餐饮-付款码-Java.md) §"时间统计"） |
| **隐私安全** | `gitleaks detect` 跨全部 commit 0 leak；个人 open_id / app_id / 真名 / 邮箱（除提交身份外）均未入库；隐私审计后做了一次 `filter-branch` 清理早期 commit message 中残留的真测工件 token，并 force-push 同步 |
| **commit 历史可追** | 6 commit 严格按 `feat:` / `fix:` / `docs:` 类型分章，每条 message 写 why-not-only-what。`git log --oneline` 还原完整开发节奏（D1 scaffold → D2 templates+dry-run → D3 example+README → D4 polish）|
| **多端同步** | Mac 工作树 ↔ ECS（部署侧）↔ GitHub origin/main 三方一致，遵循 `~/CLAUDE.md` 的"Mac edit / ECS commit+push" DLP-bypass 流程 |
| **License 边界** | Apache-2.0 上游署名置于 README 顶部 + LICENSE 文件 + SKILL.md 内多处反复声明 |

---

## 仓库结构

```
.
├── feishu-sqb-onboarding/         ← 本仓的核心交付物（lark-cli skill）
│   ├── SKILL.md                   ← 主入口（680+ 行，6 字段抽取 / 工作流 / 错误兜底 / 权限表 / 引用）
│   ├── templates/                 ← 3 份 Handlebars 风格模板
│   │   ├── proposal.md.hbs        →  方案云文档
│   │   ├── kickoff.md.hbs         →  zip 顶层 KICKOFF.md
│   │   ├── card-dispatch.json.hbs →  IM 派发卡片
│   │   └── README.md              ← 模板使用约定（变量 / 转义 / 渲染时务必）
│   ├── data/
│   │   ├── scene-skill-map.json   ← 5 支付场景 → SKILL 并集（数据驱动）
│   │   └── industry-copy.json     ← 5 行业开头话术（注入 proposal.md.hbs）
│   └── examples/
│       └── 餐饮-付款码-Java.md     ← 1 个完整端到端样例（含每步 stdout）
├── vendor/
│   └── sqb-payment-skills/        ← git submodule（pin commit，零修改，Apache-2.0）
├── LICENSE                        ← Apache-2.0
└── README.md
```

---

## 6 字段输入规约

| 字段 | 类型 | 取值 | 下游用途 |
|---|---|---|---|
| `merchant_name` | 文本 | "深圳市XX餐饮管理有限公司" | 方案文档 / Bitable 主键 |
| `industry` | 单选 | 餐饮 / 零售 / 教培 / 服务 / 其他 | 行业话术注入（`industry-copy.json`） |
| `scenes` | 多选 | `pay`（付款码 B 扫 C）/ `precreate`（预下单 C 扫 B）/ `refund`（退款）/ `activate-checkin`（激活签到）/ `notify`（异步通知） | 决定要嵌入哪些 SKILL 接口 + 子表任务行 |
| `language` | 单选 | java / python（其他仅出文档） | 决定 zip 包 reference/<lang>/ |
| `target_launch` | 日期 | YYYY-MM-DD | 进度表里程碑 + days_left 提示 |
| `owner` | 飞书人员 | open_id 或邮箱 | Bitable person 字段 + IM `--user-id` |

---

## 与上游的边界

| 维度 | 上游 [WoSai/sqb-payment-skills](https://github.com/WoSai/sqb-payment-skills) | 本仓 |
|---|---|---|
| **业务接口知识**（付款码 / 预下单 / 退款 / 查询 / 激活 / 签到 / 撤单 / 通知） | ✅ 8 个 SKILL.md + Java/Python reference | ❌ 不实现 |
| **共享模块**（签名 / 状态判定 / 轮询 / 验签） | ✅ 4 个 SKILL.md + 共享代码 | ❌ 不实现 |
| **飞书工作台编排**（doc / base / drive / im） | ❌ | ✅ |
| **6 字段 → 4 件交付的工作流** | ❌ | ✅ |
| **License** | Apache-2.0 | Apache-2.0（与上游对齐） |

**本仓的真增量是工作流编排**，不再造 SQB 接入轮子。

---

## FAQ

**Q：为什么不做成飞书群里 `/sqb-init` 命令的 bot？**
A：评估过——bot 形态需要长连接 / 公网回调（cloudflared / ngrok），demo 脆弱（断线即翻车），且偏离 lark-cli skill 评分点。AI agent 形态用你已有的 Claude Code/Cursor 当运行时，零运维。

**Q：那用户在飞书群里怎么触发？**
A：本 skill 设计触发位是开发者本机的 AI agent（终端）。如果有"群里触发"诉求，配合飞书内部的 AI 应用（如多维表自动化 + AI 字段）即可，无需我们维护一个 bot。

**Q：上游 sqb-payment-skills 更新了怎么办？**
A：submodule pin commit 不动，重新 `git submodule update --remote vendor/sqb-payment-skills` 后 review 上游变更，按需更新本仓 [`scene-skill-map.json`](feishu-sqb-onboarding/data/scene-skill-map.json) 里的引用，再发新版。

**Q：换个业务（不是 SQB 是别的支付/SaaS）能复用吗？**
A：能。把 `vendor/sqb-payment-skills/` 换成你自己的 skill 仓 + 重写 [`scene-skill-map.json`](feishu-sqb-onboarding/data/scene-skill-map.json) + 改 [`industry-copy.json`](feishu-sqb-onboarding/data/industry-copy.json) 与三份模板里的业务话术。SKILL.md 的工作流骨架（6 字段 / Step 1-6 / 错误兜底）可保持不变。

**Q：30 秒是真的吗？**
A：开发期 1 次完整真测下来 ~18 秒（详 `examples/餐饮-付款码-Java.md` §时间统计）；30 秒是含网络抖动/限流的工程上限。**clone 后跑一次即可复现**。

**Q：和 [`lark-skill-maker`](https://github.com/larksuite/cli/tree/main/skills/lark-skill-maker) 是什么关系？**
A：lark-skill-maker 是 lark-cli 的"造 skill 的 skill"。本 skill 是它的下游产物 —— 用 lark-skill-maker 范式造出来的一个 workflow skill。

---

## License & 致谢

[Apache-2.0](LICENSE) — 与上游 `WoSai/sqb-payment-skills` 一致。

- 上游 SKILL 作者：[@WoSai](https://github.com/WoSai)（收钱吧）
- 飞书 lark-cli：[larksuite/cli](https://github.com/larksuite/cli)（MIT）
- 本仓编排：[@jasonshao](https://github.com/jasonshao)

