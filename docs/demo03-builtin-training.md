# Demo03 — 内置算法 Training Job（XGBoost）

## 实验简介

使用 SageMaker 内置 XGBoost 算法，基于 Demo02 预处理产出的数据集提交一个 Training Job，产出可供后续 Demo 部署的模型工件（model.tar.gz）。

**实验目标：**
- 理解 Training Job 的输入（数据 + 超参数 + 算力配置）与输出（S3 模型工件）
- 掌握内置算法镜像的获取方式（`sagemaker.image_uris.retrieve`）
- 能够独立提交并监控训练任务至完成

**实验流程：**
1. 引用 Demo02 产出的 train/validation 数据集
2. 获取 XGBoost 内置算法镜像 URI
3. 配置 Estimator 并提交 Training Job
4. 轮询至 `Completed`，记录模型 S3 路径供后续 Demo 引用

**预计 AI 执行时长：** 10-12 分钟（含训练等待）

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateTrainingJob）、S3（读写数据桶）
- **前提**：Demo01、Demo02 已完成（执行角色、S3 桶、预处理后的训练数据就绪）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

## 步骤

### 1. 确认 Demo02 产出的训练数据就位

```bash
aws s3 ls s3://${SM_BUCKET}/train/ --region ${AWS_REGION}
aws s3 ls s3://${SM_BUCKET}/validation/ --region ${AWS_REGION}
```

**预期输出**：两个前缀下各至少有一个 CSV 对象（如 `train.csv`、`validation.csv`）；若为空，先回到 Demo02。

> ⚠️ 内置 XGBoost 要求 CSV **首列为 label 且无表头**。Demo02 预处理需保证该格式，否则训练会报列解析错误。

### 2. 获取 XGBoost 内置算法镜像 URI

```bash
python3 - <<'PY'
from sagemaker.image_uris import retrieve
import os
uri = retrieve("xgboost", os.environ["AWS_REGION"], version="1.7-1")
print(uri)
PY
```

**预期输出**：形如 `683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1`（账号号段随 Region 变化）。

### 3. 提交 Training Job（sagemaker SDK）

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.estimator import Estimator
from sagemaker.inputs import TrainingInput
from sagemaker.image_uris import retrieve

region = os.environ["AWS_REGION"]
role   = os.environ["SM_ROLE_ARN"]
bucket = os.environ["SM_BUCKET"]

image_uri = retrieve("xgboost", region, version="1.7-1")

xgb = Estimator(
    image_uri=image_uri,
    role=role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    output_path=f"s3://{bucket}/model/",
    sagemaker_session=sagemaker.Session(),
)
xgb.set_hyperparameters(
    objective="binary:logistic",
    num_round=100, max_depth=5, eta=0.2, subsample=0.8,
)
xgb.fit({
    "train":      TrainingInput(f"s3://{bucket}/train/",      content_type="csv"),
    "validation": TrainingInput(f"s3://{bucket}/validation/", content_type="csv"),
})
print("JOB_NAME=", xgb.latest_training_job.name)
print("MODEL_DATA=", xgb.model_data)
PY
```

**预期输出**：日志约 2-3 分钟走完 `Starting → Downloading → Training → Uploading → Completed - Training job completed`，并打印 `JOB_NAME= sagemaker-xgboost-<时间戳>` 与 `MODEL_DATA=s3://.../model/<job>/output/model.tar.gz`（实测 `model.tar.gz` 约 12 KB）。**记下 `MODEL_DATA` 供 Demo04/06/07/11/20 引用。**

> ✅ 实测：`version="1.7-1"` 镜像有效，超参 `objective=binary:logistic` 正常训练完成。`fit(..., logs=False)` 只等待不刷全量日志，适合本机 Python 3.9。

### 4. 轮询确认任务状态（CLI 二次校验）

```bash
export JOB_NAME=<上一步打印的 JOB_NAME>
aws sagemaker describe-training-job --training-job-name ${JOB_NAME} \
  --region ${AWS_REGION} --query 'TrainingJobStatus' --output text
```

**预期输出**：`Completed`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Training Job 状态为 `Completed`
- [ ] `s3://${SM_BUCKET}/model/.../output/model.tar.gz` 存在
- [ ] 能说出 `train`/`validation` 两个数据通道的作用与 CSV 格式要求
- [ ] 记录了 `MODEL_DATA` S3 URI 供后续 Demo 引用

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-training-job --training-job-name ${JOB_NAME} --region ${AWS_REGION} --query 'TrainingJobStatus' --output text` | `Completed` |
| 2 | `aws sagemaker describe-training-job --training-job-name ${JOB_NAME} --region ${AWS_REGION} --query 'ModelArtifacts.S3ModelArtifacts' --output text` | 以 `s3://${SM_BUCKET}/model/` 开头、以 `/output/model.tar.gz` 结尾 |
| 3 | `aws s3 ls $(aws sagemaker describe-training-job --training-job-name ${JOB_NAME} --region ${AWS_REGION} --query 'ModelArtifacts.S3ModelArtifacts' --output text)` | 输出一行，大小非 0 |

---

## 实验总结

本实验用内置 XGBoost 算法完成了一次标准训练：获取托管算法镜像、配置 Estimator 与超参数、以 `train`/`validation` 双通道提交 Training Job，产出 `model.tar.gz` 模型工件。该工件是后续实时端点（Demo04）、各推理形态（Demo06/07/11）与成本对比（Demo20）的共同输入。下一个实验将把它部署为实时推理端点。

---

## 清理

```bash
# Training Job 无法删除（仅保留元数据，不计费）；如需清理模型工件与中间数据：
aws s3 rm s3://${SM_BUCKET}/model/ --recursive --region ${AWS_REGION}
# 如后续 Demo 仍需 MODEL_DATA，请勿执行上一条
echo "清理完成（训练任务元数据保留，不产生费用）"
```
