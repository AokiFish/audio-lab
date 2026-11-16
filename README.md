# Audio Lab 开发文档包

给任意 AI 编程 agent 使用的项目文档,不绑定某个工具(Claude Code、Codex、Cursor、Gemini CLI、Copilot agent 等都可以)。目标:让能力一般的 agent 也能分里程碑做出可验证的高质量结果。

## 怎么用

1. 新建一个空仓库,把本包内容放进根目录(`AGENTS.md`、`PROGRESS.md`、`docs/`)。
2. 在仓库根目录启动你选的 agent。
3. 一次只发一个里程碑的指令(见下)。每个里程碑结束后,你检查、听、勾选 `PROGRESS.md` 里的"需要人工确认",再开始下一个。

### 兼容不同工具

- 所有文档都是普通 Markdown,不依赖任何工具的专有功能。
- 不同工具自动读取的规则文件名不同(例如 `AGENTS.md`、`CLAUDE.md`、`GEMINI.md`,或 IDE 里的 rules 目录)。**不确定时不用查**:下面的开场指令第一句就是"先读 `AGENTS.md`",任何能读文件的 agent 都会照做。
- 如果你想让工具自动加载,把 `AGENTS.md` 的内容复制(或建立符号链接)到该工具约定的文件名即可,只维护一份内容。
- 文档里的命令都是标准的 `make` + Python + Node,不要求 agent 有特殊能力。agent 至少需要:读写文件、运行 shell 命令。

## 每个里程碑的开场指令(复制使用)

```
先读 AGENTS.md 和 PROGRESS.md。
现在做里程碑 M0(把 M0 换成当前里程碑),按 docs/07-milestones.md 里的范围和验收标准执行。
开始之前,先读该里程碑用到的文档(见 AGENTS.md 的文档索引),然后用 5 行以内说明你的实施计划。
完成后更新 PROGRESS.md,列出需要我人工确认的项,然后停下,不要开始下一个里程碑。
遇到文档没写的决策、需要新增依赖或需要改契约,先停下来问我。
```

## 先验证文档本身(强烈建议)

用一个能力较弱的 agent 或较省钱的模式只跑 M0–M2,观察:

- 它在哪里反复卡住或来回问你?→ 那里文档不够具体,补上
- 它在哪里"自由发挥"(加了没要求的东西)?→ 在 AGENTS.md 里加一条禁止
- 它是否真的跑了 `make check`、`make eval`,还是只说"已完成"?→ 要求它贴出命令输出

## 你(负责人)需要做的事

agent 听不到声音,以下只有你能做:

- 把 5–10 首真实 Suno 歌曲放进 `tests/golden/real/`(该目录不提交)
- M1、M2:看分析数值和频谱图是否合理
- M3、M4、M5:同响度 A/B 试听并按 `docs/02-audio-quality.md` 第 6 节模板记录
- M4:根据听感校准预设,让 agent 把结果写回 `recipes/*.json` 和 `docs/03-recipe-schema.md`

## 文档里哪些是"起点值"

`docs/03-recipe-schema.md` 里的 EQ 频率/增益/压缩参数、自适应规则的触发阈值、`docs/02` 里的护栏容差,都是合理的初始设定,不是经过大量样本验证的结论。M4 的人工试听就是用来校准它们的。

## 文件清单

```
AGENTS.md                      硬规则(每次会话都读)
PROGRESS.md                    当前状态、版本、待人工确认项
docs/01-product.md             产品范围与"不做什么"
docs/02-audio-quality.md       指标、安全护栏、测试音频、评测、人工验证
docs/03-recipe-schema.md       处理方案 schema、预设、自适应规则
docs/04-architecture.md        架构、任务、存储、限制
docs/05-api-contract.md        接口契约与错误码
docs/06-ui-spec.md             前端状态机、布局、令牌、文案
docs/07-milestones.md          M0–M6 的范围与验收标准
docs/08-pitfalls.md            已知陷阱
```
