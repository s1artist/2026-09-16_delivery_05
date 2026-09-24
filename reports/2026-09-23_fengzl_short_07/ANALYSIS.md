# `2026-09-23_fengzl_short_07` 中断后产物审计

分析原则：只解析已落盘原始日志；不重训、不补点、不推断不存在的 fixed eval 结果。

## 产物定位

- 训练日志：`/hpfs/huawei/fengzl/mopd-delivery-05/experiments/2026-09-23_fengzl_short_07/train.log`
- 提交日志：`/hpfs/huawei/fengzl/mopd-delivery-05/experiments/2026-09-23_fengzl_short_07/submit.log`
- 运行快照：`/hpfs/huawei/fengzl/mopd-delivery-05/experiments/2026-09-23_fengzl_short_07/run.sh`
- 配置目标 checkpoint：`/hpfs/huawei/fengzl/mopd-delivery-05/models/mopd-short/mopd-2026-09-23_fengzl_short_07`
- 实际 checkpoint：不存在。训练设置为每 20 步保存，中断前未到首次保存点。
- `fixed-eval.log`：未找到。本次 run 没有任何 `fixed_kl/*` 评测点。
- `full-inference-07.log` 只是等待导出模型的记录，不是 fixed eval。
- 项目副本和共享运行目录中均未找到 `VISUALIZATION.md`。

## 已完成进度

- 最后完整训练记录：step 9。
- 日志没有显式 `completed_updates` 字段。
- step 0–9 共 10 条完整更新记录，因此可直接确定已完成更新数为 10。
- step 10 rollout 在 7/8 时收到 SIGTERM，没有完整 rollout 聚合记录，也没有训练更新记录；不计入曲线和统计。

## 指标完整性

以下九项均完整覆盖 step 0–9，每项 10 个真实数据点：

```json
["'train/loss':","'train/grad_norm':","'train/train_rollout_logprob_abs_diff':","'rollout/truncated_ratio':","'perf/step_time':","'perf/rollout_time':","'train/mopd_topk_kl':","'train/mopd_topk_kl/math':","'train/mopd_topk_kl/code':"]
```

| 指标 | 最小值 | 最大值 | 均值 |
|---|---:|---:|---:|
| `train/loss` | 0.039570 | 0.065006 | 0.051896 |
| `train/grad_norm` | 1.923217 | 6.231790 | 4.126903 |
| `train/train_rollout_logprob_abs_diff` | 0.008198 | 0.012497 | 0.011052 |
| `rollout/truncated_ratio` | 0 | 0.25 | 0.075 |
| `perf/step_time` | 100.220 s | 1977.358 s | 1167.304 s |
| `perf/rollout_time` | 64.331 s | 1931.416 s | 1126.524 s |
| `train/mopd_topk_kl` | 0.039699 | 0.064761 | 0.051875 |
| `train/mopd_topk_kl/math` | 0.019832 | 0.035510 | 0.026457 |
| `train/mopd_topk_kl/code` | 0.019385 | 0.031858 | 0.025418 |

## 中断前记录检查

要求检查最后 20 条，但本次只有 10 条完整记录，因此实际检查 step 0–9 全部记录。

- NaN/inf：九项指标中没有非有限值。
- loss：范围 0.039570–0.065006。最大相邻上升为 step 0→1 的 +33.94%，最大下降为 step 4→5 的 -34.86%；没有持续发散。最后 step 9 为 0.061474，较 step 8 上升 18.39%，仍在此前范围内。
- grad norm：范围 1.923–6.232，没有梯度爆炸或非有限值。step 9 降至最低值 1.923，但不是异常升高。
- KL：总 KL 范围 0.039699–0.064761，与 loss 同步波动，没有越出既有范围或非有限值。step 9 为 0.061693，较 step 8 上升 18.71%，仍低于历史最高点 step 4。
- truncated ratio：step 6 最高 0.25，即 2/8；step 1、3、5、7 为 0.125，其余为 0。10 步总计 6/80，整体 7.5%。step 6 是局部高点，但没有连续恶化。
- 时间：step 1、3、5、6、7 的 rollout 约 1903–1931 秒，且均出现至少一条截断，符合生成到 32K 上限的耗时形态。最高 rollout time 为 step 6 的 1931.416 秒，最高 step time 为 step 3 的 1977.358 秒；这不是中断前突然升高。最后 step 9 分别降至 177.478 秒和 219.050 秒。
- 中断发生在下一轮 rollout 期间，由显式 SIGTERM 停止；最后完整 step 9 本身没有数值异常信号。

## Fixed eval 与 checkpoint

期望 fixed-eval key：

```json
["'fixed_kl/math':","'fixed_kl/code':","'fixed_kl/code/le128k':","'fixed_kl/code/gt128k':"]
```

实际评测点为 0；四项都没有落盘数据，因此没有生成空图。

本次 run 没有 checkpoint，不能针对本次训练状态单独重跑 fixed eval。共享目录里虽然有原实验 `2026-09-16_short_05` 的 step 20/40/60 checkpoint，但它们不是本次 run 的产物，不能用来补本次 fixed eval。

## 输出文件

- `metrics.csv`：九项指标的 step 0–9 原始解析值。
- `persisted-metrics.png`：只绘制已有数据点。
- `analysis.json`：机器可读统计、连续变化和非有限值检查。
- `analyze_existing.py`：只读解析与绘图脚本，不属于训练代码。
- `traininglogparser-train-top.png` / `traininglogparser-train-bottom.png`：实际在 TrainingLogParser 网页上传本次 `train.log`、填入指定 Global parse key 后的上下两部分截图。
- `traininglogparser-result.json`：网页内 `FileData.keyDatas` 的实际解析点数和值；九项均为 10 点，与正式 Python 解析逐项一致。

当前 TrainingLogParser 版本没有独立的 `Parse key` 按钮；修改 Global parse key 的输入事件会自动调用 `setGlobalLossTag()` 和刷新曲线。本次操作已先清空再填入指定 key，并用 Enter 和失焦确认。

由于 `fixed-eval.log` 不存在，未向网页上传伪造文件。四个 fixed KL key 的实际点数均为 0，值均为空数组。

## 还能分析与缺失项

现有日志可以直接分析：十次更新的训练稳定性、KL/math/code 波动、rollout/train logprob 对齐、截断率、step/rollout 耗时，以及这些指标之间的对应关系。

当前缺少：step 10 及之后完整指标、20 步 checkpoint、所有 fixed eval 指标、长期收敛趋势。由于不存在本次 checkpoint，目前没有“只额外跑 eval、不重训”即可补齐 fixed eval 的路径。
