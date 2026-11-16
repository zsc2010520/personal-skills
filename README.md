# personal-skills

个人 AI Agent 技能库，收集和分享可复用的技能与工作流。

## 已发布技能

技能按领域分目录存放，同一领域的技能放在 `skills/<领域>/` 下。

### Adobe Experience Platform（`skills/aep/`）

| 技能 | 用途 |
| --- | --- |
| [aep-query-service-sql](skills/aep/aep-query-service-sql/SKILL.md) | 编写、审查和修复 Adobe Experience Platform Query Service 的 SQL：嵌套 XDM 字段、去重、写回数据集、增量加载和排错 |

### 小红书

| 技能 | 用途 |
| --- | --- |
| [xhs-browser-publish](skills/xhs-browser-publish/SKILL.md) | 读取已准备的 JSON 文案，通过 Codex 内置浏览器上传素材并保存小红书草稿 |
| [xhs-browser-note-count](skills/xhs-browser-note-count/SKILL.md) | 按自然月统计小红书账号图文、视频和总发布数量，保存逐条明细 |

## 安装

将需要的技能文件夹（包含 `SKILL.md` 的那一层，例如 `skills/aep/aep-query-service-sql`）复制到本机 `~/.codex/skills/` 目录，保留其中的子目录和参考文件。领域目录本身只用于归类，不需要复制。

其他 Agent 把文件夹复制到各自的技能目录即可。不支持技能目录的工具，可以直接把 `SKILL.md` 的内容贴进对话或系统提示词。

## 使用条件

`aep-query-service-sql` 是纯文本说明，不依赖特定模型、Agent 或浏览器。

小红书相关技能使用已登录小红书的 Codex 内置浏览器，并遵循当前环境提供的浏览器工具文档。浏览器 API 和平台页面可能变化，运行时核验当前可用能力。

发布技能需要用户提供已准备好的 JSON 文案与素材路径，默认保存为草稿。统计技能接受账号主页或账号 CSV 清单。输入与输出路径均可由用户指定。
