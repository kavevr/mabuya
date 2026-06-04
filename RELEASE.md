# Mabuya 主题重写总结

## 改了哪些东西

- 将站点配置从 `config.toml` 迁移为新的 `zola.toml`。
- 按最新 Zola 模板结构重组页面：
  - 新增 `templates/section.html`
  - 新增 `templates/taxonomy_list.html`
  - 新增 `templates/taxonomy_single.html`
  - 调整 `templates/base.html`
  - 调整 `templates/page.html`
  - 调整 `templates/index.html`
- 删除了旧的 taxonomy 专用目录模板：
  - `templates/tags/list.html`
  - `templates/tags/single.html`
- 删除了已不再需要的 `templates/partials/head.html`，把 head 内容并入 `base.html`。
- 更新了主题最低 Zola 版本要求：
  - `theme.toml` 的 `min_version` 提升到 `0.22.1`
- 修复了示例内容的 front matter 格式：
  - `content/blog/test.md` 从 TOML 格式修正为合法写法
  - 新增 `content/blog/_index.md`，让 `content/blog/` 成为正式 section
- 补充并更新了 README 中的旧配置说明：
  - 把 `config.toml` 改成 `zola.toml`
  - 把最低 Zola 版本说明更新到 `0.22.1`

## 优化了哪些东西

- 使用了 Zola 最新推荐的模板变量和页面字段：
  - `page.summary`
  - `page.updated`
  - `page.reading_time`
  - `page.word_count`
  - `section.pages`
  - `paginator`
  - 通用 taxonomy 变量 `taxonomy` / `term`
- 首页、section 页、taxonomy 页现在都能复用更统一的布局逻辑。
- 文章页增加了更完整的元信息展示：
  - 作者
  - 发布日期
  - 更新日期
  - 阅读时长
  - 字数
- 列表页摘要策略改进：
  - 优先使用 `page.summary`
  - 再回退到 `page.description`
  - 最后才用正文截断
- 导航链接改成使用 Zola 内链和 taxonomy 解析方式，避免硬编码路径。
- 404 页面改为指向首页的 Zola 内链，兼容 base_url 或子路径部署。
- 回到顶部按钮改成标准 `<button>` + 事件监听，去掉内联 `onclick`。
- 样式层顺手修复了一个 footnote 的拼写错误：
  - `top: -0.2.5em` 修正为 `top: -0.25em`
- `zola check` 的外链检查做了更现实的配置：
  - `external_level = "warn"`
  - 保留内部链接校验为错误级别

## 验证结果

- `zola check` 通过
- `zola build` 通过

## 备注

- 这次重写以“兼容最新 Zola 语法和模板结构”为目标，没有改变主题的整体视觉风格。
- 如果你要继续升级，我建议下一步优先做首页和文章页的视觉重构，再考虑把示例内容整理成更真实的博客数据。
