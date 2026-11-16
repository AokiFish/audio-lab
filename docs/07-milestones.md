# 07 里程碑

规则:
- 一次只做一个里程碑。完成后更新 PROGRESS.md 并**停下**,等人确认。
- 每个里程碑的"验收标准"逐条满足才算完成;"人工确认"项由人勾选,agent 不得自行勾选。
- 范围之外的想法,只写进 PROGRESS.md 的"建议"。

---

## M0 骨架

**目标**:仓库可以安装、运行、检查;上传一个文件能走通"上传 → 下载"的最小闭环(下载的是 ffmpeg 解码再编码的原样音频)。

**范围**
- 目录结构按 AGENTS.md;`Makefile` 的全部命令可用(`fixtures`/`eval` 此时可以是明确的"尚未实现"并以非零退出,但命令必须存在)
- `api`:FastAPI 应用、`POST /uploads`、`DELETE /uploads/{id}`、`GET /files/{id}`、`GET /health`;上传校验(docs/04 的限制);统一错误响应格式
- `web`:Vite + React + TS 项目;一个能选择文件并显示上传结果(文件名、时长)的页面;tokens.css;`copy.zh.ts`
- `make openapi` 能生成 `api/openapi.json` 与前端类型
- CI 配置(GitHub Actions 或等价)运行 `make check`
- `.env.example`、`.gitignore`(含 `tests/golden/real/`、`reports/`、数据目录)
- 锁定依赖版本,记录到 PROGRESS.md

**不做**:分析、recipe、任何 DSP、任务系统、样式细节

**验收标准**
- [ ] 新克隆的仓库按 README 的 3 条命令(`make setup`、`make dev`、`make check`)可用
- [ ] 上传 wav、mp3、flac、m4a 各一个小文件成功,返回正确的 `duration_s`/`sample_rate`/`channels`
- [ ] 上传一个改了扩展名的文本文件 → `UNSUPPORTED_FORMAT`;超大文件 → `FILE_TOO_LARGE`;超长 → `DURATION_TOO_LONG`
- [ ] 路径穿越文件名(如 `../../x.wav`)不会写出数据目录(有测试)
- [ ] `DELETE /uploads/{id}` 后相关文件消失(有测试)
- [ ] `make check` 在干净环境通过

**人工确认**:无

---

## M1 分析

**目标**:对上传的音频输出 docs/02 第 1 节全部指标。

**范围**:`audio/metrics.py`、`audio/analyze.py`、`tools/make_fixtures.py`(生成 docs/02 第 4 节的合成音频)、任务系统(线程池 + SQLite)、`POST /analyses`、`GET /analyses/{id}`、`GET /jobs/{id}`;前端:上传后显示"正在听这首歌…"并在完成后显示文件名与时长(不显示指标)

**不做**:recipe、渲染、目标选择

**验收标准**
- [ ] `make fixtures` 可复现(两次运行产出的文件哈希相同)
- [ ] 数值断言:
  - 单声道 -20 dBFS 的 1 kHz 正弦波 → LUFS 与理论值相差 ≤ 0.1 LU
  - `pink_noise` 在各频段的 `band_levels` 满足粉红噪声的 −3 dB/倍频程趋势(容差 ±1.5 dB)
  - `hot_mix` 的 `clipped_samples` > 0;`lowpassed_16k` 的 `bandwidth_hz` 在 15500–16500 之间;`near_mono` 的 `stereo_correlation` > 0.98;`wide_mix` < 0.3
  - `true_peak_dbtp` 与 ffmpeg `ebur128=peak=true` 的结果相差 ≤ 0.2 dB
- [ ] 分析 4 分钟歌曲的耗时已记录到 PROGRESS.md
- [ ] 任务失败时 `error` 字段有错误码;进程重启后遗留的 `running` 任务变 `failed`(有测试)

**人工确认**:把 3 首真实歌曲放进 `tests/golden/real/`,查看分析数值是否合理(例如 LUFS 与你用其他软件看到的接近)

---

## M2 渲染引擎

**目标**:按 recipe 渲染音频,所有护栏通过;`make eval` 可用。

**范围**:`models/recipe.py`、`audio/nodes/*`(highpass、eq、compressor、saturation、stereo_width、limiter)、`audio/render.py`(含响度阶段迭代)、`recipes/*.json`(4 个目标 + neutral)、`services/resolve.py`、`services/adapt.py`、`POST /recipes/resolve`、`GET /goals`、`tools/eval.py`(`make eval`)、渲染缓存

**不做**:预览接口、前端页面、导出接口

**验收标准**
- [ ] 每个节点有数值测试:
  - `highpass` 25 Hz:对 10 Hz 正弦衰减 ≥ 10 dB,对 1 kHz 变化 < 0.1 dB
  - `eq` peak 1 kHz +3 dB(q=1):对 1 kHz 正弦增益 3 ± 0.2 dB;对 100 Hz、10 kHz 变化 < 0.3 dB
  - `stereo_width` 0.5 → 侧声道能量下降约 6 dB(±0.5);单声道输入旁路
  - `limiter`:任意输入输出真峰值 ≤ ceiling + 0.1 dB
- [ ] `strength=0` 的渲染结果除响度/限制外与输入几乎一致(频谱各频段变化 < 0.3 dB)
- [ ] `make eval` 对 4 目标 × 9 合成 fixture 全部通过 G1–G8,且 `reports/eval.md` 与前后频谱图生成
- [ ] 每条自适应规则有单元测试(docs/03 第 5 节)
- [ ] 无效 recipe 返回 `RECIPE_INVALID`,`details` 含字段路径
- [ ] 渲染 4 分钟歌曲耗时已记录到 PROGRESS.md

**人工确认**:对 3 首真实歌曲跑 `make eval`,打开报告和前后频谱图,确认变化方向合理(不听,只看图)

---

## M3 预览与 A/B

**目标**:用户能在浏览器里同响度对比原曲和处理后。

**范围**:`POST /previews`(渲染整首、缓存、切 30 秒、同响度增益)、`GET /files/{id}` 支持 `Range`;前端:Ready 状态页(播放器、A/B 切换、默认选 `balanced` 强度 70%)、Web Audio 的 A/B 双缓冲

**不做**:选择其他目标、强度滑块、导出

**验收标准**
- [ ] 服务端测试:`gain_original_db`、`gain_processed_db` 均 ≤ 0 且至少一个为 0;应用增益后两段响度相差 < 0.2 LU
- [ ] 改变 `start_s` 不触发重新渲染(缓存命中,有测试)
- [ ] 前端:dB→线性换算有测试;A/B 切换保持播放位置(用 Web Audio 的 mock 测试)
- [ ] 完整流程可走通:上传 → 分析 → 播放器出现 → 可播放、可切换 A/B

**人工确认**
- [ ] A/B 切换无爆音、无明显的位置跳变
- [ ] 切换时音量感受一致

---

## M4 声音目标与强度

**目标**:4 个目标 + 强度滑块可用,notes/warnings 展示。

**范围**:前端目标选择与强度滑块(防抖 400 ms)、更新中状态、notes/warnings 文案映射(`copy.zh.ts`)、根据人工试听校准 `recipes/*.json`

**不做**:导出、处理详情

**验收标准**
- [ ] 切换目标/强度后 B 段更新;更新期间原播放不中断
- [ ] 每个后端警告码有对应文案(测试遍历)
- [ ] 前端状态机每个转换有测试
- [ ] `make eval` 仍全部通过

**人工确认**(这是校准里程碑,必须由人完成;按 docs/02 第 6 节的模板记录)
- [ ] 4 个目标在 ≥ 5 首真实歌曲上方向符合名称
- [ ] 强度 100% 无明显失真/泵感;强度 0% ≈ 原曲
- [ ] 校准后的数值已写回 `recipes/*.json` 和 docs/03

---

## M5 导出

**目标**:导出 WAV/FLAC/MP3,通过全部护栏。

**范围**:`POST /exports`、编码(ffmpeg)、编码后护栏检查与有损格式的真峰值重试、响度目标三档、前端导出面板与 Done 状态

**验收标准**
- [ ] 三种格式导出文件均能被 ffprobe 正确读取,时长与输入相差 ≤ 10 ms(MP3 因编码延迟允许 ≤ 50 ms,但需在 PROGRESS.md 说明)
- [ ] 各格式解码后 G1–G8 通过(MP3 重点测真峰值,见 pitfalls)
- [ ] 三档响度目标各有数值测试
- [ ] 文件名净化有测试(包含中文名、路径穿越、超长名)
- [ ] `GUARDRAIL_FAILED` 时不提供下载(有测试)

**人工确认**
- [ ] WAV/FLAC/MP3 导出文件在常用播放器里正常播放,听感一致

---

## M6 处理详情(专业视图)

**目标**:同一份 recipe 的可见与可编辑。

**范围**:处理详情面板(只读 → 可编辑参数 + "应用"重新预览)、中文名 + 英文术语、重置为目标默认

**验收标准**
- [ ] 面板显示的每个节点/参数与发送给后端的 recipe 完全一致(有测试)
- [ ] 编辑后点"应用"触发新预览;超范围值在前端给出提示(范围来自 `openapi.json`/后端返回,不在前端另写一份)
- [ ] 重置能回到目标默认值

**人工确认**:专业用户试用,反馈哪些参数缺失(写入 PROGRESS.md 的"建议")

---

## M6 之后(只是路线图,不要提前做)

人声/伴奏分离后的分别处理 → 降噪与修复 → 参考曲匹配 → 多段压缩与动态 EQ → 批处理 → 账号与项目历史 → Redis 队列与 GPU worker。
