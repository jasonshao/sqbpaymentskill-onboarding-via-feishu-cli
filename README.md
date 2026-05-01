# feishu-sqb-onboarding

> 把"收钱吧支付接入立项"流程搬进飞书工作台的 lark-cli skill。一句自然语言，30 秒产出**方案文档 + 进度多维表 + 代码骨架 zip + IM 任务派发卡片**。

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![lark-cli](https://img.shields.io/badge/lark--cli-%E2%89%A51.0.15-orange)](https://github.com/larksuite/cli)

> **Built on top of [WoSai/sqb-payment-skills](https://github.com/WoSai/sqb-payment-skills) (Apache-2.0)** — 上游提供 8 个收钱吧业务接口 SKILL + 4 个共享模块 SKILL + Java/Python 参考实现。本仓库零修改引用其内容，**只做飞书工作台的最后一公里编排**。

---

## 它解决什么

现状：收钱吧 ISV / 商户接入支付，开发者要在 5 个工具间手动跨：
1. 翻文档梳理要接的接口（半小时起）
2. 建一份 Excel/飞书表跟踪进度（10 分钟）
3. 在飞书云文档里写方案 / 范围说明（20 分钟）
4. 复制 sqb-payment-skills 的 reference 代码、改商户名（10 分钟）
5. 在群里 @ 对接负责人交接（手动）

**接入后**：开发者对 AI agent（Claude Code / Cursor / Codex）说一句"给深圳XX餐饮立项收钱吧付款码，Java，5/30 上线，对接人 @张工"，AI 读 [`SKILL.md`](feishu-sqb-onboarding/SKILL.md) 后调 6 条 lark-cli 命令，**30 秒**产出：

| 产物 | 落点 |
|---|---|
| 方案云文档（含行业话术 + 命中接口的 SKILL.md 摘要） | 飞书 Drive |
| 进度多维表（主表 1 行 + 子表每个 SKILL 接口 1 行） | 飞书 Base |
| 代码骨架 zip（reference/<语言>/ + 渲染好的 KICKOFF.md） | 飞书 Drive |
| IM 任务派发卡片（@对接负责人，含三个链接） | 飞书 IM |

---

## 快速上手

### 1. 装 lark-cli + 完成认证（一次性）

```bash
npm install -g @larksuite/cli
lark-cli auth login --recommend
lark-cli auth status   # 应回 tokenStatus: valid
```

> 需要的 scope：`docx:document:create`、`base:app:create` `base:table:create` `base:field:create` `base:record:create` `base:record:update`、`drive:file:upload`、`im:message.send_as_user`、`contact:user:search`。`--recommend` 会一次批准。

### 2. 拉本仓 + submodule

```bash
git clone https://github.com/jasonshao/sqbpaymentskill-onboarding-via-feishu-cli.git
cd sqbpaymentskill-onboarding-via-feishu-cli
git submodule update --init --recursive
```

### 3. 喂给 AI agent

把 [`feishu-sqb-onboarding/`](feishu-sqb-onboarding/) 整个目录拷到你的 AI agent 的 skill 目录，或对它说：

> "读 `feishu-sqb-onboarding/SKILL.md`，按里面工作流帮我立项收钱吧接入：深圳金店火锅、餐饮、付款码 + 退款、Java、5/31 上线、对接人 @张工"

AI 会自己抽字段、调 lark-cli 命令、产出结果。完整样例见 [`feishu-sqb-onboarding/examples/餐饮-付款码-Java.md`](feishu-sqb-onboarding/examples/餐饮-付款码-Java.md)。

---

## 架构

```
你的一句话
   ↓
AI agent (Claude Code / Cursor / Codex) 读 SKILL.md
   ↓
6 个 lark-cli 命令（顺序串行）
 ├─► docs +create               方案云文档
 ├─► base +base-create
 ├─► base +table-create + +record-batch-create  主表 1 行
 ├─► base +table-create + +record-batch-create  子表 N 行
 ├─► drive +upload              代码骨架 zip
 └─► im +messages-send          @owner 派发卡片
   ↓
飞书工作台三处同步亮起 + owner 收到 @
```

**没有运行时 bot、没有 WebSocket、没有 cloudflared**——全部由 AI agent 在你已有的 lark-cli 二进制上调，你的飞书工作空间全程是 server，本仓只是把"指挥棒"交给 AI。

---

## 仓库结构

```
.
├── feishu-sqb-onboarding/         ← 本仓的核心交付物（lark-cli skill）
│   ├── SKILL.md                   ← 主入口
│   ├── templates/                 ← 3 份 Handlebars 风格模板
│   │   ├── proposal.md.hbs        →  方案云文档
│   │   ├── kickoff.md.hbs         →  zip 顶层 README
│   │   ├── card-dispatch.json.hbs →  IM 派发卡片
│   │   └── README.md
│   ├── data/
│   │   ├── scene-skill-map.json   ← 5 支付场景 → SKILL 并集映射
│   │   └── industry-copy.json     ← 5 行业话术
│   └── examples/
│       └── 餐饮-付款码-Java.md     ← 1 个完整端到端样例
├── vendor/
│   └── sqb-payment-skills/        ← git submodule（pin tag，零修改）
├── LICENSE                        ← Apache-2.0
└── README.md
```

---

## 6 字段输入规约（卡在前面）

| 字段 | 类型 | 取值 |
|---|---|---|
| `merchant_name` | 文本 | "深圳市XX餐饮管理有限公司" |
| `industry` | 单选 | 餐饮 / 零售 / 教培 / 服务 / 其他 |
| `scenes` | 多选 | `pay`（付款码 B 扫 C）/ `precreate`（预下单 C 扫 B）/ `refund` / `activate-checkin` / `notify` |
| `language` | 单选 | java / python |
| `target_launch` | 日期 | YYYY-MM-DD |
| `owner` | 飞书人员 | open_id 或邮箱 |

---

## 与上游的边界

| 维度 | 上游 sqb-payment-skills | 本仓 |
|---|---|---|
| **业务接口知识**（付款码 / 预下单 / 退款 / …） | ✅ 8 个 SKILL.md + Java/Python reference | ❌ 不实现 |
| **共享模块**（签名 / 状态 / 轮询 / 验签） | ✅ 4 个 SKILL.md + 共享代码 | ❌ 不实现 |
| **飞书工作台编排**（doc / base / drive / im） | ❌ | ✅ |
| **6 字段 → 5 件交付的工作流** | ❌ | ✅ |

**本仓的真增量是工作流编排**，不再造 SQB 接入轮子。

---

## License & 致谢

[Apache-2.0](LICENSE) — 与上游 `WoSai/sqb-payment-skills` 一致。

- 上游 SKILL 作者：[@WoSai](https://github.com/WoSai)（收钱吧）
- 飞书 lark-cli：[larksuite/cli](https://github.com/larksuite/cli)（MIT）
- 本仓编排：[@jasonshao](https://github.com/jasonshao)

提交至 **飞书 CLI Skill 大赛**（2026-05-05 截止）—— GitHub 赛道。
