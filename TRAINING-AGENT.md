# 给另一个 AI 的训练接口

启动服务后访问 `/training`。本地演示服务端口为 8771；常规脚本/Docker 默认 8768。通过 API 可执行受信任的 Python 训练脚本。路径检查和禁用 shell **不是脚本沙箱**，脚本仍拥有运行账号权限。

## HTTP

- `POST /api/training`：提交，立即返回 job_id。
- `GET /api/training`：任务列表、状态、日志尾部。
- `GET /api/training/{job_id}`：真实退出码、指标、产物清单。
- `POST /api/training/{job_id}/stop`：本地请求停止；AWS 调用对应停止接口。
- `GET /training-files/{job_id}/{relative_path}`：下载本地产物。

提交请求（JSON）：
```json
{
  "game_id":"experiment-001",
  "script":"/workspace/train.py",
  "workdir":"/workspace",
  "output_dir":"/workspace/runs/experiment-001",
  "args":["--data","/workspace/dataset/data.yaml","--epochs","30"],
  "backend":"local",
  "max_seconds":7200
}
```
Windows 原生运行请改为实际绝对路径。脚本、工作目录、输出目录必须位于服务 `--allowed-root` 下；新任务输出目录须为空。args 为参数数组，不是整条 shell 命令。脚本与依赖由操作者预先准备，不自动 pip install。

脚本读取 `NBA_OUTPUT_DIR` 保存权重、报告。可持续写入 `metrics.json`，API 会读出该文件；YOLO 的 results.csv 会作为产物保留。状态包括 queued/running/completed/failed/stopped/timed_out；只有退出码为 0 才标记 completed。

`examples/train_smoke.py` 仅验证调度，产物不是模型权重。`examples/train_yolo.py` 是实际 YOLO 训练模板，需要你自己的完整审核数据和初始模型。

## CLI

```powershell
python -m nba_assist.portable train --game-id experiment-001 --script workspace/train.py --workdir workspace --output-dir workspace/runs/experiment-001 --root workspace
python -m nba_assist.portable train-status --root workspace --job-id JOB_ID
python -m nba_assist.portable train-stop --root workspace --job-id JOB_ID
```
带 `--epochs` 等嵌套参数建议使用 HTTP JSON，避免命令行解析歧义。重试请新建任务和输出目录；恢复训练需要脚本本身支持 checkpoint 参数，本工具不猜测如何恢复权重。

## 4090 远端

可在 4090 安装同一工具，指定其项目目录为 `--allowed-root`，通过 SSH 隧道调用完全相同 API。本轮未重新部署远端版本或启动远端训练。

## AWS

设置 backend 为 `aws-batch` 或 `sagemaker`，并使用 LOCAL-AWS.md 中的队列、镜像、角色设置，额外配置 `NBA_TRAINING_S3_OUTPUT=s3://your-bucket/training`。

脚本和数据必须预装进自定义容器或挂载到 `/workspace`；当前训练入口不会自动把 Windows 脚本和数据打包上传。示例派生镜像：
```dockerfile
FROM nba-assist:local
COPY your-training-project /workspace
```
模型输出请写入 `/workspace/runs/...`，worker 会上传至 `NBA_TRAINING_S3_OUTPUT/JOB_ID/`，包括日志和状态。云作业使用 IAM 角色，凭据不写进任务 JSON。SageMaker 的默认 ModelArtifacts 不是这些自定义产物的路径，以上述 S3 前缀为准。API 查询云端任务返回调度器状态；Batch 日志位置可从 container.logStreamName 查询 CloudWatch。

当前训练任务可以并行启动，**没有跨推理/训练的统一 GPU 调度器**；另一个 AI 应先查看任务列表，避免重叠占用同一 GPU。服务异常重启不会把未结束任务伪报成功；主机重启后的状态可能残留 running，需核实后重新提交。未实现自动优化超参或根据指标自动重训。

验证：32 项本地测试通过，包含真实子进程成功/失败/停止/超时。AWS 训练未实际运行；现有 AWS SDK Stubber 测试主要覆盖识别任务接口，不能替代云端训练验收。
