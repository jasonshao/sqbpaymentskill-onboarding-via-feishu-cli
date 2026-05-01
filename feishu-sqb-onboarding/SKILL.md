---
name: feishu-sqb-onboarding
version: 0.1.0
description: "收钱吧支付接入 onboarding 工作流：基于 6 字段输入（接入主体 / 行业 / 支付场景 / 开发语言 / 上线日 / 对接负责人），AI agent 调用 lark-cli 顺序产出方案云文档 + 多维表进度 + 代码骨架 zip + IM 任务派发卡片，30 秒内把支付接入流程搬进飞书工作台。触发词：收钱吧接入 / sqb 接入 / 接付款码 / 接预下单 / sqb-init / /sqb-onboarding。"
metadata:
  requires:
    bins: ["lark-cli"]
tags: [onboarding, payment, shouqianba, sqb, workflow, lark]
---

# 收钱吧支付接入 onboarding 工作流

**CRITICAL — 开始前 MUST 完成两件事：**
1. **用 Read 工具读取本目录 [`./templates/`](./templates/) 下的所有 `.hbs` 模板文件**（proposal / kickoff / card-dispatch）以及 [`./data/scene-skill-map.json`](./data/scene-skill-map.json) 与 [`./data/industry-copy.json`](./data/industry-copy.json)
2. **确认 [`vendor/sqb-payment-skills/`](../vendor/sqb-payment-skills) 已通过 git submodule 拉取就位**（`git submodule update --init`）；本工作流引用其 `sqb-api-skills/<skill-name>/SKILL.md` 与 `reference/` 目录

## 适用场景

任何"把一次新的收钱吧支付接入流程在飞书工作台里立项"的诉求。常见提法：
- "我要给 XX 餐饮接收钱吧付款码，Java 栈"
- "深圳 XX 接 sqb 预下单 + 退款，5/30 上线，对接人 @张三"
- "新增一个 sqb-init：零售客户、Python、要付款码 + 预下单"
- "/sqb-onboarding"、"/sqb-init"
- "把收钱吧接入流程开个进度表 + 方案文档 + 派单"
- "收钱吧 ISV 接入立项"

**不适用**：
- 已存在的项目想问"我刚才那个进度表在哪" → 用 `lark-cli base +base-get` 直接查
- 只想问 SQB API 单接口怎么调 → 直接读 `vendor/sqb-payment-skills/sqb-api-skills/<skill>/SKILL.md`
- 售前报价 / 全来店报价 → 那是另一个 skill（`quanlaidian-quotation-skill`）

## 前置条件

### 1. lark-cli 已认证为 user 身份

```bash
lark-cli auth status     # 必须返回 tokenStatus: valid 或 needs_refresh（自动续）
lark-cli auth login --recommend   # 缺 scope 时调
```

需要的 scope（`auth status` 已包含的应当全部满足；缺则用 `lark-cli auth check --scope ...`）：

| 子命令 | 所需 scope |
|---|---|
| `docs +create` | `docx:document:create` |
| `base +base-create` `+table-create` `+field-create` `+record-batch-create` | `base:app:create` `base:table:create` `base:field:create` `base:record:create` |
| `drive +upload` | `drive:file:upload` |
| `im +messages-send` | `im:message`（user 身份）或 `im:message:send_as_user`（推荐） |

### 2. 子模块就位

工作目录顶部应能找到 `vendor/sqb-payment-skills/sqb-api-skills/`；若缺：

```bash
git submodule update --init --recursive
```

### 3. 凭据隔离

**任何 base_token / chat_id / user open_id 不要硬编码进 SKILL 或 templates**。从用户输入或环境变量取，输出之前用 `lark-cli` 自身做权限校验。

## 6 字段输入规约

工作流入参（Field Schema）：

| 字段 | 类型 | 必填 | 取值 |
|---|---|:--:|---|
| `merchant_name` | string | ✓ | 接入主体全称，如"深圳市XX餐饮管理有限公司" |
| `industry` | enum | ✓ | `餐饮` / `零售` / `教培` / `服务` / `其他` |
| `scenes` | array<enum> | ✓ | 取值集合：`pay`（付款码 B 扫 C）/ `precreate`（预下单 C 扫 B）/ `refund`（退款）/ `activate-checkin`（终端激活+签到）/ `notify`（异步通知） |
| `language` | enum | ✓ | `java` / `python`（其他语言不打 zip，仅出文档） |
| `target_launch` | date (ISO 8601) | ✓ | 如 `2026-05-30` |
| `owner` | string | ✓ | 飞书 open_id（`ou_*`）或邮箱；用于 Bitable person 字段与 IM `--user-id` |

### 字段提取策略（agent 做）

1. **从用户原话直接抽取**能确定的字段；未提到或歧义的，**逐一**反问，每次只问一个未知字段
2. `industry` 若用户只说行业关键词（如"火锅店"），按下表降到一档：

   | 用户说 | 映射 |
   |---|---|
   | 餐饮 / 火锅 / 茶饮 / 快餐 / 正餐 | `餐饮` |
   | 零售 / 便利店 / 超市 | `零售` |
   | 教培 / 培训 / 学校 | `教培` |
   | 美业 / 健身 / 服务 / 维修 | `服务` |
   | 其它一切 | `其他` |

3. `scenes` 若用户说"小程序支付"或"H5 支付"等**入口词**，提示用户："这些是支付入口，本工作流按 SQB API 维度建模——你的接入主要是付款码（B 扫 C）还是预下单（C 扫 B）？"
4. `owner` 若用户给的是 @姓名，先用 `lark-cli contact +search-user --query "<name>"` 解析为 open_id 再写入

### 立即停手的输入异常

- `target_launch` 早于今天 → 反问是否输错
- `scenes` 为空 → 必须至少选一个
- `owner` 解析为多个候选用户且模糊 → 反问消歧，不要瞎选

## 工作流总图

```
6 字段 ─┬─► [Step 1] 场景 → SKILL 映射（读 data/scene-skill-map.json）
        ├─► [Step 2] 渲染方案 markdown（templates/proposal.md.hbs + industry-copy.json + 命中 SKILL.md 摘要）
        ├─► [Step 3] lark-cli docs +create  ──► doc_url
        ├─► [Step 4] lark-cli base +base-create + +table-create ×2 + +record-batch-create
        │       ──► base_token / 主行 record_id / 子表 record_ids
        ├─► [Step 5] 打包 zip（reference/<lang>/ 整拷 + 渲染 KICKOFF.md）→ lark-cli drive +upload ──► zip_url
        └─► [Step 6] 渲染 card-dispatch.json + lark-cli im +messages-send（@owner，含 doc_url / base_url / zip_url）
                                                          ──► message_id（可选用作后续 thread reply 入口）
```

**全过程串行**（用户感受是 30 秒一气呵成的状态条；并行扇出会让飞书侧 base/doc/drive 的 throttling 更难控）。

**任何一步失败，立即停手**——不要试图回滚已成功的资源；把已得资源链接 + 失败步与原因清晰回报给用户，让人决策。

---

## Step 1 — 场景到 SKILL 的映射

读 [`./data/scene-skill-map.json`](./data/scene-skill-map.json)；其结构：

```json
{
  "pay":               { "skills": ["sqb-pay", "sqb-signing", "sqb-status-parsing", "sqb-polling"], "label": "付款码（B 扫 C）" },
  "precreate":         { "skills": ["sqb-precreate", "sqb-query", "sqb-signing", "sqb-status-parsing", "sqb-polling", "sqb-callback-verify"], "label": "预下单（C 扫 B）" },
  "refund":            { "skills": ["sqb-refund", "sqb-signing", "sqb-status-parsing"], "label": "退款" },
  "activate-checkin":  { "skills": ["sqb-activate", "sqb-checkin", "sqb-signing"], "label": "终端激活与签到" },
  "notify":            { "skills": ["sqb-notify", "sqb-callback-verify"], "label": "异步通知" }
}
```

**算法**：
1. 取用户勾选的 `scenes` 数组
2. 对每个场景查表，**取并集**得到 `skill_set`（去重后的 sqb-skill 列表）
3. 对 `skill_set` 中每个 `s`，读取 `vendor/sqb-payment-skills/sqb-api-skills/<s>/SKILL.md` 的 frontmatter `description` 和正文一级标题 → 得到该 skill 的"摘要 + 路径"
4. 该信息进入 Step 2 的 proposal 模板与 Step 4 的 Bitable 子表

## Step 2 — 渲染方案 markdown

读 [`./templates/proposal.md.hbs`](./templates/proposal.md.hbs) + [`./data/industry-copy.json`](./data/industry-copy.json)。模板上下文（Handlebars 风格变量）：

```
{
  "merchant_name": "...",
  "industry": "餐饮",
  "industry_intro": "<industry-copy.json[industry]>",
  "scenes": [{ "key": "pay", "label": "付款码（B 扫 C）" }, ...],
  "language": "java",
  "target_launch": "2026-05-30",
  "today": "<ISO 8601 date>",
  "skill_summaries": [
    { "name": "sqb-pay", "description": "...", "path": "vendor/sqb-payment-skills/sqb-api-skills/sqb-pay/SKILL.md" },
    ...
  ]
}
```

**Handlebars 不强求依赖**：你（agent）可以用任何字符串模板手段（Python `str.format`、JS 模板字符串、纯字符串拼接）渲染，只要产出符合 proposal.md.hbs 描述的最终 markdown 即可。**不要为了"用 Handlebars"额外装包**。

## Step 3 — 创建方案云文档

```bash
# 把渲染好的 markdown 写到 cwd 内的相对路径（lark-cli 不接受 /tmp 等绝对路径）
# 或用 stdin: cat proposal.md | lark-cli docs +create --title "..." --markdown -
lark-cli docs +create \
  --title "《{{merchant_name}}》收钱吧支付接入方案" \
  --markdown ./proposal-{{timestamp}}.md
```

**输出捕获**：解析 stdout JSON，取 `data.url` 作为 `doc_url`，`data.document_id` 作为 `doc_id`。

> ⚠️ **路径限制**：`--markdown @<file>` 与 `--content @<file>`、`--file <file>` 都强制相对路径且必须在 cwd 之内（命令本身做 path 校验）。绝对路径如 `/tmp/...` 会被拒。**临时文件先 `cd` 到工作目录或用相对子路径**；agent 实践上推荐 `./` 前缀或纯文件名。

## Step 4 — 创建 Bitable 与任务行

### 4.1 创建 Base

```bash
lark-cli base +base-create \
  --name "《{{merchant_name}}》收钱吧接入" \
  --time-zone "Asia/Shanghai"
```

捕获 `data.app.app_token` 作为 `base_token`。

### 4.2 创建主表"接入登记"

```bash
lark-cli base +table-create \
  --base-token {{base_token}} \
  --name "接入登记" \
  --fields '[
    {"type":"text","name":"接入主体"},
    {"type":"select","name":"行业","multiple":false,"options":[{"name":"餐饮"},{"name":"零售"},{"name":"教培"},{"name":"服务"},{"name":"其他"}]},
    {"type":"select","name":"支付场景","multiple":true,"options":[{"name":"付款码"},{"name":"预下单"},{"name":"退款"},{"name":"终端激活与签到"},{"name":"异步通知"}]},
    {"type":"select","name":"开发语言","multiple":false,"options":[{"name":"Java"},{"name":"Python"},{"name":"其他"}]},
    {"type":"datetime","name":"预计上线日","style":{"format":"yyyy-MM-dd"}},
    {"type":"user","name":"对接负责人","multiple":false},
    {"type":"select","name":"状态","multiple":false,"options":[{"name":"待启动"},{"name":"资料补齐中"},{"name":"联调中"},{"name":"上线"},{"name":"已完成"}]},
    {"type":"text","name":"方案文档","style":{"type":"url"}},
    {"type":"text","name":"代码骨架","style":{"type":"url"}}
  ]'
```

捕获 `data.table_id` 作为 `main_table_id`。

> 字段 schema 用 lark-cli 的 shortcut 形式（`type` 为字符串 + 顶层 `name`），不要用旧版数字 type / `field_name` / `property`；详见 [`lark-base-shortcut-field-properties.md`](https://github.com/larksuite/cli/blob/main/skills/lark-base/references/lark-base-shortcut-field-properties.md)。

### 4.3 写主表行

```bash
lark-cli base +record-batch-create \
  --base-token {{base_token}} --table-id {{main_table_id}} \
  --json '{
    "fields": ["接入主体","行业","支付场景","开发语言","预计上线日","对接负责人","状态","方案文档","代码骨架"],
    "rows": [[
      "{{merchant_name}}",
      "{{industry}}",
      [{{scenes_labels_json_array}}],
      "{{language_label}}",
      "{{target_launch}} 00:00:00",
      [{"id":"{{owner_open_id}}"}],
      "待启动",
      "{{doc_url}}",
      null
    ]]
  }'
```

捕获返回 `data.record_id_list[0]` 作为 `main_row_id`（Step 5.5 回填代码骨架链接用）。

> CellValue 形态见 [`lark-base-cell-value.md`](https://github.com/larksuite/cli/blob/main/skills/lark-base/references/lark-base-cell-value.md)：URL 字段就是字符串、user 字段是 `[{id:"ou_..."}]`、多选是字符串数组、datetime 是 `"yyyy-MM-dd HH:mm:ss"`。空值用 `null`。

### 4.4 创建子表"接入任务" + 批插

```bash
lark-cli base +table-create \
  --base-token {{base_token}} \
  --name "接入任务" \
  --fields '[
    {"type":"text","name":"任务"},
    {"type":"text","name":"接口SKILL"},
    {"type":"user","name":"负责人","multiple":false},
    {"type":"select","name":"状态","multiple":false,"options":[{"name":"待办"},{"name":"进行中"},{"name":"已完成"},{"name":"阻塞"}]},
    {"type":"text","name":"备注"}
  ]'
```

捕获 `task_table_id`，然后**对 Step 1 算出的 `skill_set` 逐项**生成一行 cell，整体一次 `+record-batch-create`（最多 200 行/次，足够本场景）：

```bash
lark-cli base +record-batch-create \
  --base-token {{base_token}} --table-id {{task_table_id}} \
  --json '{
    "fields": ["任务","接口SKILL","负责人","状态","备注"],
    "rows": [
      ["对接 sqb-pay 接口", "sqb-pay", [{"id":"{{owner_open_id}}"}], "待办", null],
      ["对接 sqb-signing 接口", "sqb-signing", [{"id":"{{owner_open_id}}"}], "待办", null]
      // ... 对 skill_set 中每个 skill 重复
    ]
  }'
```

### 4.5 base_url 拼接

```
base_url = "https://sqb.feishu.cn/base/{{base_token}}"
```

（domain 视租户而定；从 `auth status` 输出推不出来时，回落到 `https://feishu.cn/base/<token>`，让用户自己点）

## Step 5 — 打包代码骨架并上传

### 5.1 选源

按 `language` + `skill_set`：

```
sources = [
  vendor/sqb-payment-skills/sqb-api-skills/<skill>/reference/   for each skill in skill_set
] + [
  vendor/sqb-payment-skills/sqb-api-skills/shared-reference/
]
```

只挑当前语言（`*.java` 或 `*.py`）。

### 5.2 渲染 KICKOFF.md（顶部 README）

读 [`./templates/kickoff.md.hbs`](./templates/kickoff.md.hbs)，注入 6 字段 + skill_set 的接口清单 + sqb-payment-skills 主 README 的"快速上手"两段引用。

### 5.3 打 zip

```bash
# 必须在 cwd 之内；不要 /tmp 绝对路径，--file 校验会拒
zip_path="./{{merchant_name_slug}}-sqb-{{language}}-{{timestamp}}.zip"
staging="./.staging-{{timestamp}}"
( cd "$staging" && zip -r "../{{merchant_name_slug}}-sqb-{{language}}-{{timestamp}}.zip" . )
```

### 5.4 上传

```bash
lark-cli drive +upload \
  --file "$zip_path" \
  --name "《{{merchant_name}}》收钱吧接入代码骨架.zip"
```

捕获 `data.file_token`，拼出 `zip_url = https://sqb.feishu.cn/file/{{file_token}}`（同 base_url 域名规则）。

### 5.5 回填 Bitable 主行的"代码骨架"字段

```bash
lark-cli base +record-batch-update \
  --base-token {{base_token}} --table-id {{main_table_id}} \
  --json '{
    "record_id_list": ["{{main_row_id}}"],
    "patch": { "代码骨架": "{{zip_url}}" }
  }'
```

## Step 6 — 派发 IM 卡片给 owner

读 [`./templates/card-dispatch.json.hbs`](./templates/card-dispatch.json.hbs)，渲染产出 interactive card 的 content JSON：

```bash
# ⚠️ --content 不支持 @file 也不支持 - stdin；必须 inline JSON。用 shell 命令替换：
lark-cli im +messages-send \
  --user-id {{owner_open_id}} \
  --msg-type interactive \
  --content "$(cat ./dispatch-card-{{timestamp}}.json)" \
  --idempotency-key "sqb-onboarding-{{merchant_name_slug}}-{{today}}"
```

> `--idempotency-key` 透传到请求 body 的 `uuid` 字段；同 key 重发不会再创新消息。

**`--idempotency-key` 必填**：避免重复触发时给 owner 刷屏。

捕获 `data.message_id` 作为后续答疑线程的入口（可选，本 skill 不必再扇）。

## 输出格式（agent 给用户的最终回复）

```
## ✅ 收钱吧接入立项完成 — {{merchant_name}}

| 资产 | 链接 |
|---|---|
| 方案文档 | {{doc_url}} |
| 进度表 | {{base_url}} |
| 代码骨架 zip | {{zip_url}} |
| 派发卡片 | 已发给 @{{owner_name}}（消息 id: {{message_id}}） |

涵盖场景：{{scenes_labels_joined}}
开发语言：{{language_label}}
预计上线：{{target_launch}}（剩 {{days_left}} 天）

下一步建议：
1. 让 {{owner_name}} 在进度表里把"接入任务"子表的 {{n_tasks}} 个 SKILL 接口认领并填备注
2. 把 vendor/sqb-payment-skills/sqb-api-skills/<scene>/SKILL.md 喂给 Claude Code/Cursor 生成具体调用代码
3. 上线日前 1 周，跑 {{vendor/sqb-payment-skills/tests/}}（或自行补 e2e 测试）
```

## 错误处理 / 常见兜底

| 现象 | 处理 |
|---|---|
| `auth status` 显示 `needs_refresh` | 继续跑（lark-cli 自动续）；连续两次失败再调 `lark-cli auth login` |
| `base +base-create` 报权限不足 | 提示用户：`lark-cli auth login --domain base` 重新授权 |
| `drive +upload` 失败（>20MB 超时） | zip 一般 < 5MB；如真超，提示用户改用 `--folder-token` 指定个人空间根 |
| `docs +create` markdown 体积过大（> 500KB） | 拆成两份：方案 + 接口清单分开 |
| `im +messages-send --user-id` 找不到用户 | 调 `lark-cli contact +search-user --query "<name>"` 重解析；仍失败则提示用户改提供 open_id |
| 用户中途追加场景 | **不要**重新跑全流程；用 `+record-batch-create` 在子表追加新行，并 `+record-batch-update` 主行的"支付场景" multi-select |

## 权限表（汇总）

| 命令 | scope |
|---|---|
| `auth login --recommend` | （交互式批准全集） |
| `docs +create` | `docx:document:create` |
| `docs +update` | `docx:document:write_only` |
| `base +base-create` | `base:app:create` |
| `base +table-create` | `base:table:create` |
| `base +field-create` | `base:field:create` |
| `base +record-batch-create` | `base:record:create` |
| `base +record-batch-update` | `base:record:update` |
| `drive +upload` | `drive:file:upload` |
| `im +messages-send`（user） | `im:message.send_as_user` 或 `im:message` |
| `contact +search-user` | `contact:user:search` |

## 参考

### 本 skill 的依赖
- [`./templates/proposal.md.hbs`](./templates/proposal.md.hbs) — 方案文档模板
- [`./templates/kickoff.md.hbs`](./templates/kickoff.md.hbs) — 代码骨架顶部 README 模板
- [`./templates/card-dispatch.json.hbs`](./templates/card-dispatch.json.hbs) — IM 派发卡片 payload 模板
- [`./data/scene-skill-map.json`](./data/scene-skill-map.json) — 支付场景到 SKILL 的映射
- [`./data/industry-copy.json`](./data/industry-copy.json) — 5 行业开头话术

### 上游 SQB 接入知识
- [`vendor/sqb-payment-skills/README.md`](../vendor/sqb-payment-skills/README.md) — 上游总览（Apache-2.0）
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-pay/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-pay/SKILL.md) — 付款码
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-precreate/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-precreate/SKILL.md) — 预下单
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-refund/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-refund/SKILL.md) — 退款
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-activate/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-activate/SKILL.md) — 激活
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-checkin/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-checkin/SKILL.md) — 签到
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-notify/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-notify/SKILL.md) — 异步通知
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-query/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-query/SKILL.md) — 查询
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-cancel/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-cancel/SKILL.md) — 撤单
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-signing/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-signing/SKILL.md) — MD5 签名
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-status-parsing/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-status-parsing/SKILL.md) — 三层状态判定
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-polling/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-polling/SKILL.md) — 参数化轮询
- [`vendor/sqb-payment-skills/sqb-api-skills/sqb-callback-verify/SKILL.md`](../vendor/sqb-payment-skills/sqb-api-skills/sqb-callback-verify/SKILL.md) — RSA 回调验签

### lark-cli 内置依赖（不在本仓内）
- [`lark-shared`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、错误码（agent 应已加载）
- [`lark-base`](https://github.com/larksuite/cli/blob/main/skills/lark-base/SKILL.md) — base 字段类型与限制
- [`lark-doc`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — markdown 体积上限与格式约束
- [`lark-im`](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md) — interactive card schema
- [`lark-drive`](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md) — 上传与权限

## 示例对话

完整端到端样例见 [`./examples/`](./examples/) 目录。
