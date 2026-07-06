# Demo02 — 数据预处理 Processing Job

## 实验简介

使用 SageMaker Processing Job（SKLearn Processor）对原始数据做清洗、特征工程与训练/验证集切分，产出可供 Demo03 训练直接消费的数据集，补齐「原始数据 → 训练」这一环，让后续训练不再假设数据已就绪。

**实验目标：**
- 理解 Processing Job 与 Training Job 的分工（数据处理 vs 模型训练）
- 掌握 SKLearn Processor 的输入/输出 S3 通道约定
- 能产出规范的 train/validation 数据集供后续训练复用

**实验流程：**
1. 上传原始数据集到 S3 `raw/` 前缀
2. 编写预处理脚本（清洗 + 切分 train/validation）
3. 提交 Processing Job（SKLearn Processor）并轮询至 `Completed`
4. 验证 `train/`、`validation/` 输出并记录 S3 路径供 Demo03 引用

**预计 AI 执行时长：** 10-12 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateProcessingJob）、S3（读写数据桶）
- **前提**：Demo01 已完成（执行角色、S3 桶就绪）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

## 步骤

### 1. 准备并上传原始数据到 S3 `raw/`

```bash
# 用 sklearn 自带 breast_cancer 数据集做演示（二分类，label 在首列，无表头）
python3 - <<'PY'
import os, pandas as pd
from sklearn.datasets import load_breast_cancer
d = load_breast_cancer(as_frame=True)
df = pd.concat([d.target.rename("label"), d.data], axis=1)
df.to_csv("/tmp/raw.csv", index=False, header=True)
print(df.shape)
PY
aws s3 cp /tmp/raw.csv s3://${SM_BUCKET}/raw/raw.csv --region ${AWS_REGION}
```

**预期输出**：打印 `(569, 31)`，并 `upload: /tmp/raw.csv to s3://.../raw/raw.csv`

### 2. 编写预处理脚本

```bash
cat > /tmp/preprocess.py <<'PY'
import pandas as pd
from sklearn.model_selection import train_test_split
df = pd.read_csv("/opt/ml/processing/input/raw.csv")
train, val = train_test_split(df, test_size=0.2, random_state=42, stratify=df["label"])
# 内置 XGBoost 要求：label 首列、无表头
train.to_csv("/opt/ml/processing/train/train.csv", index=False, header=False)
val.to_csv("/opt/ml/processing/validation/validation.csv", index=False, header=False)
print("train", train.shape, "val", val.shape)
PY
```

### 3. 提交 Processing Job

```bash
python3 - <<'PY'
import os
from sagemaker.sklearn.processing import SKLearnProcessor
from sagemaker.processing import ProcessingInput, ProcessingOutput

bucket = os.environ["SM_BUCKET"]; role = os.environ["SM_ROLE_ARN"]
proc = SKLearnProcessor(framework_version="1.2-1", role=role,
                        instance_type="ml.m5.xlarge", instance_count=1)
proc.run(
    code="/tmp/preprocess.py",
    inputs=[ProcessingInput(source=f"s3://{bucket}/raw/raw.csv",
                            destination="/opt/ml/processing/input")],
    outputs=[ProcessingOutput(source="/opt/ml/processing/train",
                              destination=f"s3://{bucket}/train"),
             ProcessingOutput(source="/opt/ml/processing/validation",
                              destination=f"s3://{bucket}/validation")],
    wait=True, logs=False,   # logs=False 避免 3.9 上 CloudWatch 拉日志的噪声，仅等待完成
)
print("JOB_NAME=", proc.latest_job.job_name)
PY
```

**预期输出**：作业约 2 分钟完成（`.....!`），末行打印 `JOB_NAME= sagemaker-scikit-learn-<时间戳>`。

> ✅ 实测：`framework_version="1.2-1"` 有效（镜像 `683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-scikit-learn:1.2-1-cpu-py3`）。SDK 2.235.2 不支持 `1.2-2`/`1.5-1`（会报 `Unsupported sklearn version`）。

### 4. 验证输出

```bash
aws s3 ls s3://${SM_BUCKET}/train/ --region ${AWS_REGION}
aws s3 ls s3://${SM_BUCKET}/validation/ --region ${AWS_REGION}
```

**预期输出**：`train/train.csv`（约 117 KB，455 行）、`validation/validation.csv`（约 29 KB，114 行）。首列为 label（0/1），共 31 列，无表头 —— 满足内置 XGBoost 要求。

---

## 验收标准

- [ ] Processing Job 状态 `Completed`
- [ ] `s3://${SM_BUCKET}/train/train.csv` 与 `validation/validation.csv` 存在
- [ ] 输出 CSV 为「label 首列、无表头」格式（满足内置 XGBoost 要求）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-processing-job --processing-job-name ${JOB_NAME} --region ${AWS_REGION} --query 'ProcessingJobStatus' --output text` | `Completed` |
| 2 | `aws s3 ls s3://${SM_BUCKET}/train/train.csv --region ${AWS_REGION} \| wc -l` | `1` |
| 3 | `aws s3 cp s3://${SM_BUCKET}/train/train.csv - --region ${AWS_REGION} \| head -1 \| cut -d, -f1` | `0` 或 `1`（首列为 label） |

---

## 实验总结

本实验用 SKLearn Processing Job 把原始数据加工成规范的 train/validation 数据集，明确了 Processing Job（数据）与 Training Job（模型）的职责边界。产出的数据集是 Demo03 训练的直接输入，也让整条链路从「假设数据已就绪」变为「数据可复现」。

---

## 清理

```bash
# 如 Demo03 仍需训练数据，请勿删除 train/ 与 validation/
aws s3 rm s3://${SM_BUCKET}/raw/ --recursive --region ${AWS_REGION}
echo "清理完成（保留 train/validation 供 Demo03）"
```
