# 02 音频质量:指标、不变量、评测

> 你(agent)听不到声音。本文件定义"怎么用数字判断处理没有变坏",以及哪些事必须交给人听。
> 文中所有"起点阈值"都是初始值,会在 M4 人工试听后校准;校准时必须同步修改本文件和测试。

## 1. 分析指标(`api/app/audio/metrics.py`,M1 实现)

输入:解码后的 float32 数组 `(channels, samples)` + 采样率。输出:`AnalysisReport`(字段见 docs/05)。

| 指标 | 定义 | 实现要点 |
|---|---|---|
| `integrated_lufs` | ITU-R BS.1770 综合响度 | pyloudnorm `Meter`;短于 0.4 s 的音频报错 |
| `loudness_range_lu` | LRA | 可用 ffmpeg `ebur128` 滤镜交叉验证 |
| `true_peak_dbtp` | 真峰值 | 4 倍过采样(`scipy.signal.resample_poly(x, 4, 1)`)后取最大绝对值转 dB;测试里用 ffmpeg `ebur128=peak=true` 交叉验证,容差 0.2 dB |
| `sample_peak_dbfs` | 采样峰值 | 直接取最大绝对值 |
| `plr_db` | 峰值/响度比 = true_peak − integrated_lufs | 衡量动态空间 |
| `crest_factor_db` | 峰值/RMS | 整体 |
| `band_levels_db` | 7 个频段的平均能量(dB,相对整体 RMS) | 频段见下表;用 Welch 或 STFT 平均功率谱,汉宁窗,n_fft=4096 |
| `bandwidth_hz` | 有效带宽:功率谱能量降到比 1–5 kHz 平均低 60 dB 以下的最低频率 | 用于识别有损源 |
| `stereo_correlation` | 左右声道相关系数(−1..1) | 整体 |
| `side_to_mid_db` | 侧声道能量相对中声道 | M=(L+R)/2, S=(L−R)/2 |
| `clipped_samples` / `clipped_ratio` | 绝对值 ≥ 0.999 的采样点数 / 占总采样点比例 | |
| `flags` | 由上面指标派生的标记:`bandwidth_limited`(bandwidth_hz < 16000)、`already_loud`(LUFS > −9)、`has_clipping`(clipped_ratio > 0.0001)、`near_mono`(相关性 > 0.98)、`very_wide`(相关性 < 0.3) | 阈值与 docs/03 第 5 节的自适应规则共用同一组常量,放在同一处定义 |
| `dc_offset` | 各声道均值 | |
| `silence_ratio` | 低于 -60 dBFS 的时间占比 | |

频段(Hz):`sub 20–60`、`bass 60–250`、`low_mid 250–500`、`mid 500–2000`、`presence 2000–5000`、`brilliance 5000–10000`、`air 10000–20000`。

## 2. 安全护栏(所有渲染输出必须满足,代码里强制 + 测试覆盖)

| 编号 | 规则 | 阈值 |
|---|---|---|
| G1 | 真峰值上限 | 无损导出 ≤ −1.0 dBTP;有损导出编码后解码再测 ≤ −1.0 dBTP(见 pitfalls) |
| G2 | 无新增削波 | 输出 `clipped_samples` ≤ 输入的 `clipped_samples` |
| G3 | 目标响度 | 选择了响度目标时,|输出 LUFS − 目标| ≤ 0.5 LU;如果被 G1 限制无法达到,返回实际值并带 `loudness_limited` 警告 |
| G4 | 时长不变 | |输出时长 − 输入时长| ≤ 10 ms;采样率与输入一致(除非导出选项指定) |
| G5 | 无非法数值 | 无 NaN/Inf;无 DC 偏移新增(> 0.001 视为失败) |
| G6 | 单声道兼容 | 输出的单声道折叠(L+R)/2 相对立体声的 RMS 下降不超过 3 dB;超过则降低 `stereo_width` 重试一次,仍超过则警告 |
| G7 | 频谱偏移有界 | 每个目标处理后,任一频段相对输入的变化不超过 recipe 允许范围 + 2 dB 容差(防止自适应规则失控) |
| G8 | 静音不被放大 | 输入静音段(< −60 dBFS)处理后仍 < −50 dBFS |

## 3. A/B 同响度

- 预览用 A(原曲段)和 B(处理段)两段 30 秒音频。
- 服务端计算两段各自的综合响度,返回 `gain_original_db` 与 `gain_processed_db`,**二者都 ≤ 0**,且恰有一个为 0:更响的那个被衰减,使两段响度相等(|差| < 0.2 LU)。
- 前端播放时应用这两个增益,不得在前端再自行计算响度。

## 4. 测试音频(`make fixtures` 生成,不提交二进制大文件之外的东西)

合成测试音频由 `tools/make_fixtures.py` 生成,用固定随机种子,保证可复现。生成的音频文件不提交到仓库(`.gitignore`),只提交脚本和 `tests/golden/expected.json`(各 fixture 的预期指标)。每个 10–20 秒,44.1 kHz,立体声 float32 → 16-bit WAV:

| 名称 | 内容 | 用来验证 |
|---|---|---|
| `sweep` | 20 Hz–20 kHz 对数扫频,-12 dBFS | 频率响应是否按 recipe 变化;无异常谐波 |
| `pink_noise` | 粉红噪声,-20 LUFS | 频段变化量精确测量 |
| `hot_mix` | 多个正弦+噪声混合,峰值贴近 0 dBFS,有少量削波 | G1/G2;限制器行为 |
| `muddy_mix` | 在 250–500 Hz 加厚的合成混音 | 自适应"低中频浑浊"规则触发 |
| `lowpassed_16k` | 16 kHz 以上截断的宽带信号 | 带宽检测;不加高频 |
| `near_mono` | 左右几乎相同 | 不凭空加宽;G6 |
| `wide_mix` | 左右相关性低 | 宽度规则;G6 |
| `quiet_mix` | -26 LUFS | 响度目标能否达到 |
| `silence_gaps` | 音频中间有 2 秒静音 | G8 |

**真实歌曲**:由负责人把 5–10 首 Suno 成品放进 `tests/golden/real/`(该目录 `.gitignore`,agent 不得下载或生成)。`make eval` 如果发现该目录有文件,则对它们同样运行并在报告里单独成表。

## 5. 评测(`make eval`,M2 起可用)

对每个 目标 × fixture 渲染一次,输出 `reports/eval.md`:

- 一张表:fixture、目标、输入/输出的 LUFS、true peak、各频段变化(dB)、stereo_correlation、警告
- 每条护栏 G1–G8 通过/失败
- 对每个 fixture 画前后频谱对比图(PNG,放 `reports/img/`),用 matplotlib(仅 tools 里使用,不进入运行时依赖)
- 任何护栏失败 → 命令以非零退出码结束

## 6. 必须由人完成的验证

agent 不得自行勾选这些项。在 PROGRESS.md 的"需要人工确认"栏列出具体曲目和要听的位置。

- [ ] 每个目标:同响度 A/B,方向是否符合名称(温暖就是更暖,明亮就是更亮)
- [ ] 强度 100% 时是否出现可察觉的失真、嘶声、泵感
- [ ] 强度 0% 时是否与原曲听感一致
- [ ] 导出 MP3 / FLAC / WAV 听感一致
- [ ] 人声部分是否更刺耳(2–5 kHz)或更浑浊(200–500 Hz)

**人工试听记录模板**(放在 `reports/listening-YYYYMMDD.md`):

```
曲目: ____  目标: ____  强度: ____
方向符合名称: 是 / 否 / 不明显
问题: (刺耳 / 浑浊 / 失真 / 泵感 / 低频过多 / 无)
建议调整: ____
```
