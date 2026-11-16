# 05 API 契约

> 契约的唯一来源是 `api/app/models/` 里的 pydantic 模型。本文件描述意图和示例;实现后运行 `make openapi` 生成 `api/openapi.json`,前端类型从它生成。
> 文档与生成物不一致时,先修文档或模型,不要在前端手写类型绕过。

基础路径:`/api/v1`。全部 JSON,除上传(multipart)和文件下载。

## 通用

**任务对象**(所有异步接口都返回它):

```json
{
  "job_id": "job_8f3a",
  "type": "analysis | preview | export",
  "status": "queued | running | succeeded | failed",
  "progress": 0.42,
  "result": null,
  "error": null
}
```

`failed` 时 `error` 为 `{ "code": "...", "message": "中文可读信息" }`。`succeeded` 时 `result` 按类型填充(见下)。

**错误响应**(HTTP 4xx/5xx):`{ "error": { "code": "...", "message": "..." } }`

**错误码**

| code | HTTP | 含义 |
|---|---|---|
| `FILE_TOO_LARGE` | 413 | 超过大小限制 |
| `DURATION_TOO_LONG` | 422 | 超过时长限制 |
| `UNSUPPORTED_FORMAT` | 415 | 无法识别或不支持的格式 |
| `UNSUPPORTED_CHANNELS` | 422 | 多声道 |
| `DECODE_FAILED` | 422 | 解码失败或文件损坏 |
| `UPLOAD_NOT_FOUND` | 404 | upload_id 不存在或已过期 |
| `JOB_NOT_FOUND` | 404 | |
| `RECIPE_INVALID` | 422 | recipe 不符合 docs/03;`details` 里给出字段路径 |
| `GOAL_NOT_FOUND` | 404 | |
| `ANALYSIS_REQUIRED` | 409 | resolve 之前必须先有分析结果 |
| `INTERRUPTED` | 500 | 任务因进程重启中断 |
| `RENDER_FAILED` | 500 | 渲染失败 |
| `GUARDRAIL_FAILED` | 500 | 输出未通过 docs/02 的护栏(不得把不合格文件交给用户) |
| `RATE_LIMITED` | 429 | |

## 端点

### `POST /uploads`
multipart,字段 `file`。成功返回:

```json
{
  "upload_id": "up_c91e",
  "duration_s": 187.4,
  "sample_rate": 44100,
  "channels": 2,
  "format": "mp3",
  "expires_at": "2026-11-17T08:00:00Z"
}
```

### `DELETE /uploads/{upload_id}` → 204
立即删除上传及全部派生物。

### `POST /analyses`
请求 `{ "upload_id": "up_c91e" }` → 202,返回任务。`result` 为 `AnalysisReport`:

```json
{
  "integrated_lufs": -11.8,
  "loudness_range_lu": 6.2,
  "true_peak_dbtp": -0.3,
  "sample_peak_dbfs": -0.5,
  "plr_db": 11.5,
  "crest_factor_db": 12.1,
  "band_levels_db": { "sub": -38.0, "bass": -18.2, "low_mid": -14.5, "mid": -16.0, "presence": -22.1, "brilliance": -30.4, "air": -42.0 },
  "bandwidth_hz": 19500,
  "stereo_correlation": 0.82,
  "side_to_mid_db": -9.5,
  "clipped_samples": 0,
  "clipped_ratio": 0.0,
  "dc_offset": [0.0001, 0.0001],
  "silence_ratio": 0.0,
  "flags": ["bandwidth_limited"]
}
```
(示例数值仅示意格式。)`flags` 可选值:`bandwidth_limited`、`already_loud`、`has_clipping`、`near_mono`、`very_wide`。

### `GET /analyses/{upload_id}` → `AnalysisReport` 或 404 `ANALYSIS_REQUIRED`

### `GET /goals` → 目标列表

```json
[
  { "id": "balanced", "name": "均衡清晰", "description": "干净、平衡", "default_strength": 0.7 },
  { "id": "warm", "name": "温暖醇厚", "description": "更柔和、更有厚度", "default_strength": 0.7 },
  { "id": "bright", "name": "明亮通透", "description": "更清楚、更有空气感", "default_strength": 0.7 },
  { "id": "punch", "name": "紧实有力", "description": "低频更紧,更有冲击", "default_strength": 0.7 }
]
```
(`name`/`description` 来自后端 `recipes/` 元数据,前端直接显示。)

### `POST /recipes/resolve`(同步)

```json
{ "upload_id": "up_c91e", "goal_id": "warm", "strength": 0.7, "loudness": { "mode": "target", "target_lufs": -14 } }
```
返回:

```json
{ "recipe": { "...见 docs/03...": "" }, "notes": ["已减轻低中频浑浊"], "warnings": [] }
```
`loudness.mode`:`target`(带 `target_lufs`)或 `keep`。`resolve` 把它写入返回 recipe 的 `loudness` 节点:`target` → `{ "target_lufs": -14 }`;`keep` → `{ "mode": "keep" }`(见 docs/03)。

### `POST /previews`

```json
{ "upload_id": "up_c91e", "recipe": { "...": "" }, "start_s": 45.0 }
```
`start_s` 缺省时服务端选取:跳过前 10% 和后 10%,在中段选响度最高的 30 秒窗口。→ 202,返回任务。`result`:

```json
{
  "start_s": 45.0,
  "duration_s": 30.0,
  "original_url": "/api/v1/files/f_01",
  "processed_url": "/api/v1/files/f_02",
  "gain_original_db": 0.0,
  "gain_processed_db": -2.3,
  "before": { "integrated_lufs": -11.8, "true_peak_dbtp": -0.3 },
  "after":  { "integrated_lufs": -14.0, "true_peak_dbtp": -1.0 },
  "warnings": []
}
```
`gain_*` 规则见 docs/02 第 3 节。`before`/`after` 是**整首**的值。

### `POST /exports`

```json
{
  "upload_id": "up_c91e",
  "recipe": { "...": "" },
  "format": "wav | flac | mp3",
  "bit_depth": 24
}
```
`bit_depth` 只对 wav/flac 有效(16/24)。→ 202。`result`:

```json
{
  "download_url": "/api/v1/files/f_03",
  "filename": "song_audiolab.wav",
  "size_bytes": 38200000,
  "final": { "integrated_lufs": -14.0, "true_peak_dbtp": -1.0 },
  "warnings": []
}
```
`filename` 由服务端生成:`<原文件名去扩展名,净化后>_audiolab.<ext>`;净化 = 只保留字母数字、中日韩字符、下划线和短横线,最长 60 字符。

### `GET /jobs/{job_id}` → 任务对象

### `GET /files/{file_id}`
下载。支持 `Range` 请求(浏览器播放需要)。`Content-Disposition` 对导出文件为 attachment。

## 警告码(`warnings` 字段的取值)

`loudness_limited`、`heavy_limiting`、`already_loud`、`has_clipping`、`bandwidth_limited`、`stereo_narrowed`。前端把它们映射成 `copy.zh.ts` 里的中文文案,**未知警告码要被忽略而不是崩溃**。

## 契约测试

- 每个端点至少:成功用例 + 每个相关错误码的用例
- `make openapi` 之后 `git diff --exit-code api/openapi.json` 必须干净(`make check` 里包含这一步)
