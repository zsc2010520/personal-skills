# personal-skills

个人 AI Agent 技能库，收集和分享可复用的技能与工作流。

## 已发布技能

| 技能 | 用途 |
| --- | --- |
| [xhs-browser-publish](skills/xhs-browser-publish/SKILL.md) | 读取已准备的 JSON 文案，通过 Codex 内置浏览器上传素材并保存小红书草稿 |
| [xhs-browser-note-count](skills/xhs-browser-note-count/SKILL.md) | 按自然月统计小红书账号图文、视频和总发布数量，保存逐条明细 |

## 安装

将需要的 `skills/<技能名>` 文件夹复制到本机 `~/.codex/skills/` 目录，保留其中的子目录和参考文件。

## 使用条件

使用已登录小红书的 Codex 内置浏览器，并遵循当前环境提供的浏览器工具文档。浏览器 API 和平台页面可能变化，运行时核验当前可用能力。

发布技能需要用户提供已准备好的 JSON 文案与素材路径，默认保存为草稿。统计技能接受账号主页或账号 CSV 清单。输入与输出路径均可由用户指定。
