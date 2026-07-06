# Global SageMaker QuickStart
## Amazon SageMaker Hands-on Lab Collection

基于 SageMaker（Studio / XGBoost 内置算法 / JumpStart）· 全球区（us-east-1）· Prompt 驱动执行

---

## Demo 列表

建议先完成 **基础**，再按角色和时间选择 **进阶**。基础覆盖从准备环境、数据处理、训练、部署、弹性/批量推理到监控与权限的最小闭环；进阶用于补充高级推理模式、模型监控、MLOps 生产化、生成式 AI（JumpStart）和成本优化专题。

## 基础

### 1. 基础环境

- [Demo01 — 准备实验环境与 SageMaker Domain](docs/demo01-studio-setup.md)

### 2. 数据、训练与部署

- [Demo02 — 数据预处理 Processing Job](docs/demo02-data-processing.md)
- [Demo03 — 内置算法 Training Job（XGBoost）](docs/demo03-builtin-training.md)
- [Demo04 — 部署实时推理端点并调用](docs/demo04-realtime-endpoint.md)
- [Demo05 — 端点健康检查与故障排查](docs/demo05-endpoint-troubleshoot.md)

### 3. 弹性与批量推理

- [Demo06 — Serverless Inference 无服务器推理](docs/demo06-serverless-inference.md)
- [Demo07 — Batch Transform 批量推理](docs/demo07-batch-transform.md)
- [Demo08 — Endpoint 自动伸缩](docs/demo08-endpoint-autoscaling.md)

### 4. 权限与可观测性

- [Demo09 — IAM 执行角色与权限最小化](docs/demo09-iam-execution-role.md)
- [Demo10 — CloudWatch 全链路可观测性](docs/demo10-cloudwatch-observability.md)

## 进阶

### 5. 高级推理模式

- [Demo11 — Async Inference 异步推理](docs/demo11-async-inference.md)
- [Demo12 — Multi-Model Endpoint 多模型端点](docs/demo12-multi-model-endpoint.md)

### 6. 模型质量与 MLOps

- [Demo13 — Model Monitor 数据漂移检测](docs/demo13-model-monitor.md)
- [Demo14 — 自带容器 BYOC](docs/demo14-byoc.md)
- [Demo15 — 超参数调优 Tuning Job](docs/demo15-hyperparameter-tuning.md)
- [Demo16 — Feature Store 特征存储](docs/demo16-feature-store.md)
- [Demo17 — SageMaker Pipelines 模型流水线](docs/demo17-pipelines.md)
- [Demo18 — Model Registry 与蓝绿/影子部署](docs/demo18-model-registry-deployment.md)

### 7. 生成式 AI 与成本优化

- [Demo19 — JumpStart 部署开源大模型](docs/demo19-jumpstart-llm.md)
- [Demo20 — 推理成本优化（Inferentia/Graviton/Savings Plans）](docs/demo20-cost-optimization.md)

---

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载执行规则与环境变量约定
2. 将对应 [`docs/`](docs/) 目录中的 Demo 文件内容粘贴到对话框，由 AI 自主执行
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| Python | 3.10+ |
| boto3 / sagemaker SDK | latest |
| SageMaker Domain | Studio（JupyterLab 3 体验） |

## 命名与执行角色约定

| 项目 | 约定 |
|------|------|
| Region | `us-east-1` |
| Domain 名称 | `demo-domain` |
| 执行角色 | `SageMakerQuickStartExecutionRole` |
| 训练/推理示例算法 | XGBoost（内置算法，`sagemaker.image_uris.retrieve`） |
| GenAI 示例 | JumpStart 开源 LLM（GPU 端点，注意配额与成本） |
| S3 数据桶 | `s3://sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}/` |
