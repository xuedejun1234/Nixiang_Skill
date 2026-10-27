# Nixiang Skill

个人使用的 Agent Skills 集合。每个顶层目录都是一个可独立使用的 Skill，入口为 `SKILL.md`。

## 目录

| Skill | 用途 | 来源 |
|---|---|---|
| [`mini-game-reverse`](mini-game-reverse/SKILL.md) | 小游戏发布包的资源定位、脚本逆向、工程恢复与验证 | 本仓库整理；[案例与工具说明](mini-game-reverse/references/case-study.md) |
| [`reverse-engineer-anything`](reverse-engineer-anything/SKILL.md) | 使用 REA 分析发布产物和运行行为 | [morluto/rea](https://github.com/morluto/rea) |
| [`find-skills`](find-skills/SKILL.md) | 查找和安装 Skill | [vercel-labs/skills](https://github.com/vercel-labs/skills) |
| [`planning-with-files`](planning-with-files/SKILL.md) | 将多步骤任务的计划和进度保存到文件 | [StarQuant/planning-with-files-skill](https://github.com/StarQuant/planning-with-files-skill) |
| [`ponytail`](ponytail/SKILL.md)、[`ponytail-audit`](ponytail-audit/SKILL.md)、[`ponytail-debt`](ponytail-debt/SKILL.md)、[`ponytail-gain`](ponytail-gain/SKILL.md)、[`ponytail-help`](ponytail-help/SKILL.md)、[`ponytail-review`](ponytail-review/SKILL.md) | 简化代码及其审查、审计和辅助命令 | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) |

## 使用

将需要的 Skill 目录复制到所用 Agent 的 Skill 搜索目录；在任务中明确提及 Skill 名称和目标文件路径。例如：`使用 mini-game-reverse 分析这个小游戏发布包，原包位于 ...`。

第三方 Skill 保留原作者署名；对应目录中的 `LICENSE` 是上游 MIT 许可证。REA Skill 是操作说明，不包含 REA 服务或其他工具的安装与授权。
