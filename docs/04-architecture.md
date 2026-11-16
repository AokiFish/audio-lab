# 04 架构

## 总览

```
浏览器 (React + Web Audio)
   │  HTTPS / JSON(契约见 docs/05)
   ▼
FastAPI (api/app)
   ├─ routes/      参数校验、返回
   ├─ services/    上传、分析、resolve、预览、导出、清理
   ├─ audio/       纯音频代码(不依赖 FastAPI,可单独测试)
   ├─ models/      pydantic:recipe、报告、请求/响应
   └─ jobs/        线程池 + SQLite 任务表
   │
   ├─ 本地磁盘:uploads/、renders/、exports/(路径来自环境变量)
   └─ ffmpeg(子进程)
```

MVP 单机运行:一个 API 进程 + 线程池。不引入 Redis / Celery / 对象存储 / 账号系统。

## 数据流

1. **上传**:`POST /uploads` 收到文件 → 校验 → 用 ffmpeg 解码为内部标准格式(32-bit float WAV,保持原采样率与声道)存盘 → 返回 `upload_id`
2. **分析**:`POST /analyses` 创建任务 → 线程池执行 `analyze` → 结果存 SQLite(JSON)
3. **resolve**:`POST /recipes/resolve`(同步,毫秒级)→ 目标 + 强度 + 分析报告 → recipe(含 notes)
4. **预览**:`POST /previews` 创建任务 → **渲染整首**(缓存键 = upload_id + recipe 哈希)→ 切出 30 秒原曲段与处理段 → 编码为同一种格式 → 计算同响度增益 → 返回
5. **导出**:`POST /exports` 创建任务 → 渲染整首(命中缓存则直接用)→ 编码 → 护栏检查(含编码后真峰值)→ 返回下载链接

### 为什么预览渲染整首

响度阶段需要整首的响度才与最终导出一致;pedalboard 渲染很快(目标:4 分钟立体声 ≤ 10 秒,M2 实测并记录)。移动试听位置只是换切片,不重新渲染。

### 预览格式

原曲段和处理段使用完全相同的编码(16-bit PCM WAV,44.1 或 48 kHz,与源一致),保证 A/B 差异只来自处理。30 秒 ≈ 5 MB 每段。

## 任务系统

- SQLite 表 `jobs(id, type, status, progress, params_json, result_json, error_json, created_at, updated_at)`
- `status`:`queued` → `running` → `succeeded` / `failed`
- 线程池大小来自 `AUDIO_LAB_WORKERS`(默认 2)。同一个 `upload_id` 的相同请求(相同类型 + 相同参数哈希)去重,返回已有任务
- 进程重启:`running` 状态的任务启动时标记为 `failed`(错误码 `INTERRUPTED`)

## 存储与清理

- 目录:`$AUDIO_LAB_DATA/uploads/<upload_id>/`,`renders/<upload_id>/<recipe_hash>/`,`exports/<export_id>/`
- 默认 TTL 24 小时(`AUDIO_LAB_TTL_HOURS`),后台每 15 分钟清理过期上传及其派生物
- `DELETE /uploads/{id}` 立即删除所有派生物和数据库记录

## 限制(来自环境变量,默认值)

| 项 | 默认 |
|---|---|
| 单文件大小 | 80 MB |
| 时长 | 10 分钟 |
| 格式 | wav、mp3、flac、m4a/aac(以 ffprobe 识别结果为准,不以扩展名为准) |
| 声道 | 单声道或立体声(多声道拒绝:`UNSUPPORTED_CHANNELS`) |
| 并发任务/客户端 | 3 |

## 配置(环境变量)

`AUDIO_LAB_DATA`、`AUDIO_LAB_WORKERS`、`AUDIO_LAB_TTL_HOURS`、`AUDIO_LAB_MAX_MB`、`AUDIO_LAB_MAX_SECONDS`、`AUDIO_LAB_CORS_ORIGINS`。提供 `.env.example`,不提交 `.env`。

## 性能目标(M2 实测后写进 PROGRESS.md;未达标要说明原因)

- 分析 4 分钟歌曲 ≤ 5 秒
- 渲染 4 分钟歌曲 ≤ 10 秒
- resolve ≤ 50 ms

## 部署(只写方向,M0–M6 不实现)

Docker Compose:`api`(含 ffmpeg)+ `web`(静态文件,由反向代理提供)。如面向中国大陆公开提供服务,需要另行处理域名备案与内容合规,这不属于代码范围,开发阶段不考虑。

## 后续演进(不要现在做)

任务量增加后:Redis + RQ,对象存储,GPU worker 运行分离/降噪模型,用户账号与历史版本。设计上 `audio/` 保持纯函数、`jobs/` 保持接口窄,是为了到时候替换而不改业务逻辑。
