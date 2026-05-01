# Templates

Three Handlebars-style templates the SKILL.md workflow renders into final artifacts. Despite the `.hbs` extension, **agents should do plain string interpolation** (no Handlebars engine required) — the `{{var}}` and `{{#each}}` syntax is just a familiar marker for "replace me".

| 文件 | 渲染目标 | 由 SKILL.md 哪一步消费 |
|---|---|---|
| `proposal.md.hbs` | 方案云文档 markdown | Step 2 → 喂给 `lark-cli docs +create --markdown` |
| `kickoff.md.hbs` | 代码骨架 zip 顶层 `KICKOFF.md` | Step 5.2 → 写到 zip 根目录 |
| `card-dispatch.json.hbs` | 飞书 interactive card payload | Step 6 → 喂给 `lark-cli im +messages-send --content` |

## 上下文变量

完整列表见 [`../SKILL.md`](../SKILL.md) Step 2 与 Step 5.2。常用：

- `merchant_name` `industry` `industry_intro` `language` `target_launch` `today` `days_left`
- `scenes` (`{key, label, primary_skill, support_skills_joined}` 数组)
- `scenes_labels_joined`（如 `"付款码、预下单"`，给 header 用）
- `skill_summaries` (`{name, description, path}` 数组，来源是 `vendor/sqb-payment-skills/sqb-api-skills/<skill>/SKILL.md` 的 frontmatter + 一级标题)
- `owner_name` `owner_open_id` `doc_url` `base_url` `zip_url` `main_row_id` `n_tasks`
- `selected_skills_paths`（在 `kickoff.md.hbs` 里展开 zip 内每个 skill 目录）
- `language_codeblock_lang`、`is_java`（仅 `kickoff.md.hbs` 用，决定 self-test 代码块）

## 渲染时务必

1. **HTML/JSON 转义 `merchant_name`**：商户名可能含 `"` `<` `>` `&`，写到 JSON 卡片里要 `\"`。markdown 里相对宽松。
2. **字符串拼接而非模板引擎**：`{{#each}}` `{{#if}}` 块需要你自己用循环/条件展开，不要假设 Handlebars 在跑。
3. **`days_left ≤ 0`**：把"剩 N 天"改成"已过期"，避免出现负数尴尬。
4. **`scenes_labels_joined` 的连接符**：用顿号（`、`），不要逗号。
