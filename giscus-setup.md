Giscus 是一个基于 GitHub Discussions 的评论系统，免费、无广告、支持 Markdown。

## 配置步骤

### 1. 准备条件
- GitHub 仓库（已有：openduran/duranblog）
- 开启 GitHub Discussions 功能

### 2. 开启 Discussions
1. 访问 https://github.com/openduran/duranblog/settings
2. 找到 **Features** 部分
3. 勾选 **Discussions**
4. 点击 Save

### 3. 获取 Giscus 配置
1. 访问 https://giscus.app/zh-CN
2. 填写信息：
   - 仓库：openduran/duranblog
   - 页面 ↔️ Discussions 映射： pathname
   - Discussion 分类：General
   - 主题：根据网站主题选择（如 light / dark / preferred_color_scheme）
   - 语言：zh-CN
3. 复制生成的代码片段（data-repo, data-category 等）

### 4. Hugo 集成
将 Giscus 提供的代码添加到 Hugo 模板中，或配置主题支持。

对于 PaperMod 主题，在 config.toml 中添加：
[params]
  [params.giscus]
    repo = "openduran/duranblog"
    repoId = "..."  # 从 Giscus 获取
    category = "General"
    categoryId = "..."  # 从 Giscus 获取
    mapping = "pathname"
    reactionsEnabled = "1"
    emitMetadata = "0"
    inputPosition = "bottom"
    theme = "preferred_color_scheme"
    lang = "zh-CN"
    loading = "lazy"
