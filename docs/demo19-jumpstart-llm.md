# Demo19 — JumpStart 部署开源大模型

## 实验简介

使用 SageMaker JumpStart 一键部署开源基础模型（如 Llama 系列或同级开源 LLM）到实时端点，体验托管 GenAI 推理，并与前面自训练 XGBoost 的部署路径做对比——这是 SageMaker 在生成式 AI 时代最常被客户问到的落地方式。

**实验目标：**
- 理解 JumpStart 模型中心与一键部署机制
- 掌握 LLM 端点的 GPU 实例选型与调用方式（含流式输出）
- 能对比「自训练模型」与「JumpStart 预训练大模型」两条落地路径

**实验流程：**
1. 在 JumpStart 中检索目标开源模型
2. 选择 GPU 实例一键部署到实时端点
3. 发起文本生成请求（含流式输出）
4. 与自训练模型部署路径对比总结

**预计 AI 执行时长：** 12-18 分钟（含大模型端点拉起等待）

> ⚠️ LLM 端点使用 GPU 实例（如 `ml.g5.2xlarge`），**单价显著高于前面的 CPU 端点**，且需要账号具备对应 GPU 实例配额。执行前确认配额，实验后**第一时间删除端点**，避免高额计费。

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（JumpStart 部署）、相应 GPU 实例配额
- **前提**：Demo01 已完成；账号具备目标 GPU 实例配额（如 `ml.g5.2xlarge`）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

> ✅ **已在真实账号 ACCOUNT_ID_REDACTED / us-east-1 验证**（实测 `model_id`=`huggingface-llm-falcon-7b-instruct-bf16`、实例 `ml.g5.2xlarge`、TGI 2.4.0 镜像、payload/返回见下）。JumpStart 模型 ID 更新快、GPU 实例要求随模型变化，换模型时仍需核对。

## 步骤

### 1. 检索并确认目标 JumpStart 模型

```bash
python3 - <<'PY'
from sagemaker.jumpstart.notebook_utils import list_jumpstart_models
# 列出文本生成类模型，挑一个能在单卡 GPU 跑起来的开源 LLM
models = [m for m in list_jumpstart_models() if any(k in m.lower() for k in ["mistral-7b","llama-3-8b","falcon-7b"]) and "instruct" in m.lower()]
print("\n".join(models[:20]))
PY
```

**预期输出**：打印一批可用文本生成模型 ID（含 `huggingface-llm-falcon-7b-instruct-bf16`、`meta-textgeneration-llama-3-8b-instruct`、`huggingface-llm-mistral-7b-instruct` 等）。

> ⚠️ **实例选型是本 Demo 最大的坑**：`mistral-7b-instruct` 与 `llama-3-8b-instruct` 的 **JumpStart 默认实例是 `ml.g5.12xlarge`（4 卡 A10G）**，多卡实例配额通常为 0，且 Llama 系列为 gated 模型需 `accept_eula=True`。本 Demo 选 **`huggingface-llm-falcon-7b-instruct-bf16`**：非 gated（无需 EULA），**默认实例即单卡 `ml.g5.2xlarge`**，最容易在配额内跑通。用如下代码核对候选模型的默认实例与 gated 状态：
>
> ```bash
> python3 - <<'PY'
> from sagemaker.jumpstart.model import JumpStartModel
> m = JumpStartModel(model_id="huggingface-llm-falcon-7b-instruct-bf16")
> print("instance_type:", m.instance_type)   # ml.g5.2xlarge
> print("image_uri:", m.image_uri)            # huggingface-pytorch-tgi-inference:2.4.0-tgi2.4.0-gpu-py311-cu124
> PY
> ```

```bash
export JS_MODEL_ID=huggingface-llm-falcon-7b-instruct-bf16
```

### 2. 一键部署到实时端点（GPU）

```bash
python3 - <<'PY'
import os
from sagemaker.jumpstart.model import JumpStartModel
# ⚠️ 必须显式传 role：否则 SDK 用当前调用方身份（如 AdminRole）当执行角色，
# 该角色不含 sagemaker.amazonaws.com 信任 / JumpStart 缓存桶 s3:GetObject，CreateModel 会 ValidationException 失败。
ROLE = "arn:aws:iam::<account-id>:role/SageMakerQuickStartExecutionRole"
model = JumpStartModel(model_id=os.environ["JS_MODEL_ID"], role=ROLE)
# Falcon 非 gated，不需要 accept_eula；gated 模型（如 Llama）才需 model.deploy(accept_eula=True, ...)
predictor = model.deploy(endpoint_name="sm-quickstart-llm")
print("ENDPOINT=", predictor.endpoint_name)
PY
export ENDPOINT_NAME=sm-quickstart-llm
```

**预期输出**：约 **9-10 分钟**（实测 08:41 → 08:50，约 9.5 分钟）后打印 `ENDPOINT= sm-quickstart-llm`

> ⚠️ 使用 GPU 实例 `ml.g5.2xlarge`，单价高——记录部署起止时间，实验后立即清理。**强烈建议把「部署 → 调用 → 删除」放在同一个脚本里**（见 `run_demo19_all.py`），端点一 `InService` 就调用并立即删，最大限度缩短 GPU 计费窗口。

### 3. 发起文本生成请求

```bash
python3 - <<'PY'
from sagemaker.predictor import Predictor
from sagemaker.serializers import JSONSerializer
from sagemaker.deserializers import JSONDeserializer
p = Predictor(endpoint_name="sm-quickstart-llm",
              serializer=JSONSerializer(), deserializer=JSONDeserializer())
# TGI 文本生成 payload schema（已核对）：{"inputs": <str>, "parameters": {...}}
resp = p.predict({"inputs": "In one sentence, explain what Amazon SageMaker is:",
                  "parameters": {"max_new_tokens": 64, "return_full_text": False}})
print(type(resp), resp)
PY
```

**预期输出**：TGI 容器返回一个 **JSON 数组**（Python `list`），首元素是 `{"generated_text": "..."}`：

```python
<class 'list'> [{'generated_text': ' <RESP_PLACEHOLDER>'}]
```

> ⚠️ **返回结构是 `list`（非 HF 常见的单个 dict）**——取生成文本要用 `resp[0]["generated_text"]`。若 payload 里带 `"details": True`，元素还会多出 `details`（token 级信息）；不带则只有 `generated_text`。`parameters` 常用键：`max_new_tokens`、`return_full_text`（False 只返回续写部分）、`temperature`、`top_p`、`stop`。

### 4. 与自训练模型部署路径对比

| 维度 | 自训练 XGBoost（Demo03/04） | JumpStart LLM（本 Demo） |
|------|------|------|
| 训练 | 需自备数据训练 | 预训练，开箱即用 |
| 实例 | CPU（ml.m5.large） | GPU（ml.g5.x） |
| 成本 | 低 | 高（GPU） |
| 适用 | 结构化预测 | 文本生成/理解 |

---

## 验收标准

- [x] 成功从 JumpStart 检索并选定一个可用开源 LLM（`huggingface-llm-falcon-7b-instruct-bf16`）
- [x] GPU 端点 `InService`（`ml.g5.2xlarge`，约 9.5 分钟拉起）
- [x] 文本生成请求返回合理结果（`list` → `[{"generated_text": ...}]`）
- [x] 能对比自训练与 JumpStart 两条路径的取舍

---

## 验证检查点

> ⚠️ GPU 端点计费高，本 Demo 推荐用同一脚本「部署→调用→立即删」（`run_demo19_all.py`），端点存活窗口极短。下面的检查点仅在端点仍 `InService` 时可跑；若已删除，改用「清理后」检查点确认无遗留。

**端点存活期间：**

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name sm-quickstart-llm --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |
| 2 | `aws sagemaker describe-endpoint-config --endpoint-config-name sm-quickstart-llm --region ${AWS_REGION} --query 'ProductionVariants[0].InstanceType' --output text` | `ml.g5.2xlarge` |

**清理后（确认无 GPU 计费遗留）：**

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 3 | `aws sagemaker list-endpoints --region ${AWS_REGION} --query "Endpoints[?contains(EndpointName,'quickstart-llm')].EndpointName" --output text` | 空（无输出） |
| 4 | `aws sagemaker list-models --region ${AWS_REGION} --query "Models[?contains(ModelName,'falcon')].ModelName" --output text` | 空（无输出） |

---

## 实验总结

本实验用 JumpStart 一键部署了一个开源大模型端点，体验了 SageMaker 在生成式 AI 时代最常见的落地方式，并与前面自训练 XGBoost 的路径做了对比。JumpStart 让「用现成大模型」的门槛降到一次 deploy 调用，是 SageMaker 承接 GenAI 需求的主入口。

---

## 清理

> ⚠️ **首选：把清理放进部署脚本的 `finally` 块**（`run_demo19_all.py` 已这么做：调用完立即 `predictor.delete_endpoint(delete_endpoint_config=True)` + `model.delete_model()`），无论调用成功与否都删，GPU 计费窗口最短。下面的 CLI 命令是兜底手动清理。

```bash
# ⚠️ GPU 端点单价高，务必第一时间删除
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-llm --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-llm --region ${AWS_REGION} 2>/dev/null || true
# ⚠️ JumpStart 自动生成的 Model 名不含 "sm-quickstart-llm"，而是 hf-llm-falcon-...；按 falcon 匹配
aws sagemaker list-models --region ${AWS_REGION} --query "Models[?contains(ModelName,'falcon')].ModelName" --output text \
  | tr '\t' '\n' | while read m; do [ -n "$m" ] && aws sagemaker delete-model --model-name "$m" --region ${AWS_REGION}; done
# 二次确认：以下两条都应为空
aws sagemaker list-endpoints --region ${AWS_REGION} --query "Endpoints[?contains(EndpointName,'quickstart-llm')].EndpointName" --output text
aws sagemaker list-models --region ${AWS_REGION} --query "Models[?contains(ModelName,'falcon')].ModelName" --output text
echo "清理完成 — 已二次确认端点/模型均已删除，避免 GPU 计费"
```
