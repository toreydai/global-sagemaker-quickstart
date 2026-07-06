# Demo11 — Async Inference 异步推理

## 实验简介

针对大负载或长耗时推理场景（如大文件、长文本），使用 Async Inference 端点，通过 S3 输入输出与 SNS 通知实现异步调用模式。

**实验目标：**
- 理解 Async Inference 的适用场景（大 payload/长耗时/可容忍延迟）
- 掌握 S3 输入输出 + SNS 通知的配置方式
- 能验证异步结果回收流程

**实验流程：**
1. 配置 `AsyncInferenceConfig`（S3 输出路径 + SNS 主题）
2. 部署异步端点
3. 提交大 payload 异步请求
4. 从 SNS 通知/S3 输出中获取结果

**预计 AI 执行时长：** 12 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker、SNS、S3
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

> ⚠️ **实测坑（SNS 名称必须含 `sagemaker`）**：`AmazonSageMakerFullAccess` 只允许执行角色 `sns:Publish` 到名称匹配 `*SageMaker*`/`*Sagemaker*`/`*sagemaker*` 的主题。若 SNS 主题名不含 `sagemaker`（如 `sm-quickstart-async`），异步端点会部署 `Failed`，报 `The provided role ... has "sns:Publish" permissions for topic ...`。因此主题名务必含 `sagemaker`。

## 步骤

### 1. 创建 SNS 主题（可选，用于完成通知）

```bash
# 主题名必须含 "sagemaker"，否则执行角色无 sns:Publish 权限（见上方 ⚠️）
export SNS_ARN=$(aws sns create-topic --name sagemaker-quickstart-async --region ${AWS_REGION} --query 'TopicArn' --output text)
echo ${SNS_ARN}
```

**预期输出**：打印 `arn:aws:sns:us-east-1:<account>:sagemaker-quickstart-async`

### 2. 部署异步端点

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.async_inference import AsyncInferenceConfig

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
from sagemaker.predictor import Predictor
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              predictor_cls=Predictor,          # 否则 deploy() 返回 None（见 Demo04）
              sagemaker_session=sagemaker.Session())
predictor = model.deploy(
    initial_instance_count=1, instance_type="ml.m5.large",
    endpoint_name="sm-quickstart-async",
    async_inference_config=AsyncInferenceConfig(
        output_path=f"s3://{bucket}/async-output/",
        notification_config={"SuccessTopic": os.environ.get("SNS_ARN",""),
                             "ErrorTopic": os.environ.get("SNS_ARN","")} if os.environ.get("SNS_ARN") else None,
    ),
)
print("ENDPOINT=", predictor.endpoint_name)
PY
```

**预期输出**：约 5-6 分钟后打印 `ENDPOINT= sm-quickstart-async`（进度条 `------!`）。若报 `sns:Publish` 权限错误，检查主题名是否含 `sagemaker`。

### 3. 上传输入到 S3 并异步调用

```bash
export ENDPOINT_NAME=sm-quickstart-async
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | cut -d, -f2- > /tmp/async_in.csv
aws s3 cp /tmp/async_in.csv s3://${SM_BUCKET}/async-input/async_in.csv --region ${AWS_REGION}

aws sagemaker-runtime invoke-endpoint-async \
  --endpoint-name ${ENDPOINT_NAME} \
  --content-type text/csv \
  --input-location s3://${SM_BUCKET}/async-input/async_in.csv \
  --region ${AWS_REGION}
```

**预期输出**：立即返回（不阻塞）含 `OutputLocation`（如 `s3://.../async-output/<uuid>.out`）与 `InferenceId` 的 JSON。

### 4. 回收结果

```bash
sleep 30
aws s3 ls s3://${SM_BUCKET}/async-output/ --region ${AWS_REGION}
```

**预期输出**：约 15-30 秒后出现 `<uuid>.out`（约 2.2 KB），含 114 行概率值，与输入行数一致（首行 `0.00047...`）。

---

## 验收标准

- [ ] 异步端点 `InService`
- [ ] `invoke-endpoint-async` 立即返回 `OutputLocation`
- [ ] S3 输出路径下生成结果对象

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |
| 2 | `aws sagemaker describe-endpoint-config --endpoint-config-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'AsyncInferenceConfig.OutputConfig.S3OutputPath' --output text` | 以 `s3://${SM_BUCKET}/async-output/` 开头 |

---

## 实验总结

本实验用 Async Inference 端点处理「大 payload / 长耗时 / 可容忍延迟」的推理请求：调用立即返回、结果异步落 S3、可选 SNS 通知。它补齐了实时/Serverless/Batch 之外的第四种推理形态，适合大文件、长文本等场景。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-async --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-async --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/async-input/ --recursive --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/async-output/ --recursive --region ${AWS_REGION} 2>/dev/null || true
aws sns delete-topic --topic-arn ${SNS_ARN} --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
