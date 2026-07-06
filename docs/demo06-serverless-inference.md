# Demo06 — Serverless Inference 无服务器推理

## 实验简介

将 Demo03 的模型改用 Serverless Inference 部署，对比与实时端点在成本、冷启动、并发上的差异，适合流量不稳定或间歇性调用的场景。

**实验目标：**
- 理解 Serverless Inference 的适用场景与限制（内存/最大并发配置）
- 掌握 Serverless 端点的部署与调用方式
- 能对比 Serverless 与实时端点两种类型的成本模型

**实验流程：**
1. 配置 `ServerlessConfig`（内存大小、最大并发）
2. 部署 Serverless 端点
3. 调用端点并观察冷启动延迟
4. 与 Demo04 实时端点做成本/延迟对比

**预计 AI 执行时长：** 8 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateEndpointConfig with ServerlessConfig）
- **前提**：Demo03 已完成（有可用模型工件）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export MODEL_DATA_URL=<Demo03 输出的模型 S3 路径>
```

---

## 步骤

### 1. 部署 Serverless 端点

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.serverless import ServerlessInferenceConfig
from sagemaker.predictor import Predictor

region = os.environ["AWS_REGION"]; role = os.environ["SM_ROLE_ARN"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              predictor_cls=Predictor,          # 否则 deploy() 返回 None（见 Demo04）
              sagemaker_session=sagemaker.Session())
predictor = model.deploy(
    endpoint_name="sm-quickstart-serverless",
    serverless_inference_config=ServerlessInferenceConfig(memory_size_in_mb=2048, max_concurrency=5),
)
print("ENDPOINT=", predictor.endpoint_name)
PY
```

**预期输出**：约 5 分钟后打印 `ENDPOINT= sm-quickstart-serverless`（进度条 `-----!`）

### 2. 轮询状态

```bash
export ENDPOINT_NAME=sm-quickstart-serverless
aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'EndpointStatus' --output text
```

**预期输出**：`InService`

### 3. 调用并观察冷启动

```bash
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
for i in 1 2; do
  echo "第 ${i} 次调用（第 1 次可能含冷启动）："
  /usr/bin/time -f "%e s" aws sagemaker-runtime invoke-endpoint --endpoint-name ${ENDPOINT_NAME} \
    --content-type text/csv --cli-binary-format raw-in-base64-out \
    --body fileb:///tmp/payload.csv --region ${AWS_REGION} /tmp/sl.json >/dev/null
  cat /tmp/sl.json; echo
done
```

**预期输出**：两次均返回 `0.00047...`。

> ℹ️ 实测本例两次调用都约 `0.5s`（且主要是 AWS CLI 进程启动开销），**未观察到明显冷启动差异**——因为模型极小（12 KB）、部署后容器仍温、且间隔很短。要放大冷启动效果需：更大模型、部署后等待端点缩容到 0（数分钟无调用）再首次调用。invoke 同样需 `--cli-binary-format raw-in-base64-out` + `fileb://`（见 Demo04）。

### 4. 成本对比说明

| 维度 | 实时端点（Demo04） | Serverless（本 Demo） |
|------|------|------|
| 计费 | 按实例在线时长 | 按调用时长 + 内存 |
| 空闲成本 | 有（实例常驻） | 无（缩容到 0） |
| 冷启动 | 无 | 有 |
| 适用 | 稳定/高频流量 | 间歇/低频流量 |

---

## 验收标准

- [ ] Serverless 端点 `InService`
- [ ] 两次调用均返回合法概率值，且能观察到冷/热启动延迟差异
- [ ] 能说清与实时端点的成本模型区别

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |
| 2 | `aws sagemaker describe-endpoint-config --endpoint-config-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'ProductionVariants[0].ServerlessConfig.MaxConcurrency' --output text` | `5` |

---

## 实验总结

本实验将同一模型部署为 Serverless 端点，直观对比了它与实时端点在计费、空闲成本、冷启动上的取舍。Serverless 适合间歇/低频流量、追求「零空闲成本」的场景，是实时端点之外的重要部署形态。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-serverless --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-serverless --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
