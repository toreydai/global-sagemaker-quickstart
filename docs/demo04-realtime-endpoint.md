# Demo04 — 部署实时推理端点并调用

## 实验简介

将 Demo03 训练产出的模型部署为实时推理端点（Real-time Endpoint），并通过 SDK/CLI 发起在线推理调用，验证端到端可用性。

**实验目标：**
- 理解 Model → EndpointConfig → Endpoint 三层资源关系
- 掌握实时端点的部署与调用方式
- 能验证端点返回预期推理结果

**实验流程：**
1. 注册 Model（引用 Demo03 模型工件）
2. 创建 EndpointConfig
3. 创建 Endpoint 并等待状态变为 `InService`
4. 发起测试推理请求验证输出

**预计 AI 执行时长：** 8-10 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateModel/CreateEndpointConfig/CreateEndpoint）
- **前提**：Demo03 已完成（有可用模型工件）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export MODEL_DATA_URL=<Demo03 输出的模型 S3 路径>
```

---

> 补充：Step 3 需读 `s3://${SM_BUCKET}/validation/`，请确保已 `export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}`。

## 步骤

### 1. 部署端点（sagemaker SDK 一步完成 Model/Config/Endpoint）

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.predictor import Predictor

region = os.environ["AWS_REGION"]; role = os.environ["SM_ROLE_ARN"]
model = Model(
    image_uri=retrieve("xgboost", region, version="1.7-1"),
    model_data=os.environ["MODEL_DATA_URL"],
    role=role,
    predictor_cls=Predictor,          # 关键：不指定 predictor_cls 时 deploy() 返回 None
    sagemaker_session=sagemaker.Session(),
)
predictor = model.deploy(initial_instance_count=1, instance_type="ml.m5.large",
                         endpoint_name="sm-quickstart-rt")
print("ENDPOINT=", predictor.endpoint_name)
PY
```

**预期输出**：约 5-7 分钟后打印 `ENDPOINT= sm-quickstart-rt`（进度条 `------!`）。

> ⚠️ **实测坑**：基类 `Model.deploy()` 只有在传了 `predictor_cls` 时才返回 predictor 对象，否则返回 `None`，直接 `predictor.endpoint_name` 会报 `AttributeError`。草稿已加 `predictor_cls=Predictor`。端点名固定为 `sm-quickstart-rt`。

### 2. 轮询端点状态

```bash
export ENDPOINT_NAME=sm-quickstart-rt
aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} \
  --region ${AWS_REGION} --query 'EndpointStatus' --output text
```

**预期输出**：`InService`

### 3. 发起推理请求

```bash
# 取一行验证数据（去掉首列 label）作为推理输入，写入文件
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name ${ENDPOINT_NAME} \
  --content-type text/csv \
  --cli-binary-format raw-in-base64-out \
  --body fileb:///tmp/payload.csv \
  --region ${AWS_REGION} /tmp/out.json >/dev/null
cat /tmp/out.json; echo
```

**预期输出**：返回一个 0~1 之间的概率值。实测该样本（真实 label=0）返回约 `0.00047`（接近 0，预测正确）。

> ⚠️ **实测坑（AWS CLI v2 blob）**：CLI v2 默认把 `--body` 字符串当作 base64 解码，直接传 `--body "${PAYLOAD}"` 会报 `415 ... UnicodeDecodeError: 'utf-8' codec can't decode byte 0xd7`。必须加 `--cli-binary-format raw-in-base64-out` 并用 `--body fileb://<文件>`。

---

## 验收标准

- [ ] 端点状态 `InService`
- [ ] `invoke-endpoint` 返回合法概率值（0~1）
- [ ] 能说清 Model/EndpointConfig/Endpoint 三层关系

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |
| 2 | `cat /tmp/out.json` | 单个 0~1 浮点数 |

---

## 实验总结

本实验把训练工件部署为可在线调用的实时端点，串起了 Model→EndpointConfig→Endpoint 三层资源。该端点是后续排障（Demo05）、自动伸缩（Demo08）、CloudWatch 监控（Demo10）、Model Monitor（Demo13）的共同对象；请保留至这些 Demo 完成后再清理。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name ${ENDPOINT_NAME} --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成（如后续 Demo 仍需此端点，请勿执行）"
```
