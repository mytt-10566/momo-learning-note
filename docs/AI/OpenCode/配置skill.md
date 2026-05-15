用户目录下全局可用的skill：

~/.config/opencode/skills

也支持兼容路径：

~/.claude/skills/<name>/SKILL.md
~/.agents/skills/<name>/SKILL.md

SKILL.md 文件格式
每个 SKILL.md 必须以 YAML frontmatter 开头：

```
---
name: my-skill
description: 这个 skill 做什么的描述（1-1024字符）
---

## 具体指令内容

在这里写 skill 的详细指令...
```

命名规则：

1-64 个字符
只能用小写字母、数字和单个连字符
不能以 - 开头或结尾
目录名必须和 name 字段一致

在项目根目录下创建以下路径的文件：

.opencode/skills/<skill-name>/SKILL.md

也支持兼容路径：

.claude/skills/<name>/SKILL.md
.agents/skills/<name>/SKILL.md

OpenCode 会从当前工作目录向上查找到 git 仓库根目录，加载所有匹配的 skills/*/SKILL.md。