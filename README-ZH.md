# PaperBell

## 当前功能

- 监控文件创建，从 yaml frontmatter 中提取 `institute` 信息用于创建笔记
- 为 `Quickadd` 集成提供全局 API
- 检索信息后显示或自动创建笔记
- 支持自定义机构笔记模板

## 模板变量

您可以在机构笔记模板中使用以下模板变量：

| 变量 | 描述 | 示例 |
|----------|-------------|---------|
| {{paperbell.name}} | 机构全称 | 哈佛大学 |
| {{paperbell.abbr}} | 机构缩写 | HU |
| {{paperbell.aliases}} | 替代名称列表 | - Harvard<br>- Harvard College |
| {{paperbell.website}} | 机构网站 URL | https://harvard.edu |
| {{paperbell.location}} | 地理坐标 | [42.3744, -71.1169] |
| {{paperbell.logo}} | 机构 logo URL | https://example.com/logo.png |
| {{paperbell.tags}} | 标签列表 | - university |

### 使用模板

1. 在您的 vault 中创建模板文件（例如：`templates/institution.md`）
2. 在模板文件中使用上述变量
3. 在插件设置中设置模板路径
4. 插件将在创建新的机构笔记时使用您的模板

示例模板：

````markdown
---
name: {{paperbell.name}}
abbr: {{paperbell.abbr}}
aliases: 
{{paperbell.aliases}}
website: {{paperbell.website}}
location: {{paperbell.location}}
tags: 
{{paperbell.tags}}
---

# {{paperbell.name}}

## 概览

[Website]({{paperbell.website}})

## 相关学者

```dataviewjs
let name = dv.current().name

dv.table(["姓名", "职称", "网站", "邮箱"],
dv.pages(`#scholar`)
  .where(b => b.institute.includes(name))
  .map(b => [b.file.link, b.title, ("[🔗]("+b.website+")"), b.email])
  .sort(b => b.paper_date, 'desc')
)
```
````
