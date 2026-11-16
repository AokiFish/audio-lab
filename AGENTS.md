# AGENTS.md — Audio Lab(Suno 歌曲音质提升网站)

> 每次会话开始先读本文件和 PROGRESS.md。动手前按文末"文档索引"读对应文档。
> 本文件只放硬规则。细节在 docs/ 里,不要凭印象补全。

## 项目一句话

用户上传一首 AI 生成的歌曲(主要来自 Suno),选择"想要的声音",系统自动生成一条处理链(recipe),用户在同响度条件下 A/B 试听,满意后导出。
界面极简。复杂度在后端,专业用户可在"处理详情"里看到并编辑同一份 recipe。

## 工作方式(最重要)

1. **一次只做一个里程碑**(见 docs/07-milestones.md)。做完、验收、更新 PROGRESS.md 后**停下**,等人确认再开始下一个。
2. **只做里程碑里写了的东西**。不要顺手加效果、预设、页面、登录、数据库迁移框架等。想加就写进 PROGRESS.md 的"建议"栏,不要实现。
3. **契约先行**。要改 recipe schema 或 API:先改 docs/03 或 docs/05,再改 pydantic 模型,再重新生成 `api/openapi.json` 和前端类型,最后改实现。
4. **你听不到声音**。任何"听起来更好"的判断都不能由你自己下结论。你只能用 docs/02 里的客观指标和不变量来验证,并把"需要人工试听确认"的项留给人(不要自己打勾)。
5. **不允许假实现**。不得用 mock、占位或原样返回音频来冒充处理结果。做不到就写明做不到,并记录在 PROGRESS.md。
6. **不确定就停下来问**:需求冲突、需要新增依赖、需要改契约、文档没写的决策、测试通不过又找不到原因。写清楚问题和你的两个备选方案,再停。

## 技术栈(已定,不要更换;新增依赖先问)

| 部分 | 选择 |
|---|---|
| 后端 | Python(当前稳定版 ≥3.11)、FastAPI、pydantic v2、uvicorn |
| 音频处理 | pedalboard(DSP 节点)、numpy、scipy、pyloudnorm(响度)、ffmpeg(解码/编码) |
| 任务 | MVP:进程内线程池 + SQLite 任务表。不引入 Redis/Celery(见 docs/04) |
| 前端 | TypeScript、React、Vite、纯 CSS(CSS 变量 + CSS Modules)。**不引入 UI 组件库**。状态用 React 内置(useReducer/useContext) |
| 前端音频 | Web Audio API(A/B 试听) |
| 测试 | 后端 pytest;前端 vitest + Testing Library;契约用 openapi 生成的类型 |

具体版本在 M0 创建 `pyproject.toml` / `package.json` 时锁定,并把版本号记录到 PROGRESS.md。

## 仓库结构

```
api/                 # FastAPI 应用、音频引擎、测试
  app/
    main.py
    routes/          # 只做参数校验和调用 service
    services/        # 业务逻辑
    audio/           # 纯音频代码:analyze.py render.py nodes/ metrics.py
    models/          # pydantic 模型 = 契约的唯一来源
  tests/
  openapi.json       # 生成物,提交到仓库
web/
  src/
    copy.zh.ts       # 所有用户可见中文文案集中在这里
    api/             # 由 openapi.json 生成的类型 + 薄封装
    screens/ components/ audio/ styles/
recipes/             # 预设 recipe JSON(goals)
tests/golden/        # 固定测试音频(合成)+ 预期指标
tools/               # 生成测试音频、跑评测的脚本
docs/                # 本文档集
reports/             # make eval 的输出(不提交)
Makefile
```

## 常用命令(M0 必须全部可用)

```
make setup     # 安装依赖
make dev       # 同时启动 api 和 web
make check     # lint + 类型检查 + 全部测试 + 前端构建。提交前必须通过
make fixtures  # 生成合成测试音频到 tests/golden/
make eval      # 对所有 goals × fixtures 跑评测,输出 reports/eval.md
make openapi   # 重新生成 api/openapi.json 和前端类型
```

## 完成的定义(每个里程碑都适用)

- [ ] `make check` 通过
- [ ] 里程碑里的验收标准逐条满足,并在 PROGRESS.md 里写明"怎么验证的"
- [ ] 涉及音频处理的改动:`make eval` 全部不变量通过,且没有回退
- [ ] 新代码有测试(音频代码必须有数值断言,不能只测"不报错")
- [ ] 文档与实现一致(改了行为就改文档)
- [ ] PROGRESS.md 已更新,"需要人工确认"栏列出了人该听/看什么

## 代码规范

- 代码、注释、提交信息用英文;用户可见文案用中文,集中在 `web/src/copy.zh.ts`
- 前端不出现 EQ、Compressor、LUFS 等术语,除非在"处理详情"面板里
- 音频数组统一 `float32`,形状 `(channels, samples)`,采样率在 `AudioBuffer` 数据类里显式携带
- 所有外部命令(ffmpeg)用 `subprocess.run([...], shell=False)`;文件名一律用服务端生成的 id,不信任用户文件名
- 不吞异常。错误用 docs/05 里的错误码返回

## 安全与隐私(不可协商)

- 上传校验:大小、时长、真实格式(不只看扩展名)、解码成功
- 用户音频默认 24 小时后自动删除(见 docs/04),删除接口必须可用
- 日志里不记录音频内容或原始文件名
- 不实现任何登录/支付,除非里程碑要求

## 文档索引(做 X 之前先读 Y)

| 要做的事 | 先读 |
|---|---|
| 理解产品、什么不做 | docs/01-product.md |
| 写分析、渲染、评测、安全护栏 | docs/02-audio-quality.md |
| 写 recipe、预设、自适应规则 | docs/03-recipe-schema.md |
| 搭后端、任务、存储 | docs/04-architecture.md |
| 写接口 | docs/05-api-contract.md |
| 写前端页面 | docs/06-ui-spec.md |
| 知道现在该做什么 | docs/07-milestones.md + PROGRESS.md |
| 遇到奇怪的音频问题 | docs/08-pitfalls.md |
