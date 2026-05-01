# 例：深圳火锅 → 付款码 + 退款 / Java

> 一个完整端到端跑过的接入立项样例。展示用户原话、字段抽取、6 个 lark-cli 命令真实产出、最终回复结构。所有商户名/凭据已脱敏。

## 用户原话

> "帮我立项一下，深圳金店火锅要接收钱吧的付款码和退款，Java，5月底上线，对接人 @张工"

## Agent 字段抽取（一次抽满，无需反问）

| 字段 | 抽取值 | 来源 |
|---|---|---|
| `merchant_name` | 深圳金店火锅 | 直接抽 |
| `industry` | `餐饮` | "火锅" 命中映射 |
| `scenes` | `["pay", "refund"]` | "付款码" → `pay`、"退款" → `refund` |
| `language` | `java` | "Java" |
| `target_launch` | `2026-05-31` | "5月底" → 当月最后一天 |
| `owner` | `ou_xxx`（解析自 @张工） | 调 `lark-cli contact +search-user --query "张工"` |

## Step 1 — 场景到 SKILL 解析

读 `data/scene-skill-map.json`：

- `pay` → `[sqb-pay, sqb-signing, sqb-status-parsing, sqb-polling]`
- `refund` → `[sqb-refund, sqb-signing, sqb-status-parsing]`

并集 `skill_set`（去重）：`[sqb-pay, sqb-refund, sqb-signing, sqb-status-parsing, sqb-polling]`（5 个 SKILL）

逐个读 `vendor/sqb-payment-skills/sqb-api-skills/<skill>/SKILL.md`，取 frontmatter `description` + 一级标题。例 `sqb-pay`：

```
name: sqb-pay
description: [后端项目使用]收钱吧 B扫C 付款码支付接口技能。用于生成付款码支付的分层适配代码。…
```

## Step 2 — 渲染方案文档

读 `templates/proposal.md.hbs`，注入：
- `merchant_name="深圳金店火锅"`
- `industry="餐饮"`
- `industry_intro=<data/industry-copy.json["餐饮"]>`
- `scenes=[{key:"pay", label:"付款码（B 扫 C）", primary_skill:"sqb-pay", support_skills_joined:"sqb-signing / sqb-status-parsing / sqb-polling"}, {key:"refund", label:"退款", primary_skill:"sqb-refund", support_skills_joined:"sqb-signing / sqb-status-parsing"}]`
- `language="Java"` `target_launch="2026-05-31"` `today="2026-05-01"` `days_left=30`
- `skill_summaries=[…5 项…]`
- `primary_scene_skill="sqb-pay"`
- `owner_name="张工"`

写出 `./proposal-20260501-153042.md`。

## Step 3 — 创建云文档

```bash
lark-cli docs +create \
  --title "《深圳金店火锅》收钱吧支付接入方案" \
  --markdown ./proposal-20260501-153042.md
```

返回（截取）：

```json
{
  "ok": true,
  "data": {
    "url": "https://sqb.feishu.cn/docx/Df6Xwvvf1ilk4okJjEPcQHkYn6d",
    "document_id": "Df6Xwvvf1ilk4okJjEPcQHkYn6d"
  }
}
```

→ `doc_url = "https://sqb.feishu.cn/docx/Df6Xwvvf1ilk4okJjEPcQHkYn6d"`

## Step 4 — 创建 Bitable

### 4.1 建 Base

```bash
lark-cli base +base-create \
  --name "《深圳金店火锅》收钱吧接入" \
  --time-zone "Asia/Shanghai"
```

→ `base_token = "bscxxxxxxx"`、`base_url = "https://sqb.feishu.cn/base/bscxxxxxxx"`

### 4.2 建主表"接入登记"（9 字段）

详见 SKILL.md Step 4.2。返回 `main_table_id = "tblABC"`。

### 4.3 写主表 1 行

```bash
lark-cli base +record-batch-create \
  --base-token bscxxxxxxx --table-id tblABC \
  --json '{
    "fields": ["接入主体","行业","支付场景","开发语言","预计上线日","对接负责人","状态","方案文档","代码骨架"],
    "rows": [[
      "深圳金店火锅",
      "餐饮",
      ["付款码","退款"],
      "Java",
      "2026-05-31 00:00:00",
      [{"id":"ou_xxx"}],
      "待启动",
      "https://sqb.feishu.cn/docx/Df6Xwvvf1ilk4okJjEPcQHkYn6d",
      null
    ]]
  }'
```

→ `main_row_id = "rec001"`

### 4.4 建子表"接入任务" + 批插 5 行

```bash
lark-cli base +table-create --base-token bscxxxxxxx --name "接入任务" --fields '[
  {"type":"text","name":"任务"},
  {"type":"text","name":"接口SKILL"},
  {"type":"user","name":"负责人","multiple":false},
  {"type":"select","name":"状态","multiple":false,"options":[{"name":"待办"},{"name":"进行中"},{"name":"已完成"},{"name":"阻塞"}]},
  {"type":"text","name":"备注"}
]'
```

→ `task_table_id = "tblDEF"`

```bash
lark-cli base +record-batch-create \
  --base-token bscxxxxxxx --table-id tblDEF \
  --json '{
    "fields": ["任务","接口SKILL","负责人","状态","备注"],
    "rows": [
      ["对接 sqb-pay 接口", "sqb-pay", [{"id":"ou_xxx"}], "待办", null],
      ["对接 sqb-refund 接口", "sqb-refund", [{"id":"ou_xxx"}], "待办", null],
      ["对接 sqb-signing 接口", "sqb-signing", [{"id":"ou_xxx"}], "待办", null],
      ["对接 sqb-status-parsing 接口", "sqb-status-parsing", [{"id":"ou_xxx"}], "待办", null],
      ["对接 sqb-polling 接口", "sqb-polling", [{"id":"ou_xxx"}], "待办", null]
    ]
  }'
```

→ 5 个 `record_id_list`。

## Step 5 — 打 zip + 上传

### 5.1–5.2 选源 + 渲染 KICKOFF

按 `language=java` 选每个 skill 的 `reference/*.java` 与 `shared-reference/*.java`：

```
.staging-20260501-153042/
├── KICKOFF.md                   ← 渲染 templates/kickoff.md.hbs
├── shared-reference/
│   ├── SqbSignUtil.java
│   ├── SqbStatusUtil.java
│   └── SqbPollingUtil.java
├── sqb-pay/
│   ├── SKILL.md
│   └── reference/PayExample.java
├── sqb-refund/
│   ├── SKILL.md
│   └── reference/(refund Java 文件)
└── ... (其他 SKILL.md)
```

### 5.3 zip

```bash
( cd ./.staging-20260501-153042 && zip -r ../深圳金店火锅-sqb-java-20260501-153042.zip . )
```

### 5.4 upload

```bash
lark-cli drive +upload \
  --file ./深圳金店火锅-sqb-java-20260501-153042.zip \
  --name "《深圳金店火锅》收钱吧接入代码骨架.zip"
```

→ `file_token = "boxcnxxxx"`、`zip_url = "https://sqb.feishu.cn/file/boxcnxxxx"`

### 5.5 回填主行

```bash
lark-cli base +record-batch-update \
  --base-token bscxxxxxxx --table-id tblABC \
  --json '{
    "record_id_list": ["rec001"],
    "patch": { "代码骨架": "https://sqb.feishu.cn/file/boxcnxxxx" }
  }'
```

## Step 6 — 派发卡片给 owner

渲染 `templates/card-dispatch.json.hbs` 注入：
- `merchant_name="深圳金店火锅"` `industry="餐饮"` `language="Java"`
- `scenes_labels_joined="付款码、退款"`
- `target_launch="2026-05-31"` `days_left=30`
- `owner_name="张工"` `n_tasks=5`
- `doc_url=...` `base_url=...` `zip_url=...`

写到 `./dispatch-card-20260501-153042.json`，发送：

```bash
lark-cli im +messages-send \
  --user-id ou_xxx \
  --msg-type interactive \
  --content "$(cat ./dispatch-card-20260501-153042.json)" \
  --idempotency-key "sqb-onboarding-深圳金店火锅-2026-05-01"
```

→ `message_id = "om_xxxxx"`

## Agent 给用户的最终回复

```
## ✅ 收钱吧接入立项完成 — 深圳金店火锅

| 资产 | 链接 |
|---|---|
| 方案文档 | https://sqb.feishu.cn/docx/Df6Xwvvf1ilk4okJjEPcQHkYn6d |
| 进度表 | https://sqb.feishu.cn/base/bscxxxxxxx |
| 代码骨架 zip | https://sqb.feishu.cn/file/boxcnxxxx |
| 派发卡片 | 已发给 @张工（消息 id: om_xxxxx）|

涵盖场景：付款码、退款
开发语言：Java
预计上线：2026-05-31（剩 30 天）

下一步建议：
1. 让 @张工 在进度表里把 5 个 SKILL 接口任务认领并填备注
2. 把 vendor/sqb-payment-skills/sqb-api-skills/<scene>/SKILL.md 喂给 Claude Code/Cursor 生成具体调用代码
3. 上线日前 1 周，跑端到端联调
```

## 时间统计（预估）

| 步骤 | 耗时 |
|---|---|
| 字段抽取 + Step 1 SKILL 解析 | <2s |
| Step 2 渲染 + Step 3 docs+create | ~3s |
| Step 4.1–4.4 base 全套 | ~6s |
| Step 5 打 zip + 上传 | ~5s（zip 约 200KB） |
| Step 6 卡片发送 | ~2s |
| **合计** | **~18s** |

> 30 秒一气呵成是工作流上限；网络抖动 / API 限流时可能拉到 60 秒，仍属正常。
