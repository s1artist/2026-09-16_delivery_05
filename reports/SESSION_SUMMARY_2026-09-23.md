# MOPD delivery_05 复现会话总结

更新时间：2026-09-23 15:44（Asia/Shanghai）

## 1. 任务目标与结论

目标是在两台 Ascend 机器上，不依赖 Kubernetes，按 `RUNBOOK.md` 和 `TEST_REPORT.md` 复现 MOPD 短训练、长训练、checkpoint 导出及推理流程。

本次已完成环境、镜像、补丁、Ray 双节点、模型和数据核验，并实际运行精确镜像短训练。短训练完整完成 step 0–9，step 10 rollout 运行到 5/8 时按用户要求停止。前 0–8 步的 loss/KL 与原始实验均值接近，可判定前期训练路径复现正确；300 步、长训练、HF 导出及最终推理尚未完成，不能宣称端到端全部验收完成。

## 2. 机器与运行方式

- Master：`10.127.10.167`，容器 `fengzl-mopd-master`
- Worker：`10.127.10.168`，通过本机 tmux 会话 `mashine2` 连接，容器 `fengzl-mopd-worker`
- Ray：Master `10.127.10.167:6166`，Jobs API `http://10.127.10.167:8265`
- 拓扑：2 个 Active 节点、共 32 NPU
- 实际分配：Master 16 NPU 训练；Worker 8 NPU rollout；其余卡不属于本任务
- 本次采用裸 Docker + Ray，无 Kubernetes

## 3. 镜像与补丁

- 指定镜像：`harbor.telecom-ai.com.cn/library/slime-ascend:v0.3.0-0818-nettools`
- 两机镜像 ID：`sha256:5c9925d63b98adda50cfad3b52208c8b578cb48b3fa68a5df6c4216f3845e595`
- 镜像 tar：`/hpfs/huawei/songchunjiang/docker-images/slime-ascend-v0.3.0-0818-nettools.tar`
- 加载日志：
  - `/hpfs/huawei/fengzl/mopd-delivery-05/experiments/image-load-specified-local.log`
  - `/hpfs/huawei/fengzl/mopd-delivery-05/experiments/image-load-specified-remote.log`

精确镜像是干净基础镜像，必须先执行原 K8s YAML 对应的环境准备。`manual/setup-container.sh` 已修正为按以下顺序准备：

1. 安装 Emerging Optimizers。
2. 运行 `workspace-utils/scripts/env/prepare_env.sh`，应用 Megatron、mbridge、SGLang、slime-ascend 共 5 个补丁。
3. 覆盖 MOPD 的 23 个源码文件。
4. 覆盖 9 个框架文件。
5. 检查 MHC 代码和 `telechat4-29B-Moe.sh`。

模型配置脚本 SHA256：`545120c9aafa5b1c7ed1ef06c94cc945dd4fd672c28ae3154986605f61caa4e5`。

## 4. 代码、文档和数据位置

- 本地工作副本：`/home/telenlp/bpfs/fengzl/mopd-delivery-05`
- 共享运行副本：`/hpfs/huawei/fengzl/mopd-delivery-05`
- 原始文档：
  - `/hpfs/huawei/fengzl/mopd-delivery-05/docs/original/RUNBOOK.md`
  - `/hpfs/huawei/fengzl/mopd-delivery-05/docs/original/TEST_REPORT.md`
- 当前短数据：`/hpfs/huawei/fengzl/mopd-delivery-05/datasets/mopd-short-prompt-05/train.jsonl`
- 数据共 800 行，与原数据 SHA256 完全相同：`55b8f9390f09afa5a8f38f27b796273430b6c85d76bb39617ae4f3839d999c6d`

模型没有复制，继续复用原共享路径：

- Student HF：`/hpfs/huawei/songchunjiang/weights_hf/12.05T_wodsa_256k_gbs8_id11_stage3_1000_hf`
- Student Megatron：`/hpfs/huawei/songchunjiang/mopd-delivery-05/models/mopd-torch-dist/student`
- Math teacher：`/hpfs/huawei/songchunjiang/mopd-delivery-05/models/mopd-torch-dist/math_teacher`
- Code teacher：`/hpfs/huawei/songchunjiang/mopd-delivery-05/models/mopd-torch-dist/code_teacher`

三份 Megatron 模型的 `latest_checkpointed_iteration.txt` 均为 `release`，实际训练日志确认三份权重均成功加载。

## 5. 当前精确镜像实验

- 实验名：`2026-09-23_fengzl_short_07`
- Ray Job ID：`mopd-2026-09-23_fengzl_short_07`
- 实验目录：`/hpfs/huawei/fengzl/mopd-delivery-05/experiments/2026-09-23_fengzl_short_07`
- 训练日志：`/hpfs/huawei/fengzl/mopd-delivery-05/experiments/2026-09-23_fengzl_short_07/train.log`
- checkpoint 目录：`/hpfs/huawei/fengzl/mopd-delivery-05/models/mopd-short/mopd-2026-09-23_fengzl_short_07`

主要参数与文档一致：300 rollout、每 20 步保存、global batch 8、micro batch 1、96K prompt、32K response、128K context、temperature 0.6、MOPD top-k 1024、loss chunk 512、alpha 0、Adam lr 1e-6、TP4/CP4/EP8/PP1、16 NPU 训练、8 NPU DP8 rollout、Graph/FIA、FP32 LM head。

随机种子也一致：`rollout_seed=42`、Megatron `seed=1234`、SGLang `random_seed=1234`。但两次均为 `enable_deterministic_inference=False`，因此分布式动态调度、NPU 算子和温度采样不会保证逐 token 完全一致。

## 6. 截止停止前的训练结果

已完整完成 step 0–9；停止时正在执行 step 10 rollout（5/8）。最后三个完整 step：

| Step | train/loss | train/mopd_topk_kl |
|---:|---:|---:|
| 7 | 0.048858 | 0.049101 |
| 8 | 0.051924 | 0.051969 |
| 9 | 0.061474 | 0.061693 |

文档原始实验与当前复现的 step 0–8 对比：

| Step | 文档 train/loss | 复现 train/loss | Loss 偏差 | 文档 topk_kl | 复现 topk_kl | TopK KL 偏差 |
|---:|---:|---:|---:|---:|---:|---:|
| 0 | 0.043843 | 0.046100 | +5.15% | 0.043493 | 0.046029 | +5.83% |
| 1 | 0.056370 | 0.061746 | +9.54% | 0.056273 | 0.061324 | +8.98% |
| 2 | 0.055169 | 0.049982 | -9.40% | 0.055507 | 0.049805 | -10.27% |
| 3 | 0.051189 | 0.051960 | +1.51% | 0.051358 | 0.051955 | +1.16% |
| 4 | 0.062446 | 0.065006 | +4.10% | 0.062677 | 0.064761 | +3.33% |
| 5 | 0.042935 | 0.042344 | -1.37% | 0.043293 | 0.042419 | -2.02% |
| 6 | 0.057326 | 0.039570 | -30.97% | 0.057283 | 0.039699 | -30.70% |
| 7 | 0.056678 | 0.048858 | -13.80% | 0.056457 | 0.049101 | -13.03% |
| 8 | 0.057664 | 0.051924 | -9.95% | 0.057313 | 0.051969 | -9.32% |

step 0–6 均值：

- 文档 loss `0.052754`，复现 loss `0.050958`，偏差 `-3.40%`。
- 文档 top-k KL `0.052840`，复现 top-k KL `0.050856`，偏差 `-3.76%`。

step 6 的单步偏差主要来自生成轨迹不同：原实验平均响应 1390.5 tokens、无截断；复现平均响应 8962.8 tokens、2/8 截断。固定 seed 但未启用 deterministic inference 时，这类单步差异是可能的。整体均值仍接近。

旧的替代镜像实验 `_06` 使用 `slime_030_fa:v1`，其 rollout/train logprob 差约 `0.38–0.48`，明显异常；精确镜像 `_07` 恢复到约 `0.008–0.012`，与原实验一致。旧实验日志保留，作业已停止。

## 7. 尚未完成的环节

- 短训练未跑到 20/300 步，因此没有本次实验的周期 checkpoint。
- 长训练 `_07` 未提交。
- checkpoint 转 HF 未执行。
- 短/长模型的三题流式推理未执行。
- 最终图表和完整复现报告未生成。

原先用于自动执行上述后续阶段的 tmux `fengzl_mopd:pipeline` 和远端等待推理任务，在本会话结束时一并停止，避免训练被自动续跑。

## 8. 环境注意事项

Worker 上曾有其他人启动的 `sglang-lrl-516`，占用物理 NPU 2–3，与本任务 rollout 使用的 NPU 0–3 重叠。这可能影响性能和严格可复现性，但截至停止前未出现 OOM、ActorDied、HCCL、RuntimeError 或 Traceback。

不要删除原始权重和上述实验日志。若以后继续，应使用新实验名，或明确选择从已有完整 checkpoint 恢复；当前 `_07` 未到首次周期保存点，不能从 step 10 无损续跑。

## 9. 恢复入口

关键脚本均在：`/hpfs/huawei/fengzl/mopd-delivery-05/manual/`。

- 容器准备：`setup-container.sh`
- Master/Worker 容器：`run-node-container.sh`
- Ray：`start-ray-master.sh`、`start-ray-worker.sh`
- 集群检查：`check-cluster.sh`、`status.sh`
- 短训练：`mopd-adaptation/scripts/run-tele4-topk-customer.sh`
- 长训练：`mopd-adaptation/scripts/run-tele4-topk-longrun.sh`
- 自动全流程：`follow-full-run.sh`
- 推理等待与验证：`follow-inference.sh`

重新运行前必须修改实验名，避免覆盖 `2026-09-23_fengzl_short_07`。

## 10. 会话结束状态

- 自动流水线 tmux 窗口已关闭，不会自动提交长训练。
- Worker 内的推理等待 tmux 会话已关闭。
- Ray 短训练作业已停止，Master 上 16 张 NPU 已无本任务进程。
- `fengzl-mopd-master` 和 `fengzl-mopd-worker` 均已停止，容器本身保留，便于核查；未删除镜像、日志、模型或实验目录。
- 本机只保留用户原有的 `mashine2` SSH tmux 连接。
