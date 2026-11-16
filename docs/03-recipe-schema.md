# 03 Recipe(处理方案)Schema

> recipe 是前后端共用的契约:新手选"目标"只是载入并调整一份 recipe;专业用户在"处理详情"里编辑的是同一份 JSON;
> 后端只按 recipe 渲染。**前端不允许有任何不经过 recipe 的音频处理逻辑。**
>
> 下面所有数值是**起点值**,需在 M4 经人工试听校准。校准后同步更新本文件和 `recipes/*.json`。

## 1. 结构

```json
{
  "schema_version": 1,
  "goal_id": "balanced",
  "strength": 0.7,
  "nodes": [
    { "id": "hp1", "type": "highpass", "enabled": true, "params": { "freq_hz": 25 } },
    { "id": "eq1", "type": "eq", "enabled": true, "params": { "bands": [
      { "type": "peak", "freq_hz": 300, "gain_db": -1.5, "q": 1.0 },
      { "type": "peak", "freq_hz": 3500, "gain_db": 1.0, "q": 0.8 },
      { "type": "highshelf", "freq_hz": 10000, "gain_db": 1.0, "q": 0.7 }
    ] } },
    { "id": "cmp1", "type": "compressor", "enabled": true, "params": {
      "threshold_db": -18, "ratio": 1.8, "attack_ms": 30, "release_ms": 150 } },
    { "id": "wid1", "type": "stereo_width", "enabled": true, "params": { "width": 1.0 } },
    { "id": "lim1", "type": "limiter", "enabled": true, "params": {
      "ceiling_dbtp": -1.0, "release_ms": 100 } },
    { "id": "loud1", "type": "loudness", "enabled": true, "params": { "target_lufs": -14.0 } }
  ]
}
```

规则:
- `nodes` 有序;渲染顺序见第 3 节,服务端会**校验并重排/拒绝**不合法的顺序
- `id` 在同一份 recipe 内唯一
- 未知 `type`、缺少必填参数、越界参数:一律返回 `RECIPE_INVALID`,**不要**偷偷 clamp 或修正
- `strength` 只由 `resolve` 接口使用(见第 4 节);渲染时只看各节点最终参数

## 2. 节点类型与参数范围(MVP)

| type | 参数 | 范围 | 实现 |
|---|---|---|---|
| `highpass` | `freq_hz` | 15–60 | pedalboard `HighpassFilter` |
| `eq` | `bands[]`,最多 8 个;每个 `type`(`peak`/`lowshelf`/`highshelf`/`lowpass`/`highpass`)、`freq_hz`(20–20000)、`gain_db`(−9..+9,滤波器类型忽略)、`q`(0.3–4.0) | | pedalboard `PeakFilter` / `LowShelfFilter` / `HighShelfFilter` |
| `compressor` | `threshold_db` −40..0;`ratio` 1.0–6.0;`attack_ms` 1–100;`release_ms` 20–500 | | pedalboard `Compressor`,不加 makeup(响度由 `loudness` 节点负责) |
| `saturation` | `drive` 0–1;`mix` 0–1 | | 自行实现 `tanh` 波形整形,4 倍过采样避免混叠;`mix` 为干湿比 |
| `stereo_width` | `width` 0.5–1.5(1.0=不变) | | Mid/Side 处理:S *= width;单声道输入直接旁路 |
| `limiter` | `ceiling_dbtp` −3.0..−0.1;`release_ms` 20–300 | | pedalboard `Limiter`,后接真峰值检查(见 pitfalls) |
| `loudness` | `target_lufs` −24..−8;或 `"mode": "keep"`(保持原响度) | | 见第 3 节 |

**不是 MVP 的节点**(不要实现):reverb、delay、多段压缩、动态 EQ、降噪、分离。

## 3. 渲染顺序与响度阶段

固定顺序(服务端强制):

```
highpass → eq → compressor → saturation → stereo_width → [loudness 增益] → limiter
```

`loudness` 与 `limiter` 的联动(`render.py` 必须实现):

1. 先把前面节点处理完得到信号 `x`
2. 目标增益 `g = target_lufs − LUFS(x)`(`mode: keep` 时 `target = 原曲 LUFS`)
3. 应用 `g` → 过 limiter → 测量输出 LUFS
4. 如果 |输出 LUFS − 目标| > 0.2 LU,把 `g` 调整差值,最多迭代 3 次
5. 3 次后仍达不到(受 ceiling 限制):接受当前结果,警告 `loudness_limited`,并在结果里返回实际 LUFS
6. 如果 limiter 的最大增益衰减 > 6 dB,警告 `heavy_limiting`

## 4. 目标预设与强度

预设文件:`recipes/<goal_id>.json`,格式同上,是"强度 = 1.0"时的完整 recipe。

`neutral` recipe(强度 = 0):保留 `highpass`、`limiter`、`loudness`;其余节点的增益类参数为 0 或旁路。

**强度插值**(`resolve` 接口,`api/app/services/resolve.py`):

```
param = neutral + strength × (preset − neutral)
```

只对下列参数做线性插值:`gain_db`、`drive`、`mix`、`width`、压缩器 `ratio`(插 `ratio − 1`)。
`freq_hz`、`q`、`attack_ms`、`release_ms`、`threshold_db`、`ceiling_dbtp`、`target_lufs` 不插值,取预设值。

### 预设起点值

**balanced 均衡清晰**(见第 1 节 JSON)

**warm 温暖醇厚**
- `eq`:`lowshelf` 150 Hz +1.5 dB;`peak` 2800 Hz −1.0 dB q 0.8;`highshelf` 9000 Hz −1.0 dB
- `compressor`:threshold −18、ratio 1.6、attack 30、release 180
- `saturation`:drive 0.15、mix 0.5
- `stereo_width`:1.0

**bright 明亮通透**
- `eq`:`peak` 250 Hz −1.0 dB q 1.0;`peak` 4000 Hz +1.5 dB q 0.8;`highshelf` 11000 Hz +2.0 dB
- `compressor`:threshold −18、ratio 1.5、attack 25、release 150
- `stereo_width`:1.05

**punch 紧实有力**
- `highpass` 30 Hz
- `eq`:`peak` 60 Hz +1.0 dB q 1.0;`peak` 300 Hz −1.5 dB q 1.0;`peak` 3000 Hz +0.8 dB q 0.9
- `compressor`:threshold −20、ratio 2.5、attack 15、release 100
- `stereo_width`:1.0

## 5. 自适应规则(`api/app/services/adapt.py`)

`resolve` 在插值之后,根据分析报告做有界调整。每条规则:**触发条件、动作、上限、返回的 `notes`**。所有触发阈值是起点值。

| 规则 | 触发 | 动作(有界) | 提示(`notes`) |
|---|---|---|---|
| 低中频浑浊 | `band_levels.low_mid − band_levels.mid > 3 dB` | 在 300 Hz 追加峰值切 `min(2.0, 0.5×(差−3)) dB` | "已减轻低中频浑浊" |
| 中高频偏刺 | `band_levels.presence − band_levels.mid > 2 dB` | 把 recipe 中 2–5 kHz 的正增益减半,最多减到 0 | "中高频已偏亮,已收敛提升" |
| 带宽受限 | `bandwidth_hz < 16000` | 把所有 ≥ 9 kHz 的 `highshelf` 增益置 0 | "音源高频被截断,不增加高频" |
| 已经很响 | `integrated_lufs > −9` | 压缩器 ratio 上限 1.5;警告 | "原曲已经很响,动态空间有限" |
| 近单声道 | `stereo_correlation > 0.98` | `width` 固定 1.0 | (无提示) |
| 声场过宽 | `stereo_correlation < 0.3` | `width` ≤ 1.0 | "声场已经很宽" |
| 已有削波 | `clipped_samples / total > 0.0001` | 不处理;警告 | "原曲已有削波,无法修复" |

每条规则必须有单元测试:构造一份分析报告 → 断言 recipe 的具体变化值。

## 6. 校验与测试要求

- `api/app/models/recipe.py` 用 pydantic 实现本 schema,参数范围用 `Field(ge=, le=)`
- 测试:每个节点类型至少一个越界用例(期望 `RECIPE_INVALID`);一个"往返"用例(JSON → 模型 → JSON 不变)
- `recipes/*.json` 全部能通过模型校验,有测试覆盖
