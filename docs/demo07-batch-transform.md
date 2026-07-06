# Demo07 — Batch Transform 批量推理

## 实验简介

使用 Batch Transform 对 S3 中的大批量离线数据一次性推理，适合无需实时响应、按批处理的场景（如夜间批跑）。

**实验目标：**
- 理解 Transform Job 与实时端点的适用边界
- 掌握批量输入输出的 S3 组织方式
- 能配置并行度参数控制吞吐

**实验流程：**
1. 准备批量输入数据到 S3
2. 配置并提交 Transform Job
3. 轮询至 `Completed`
4. 验证输出结果格式与条数

**预计 AI 执行时长：** 10 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateTransformJob）、S3（读写）
- **前提**：Demo03 已完成（有可用模型工件）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export MODEL_DATA_URL=<Demo03 输出的模型 S3 路径>
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

## 步骤

### 1. 准备批量输入（无 label 特征列）

```bash
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} \
  | cut -d, -f2- > /tmp/batch_in.csv
wc -l /tmp/batch_in.csv
aws s3 cp /tmp/batch_in.csv s3://${SM_BUCKET}/batch-input/batch_in.csv --region ${AWS_REGION}
```

**预期输出**：打印行数（如 `114`），并上传成功

### 2. 注册 Model 并提交 Transform Job

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              sagemaker_session=sagemaker.Session())
transformer = model.transformer(
    instance_count=1, instance_type="ml.m5.xlarge",
    output_path=f"s3://{bucket}/batch-output/",
    strategy="MultiRecord", assemble_with="Line",
)
transformer.transform(f"s3://{bucket}/batch-input/batch_in.csv",
                      content_type="text/csv", split_type="Line", wait=True, logs=False)
print("JOB_NAME=", transformer.latest_transform_job.name)
PY
```

**预期输出**：作业约 3-4 分钟走完（进度条 `....!`），打印 `JOB_NAME= sagemaker-xgboost-<时间戳>`，状态 `Completed`。

### 3. 验证输出

```bash
aws s3 ls s3://${SM_BUCKET}/batch-output/ --region ${AWS_REGION}
aws s3 cp s3://${SM_BUCKET}/batch-output/batch_in.csv.out - --region ${AWS_REGION} | head -3
```

**预期输出**：存在 `batch_in.csv.out`（约 2.2 KB），每行一个概率值，共 114 行，与输入 114 行一致。示例前 3 行：`0.00047...` / `0.99940...` / `0.00187...`。

---

## 验收标准

- [ ] Transform Job 状态 `Completed`
- [ ] 输出对象存在，且行数与输入行数一致
- [ ] 能说出 Batch Transform 与实时端点的适用边界

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-transform-job --transform-job-name ${JOB_NAME} --region ${AWS_REGION} --query 'TransformJobStatus' --output text` | `Completed` |
| 2 | `aws s3 cp s3://${SM_BUCKET}/batch-output/batch_in.csv.out - --region ${AWS_REGION} \| wc -l` | 与输入行数一致 |

---

## 实验总结

本实验用 Batch Transform 对整批离线数据一次性推理，无需常驻端点、跑完即释放资源，适合夜间批跑等无实时性要求的场景。它与实时/Serverless 端点共同构成 SageMaker 的三类基础推理形态。

---

## 清理

```bash
aws s3 rm s3://${SM_BUCKET}/batch-input/ --recursive --region ${AWS_REGION}
aws s3 rm s3://${SM_BUCKET}/batch-output/ --recursive --region ${AWS_REGION}
echo "清理完成（Transform Job 元数据保留，不计费）"
```
