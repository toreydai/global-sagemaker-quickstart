# Demo12 — Multi-Model Endpoint 多模型端点

## 实验简介

将多个同类模型托管到同一个端点（Multi-Model Endpoint），通过请求头 `TargetModel` 路由到具体模型，大幅降低多模型场景下的端点成本。

**实验目标：**
- 理解 MME 的模型懒加载/内存卸载机制
- 掌握模型打包与 S3 目录组织约定
- 能验证多模型路由调用是否正确

**实验流程：**
1. 准备 2-3 个模型工件（可复用 Demo03 训练流程多跑几次）
2. 按 MME 约定上传至同一 S3 前缀
3. 部署 Multi-Model Endpoint
4. 分别携带不同 `TargetModel` 调用验证路由

**预计 AI 执行时长：** 12 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker、S3
- **前提**：Demo03 已完成（可复用其训练流程产出多个模型）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

## 步骤

### 1. 准备多个模型工件到同一 S3 前缀

```bash
# 复用 Demo03 的 model.tar.gz，拷成 2 份不同命名模拟多模型
export MME_PREFIX=s3://${SM_BUCKET}/mme-models
aws s3 cp ${MODEL_DATA_URL} ${MME_PREFIX}/model-a.tar.gz --region ${AWS_REGION}
aws s3 cp ${MODEL_DATA_URL} ${MME_PREFIX}/model-b.tar.gz --region ${AWS_REGION}
aws s3 ls ${MME_PREFIX}/ --region ${AWS_REGION}
```

**预期输出**：列出 `model-a.tar.gz`、`model-b.tar.gz`（各约 12 KB）

### 2. 部署 Multi-Model Endpoint

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.multidatamodel import MultiDataModel
from sagemaker.model import Model

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
from sagemaker.predictor import Predictor
container = Model(image_uri=retrieve("xgboost", region, version="1.7-1"), role=role,
                  predictor_cls=Predictor)   # 让 deploy() 返回 predictor（见 Demo04）
mme = MultiDataModel(name="sm-quickstart-mme",
                     model_data_prefix=f"s3://{bucket}/mme-models/",
                     model=container, sagemaker_session=sagemaker.Session())
predictor = mme.deploy(initial_instance_count=1, instance_type="ml.m5.large",
                       endpoint_name="sm-quickstart-mme")
print("ENDPOINT=", predictor.endpoint_name if predictor else "sm-quickstart-mme")
PY
```

**预期输出**：约 5-6 分钟后打印 `ENDPOINT= sm-quickstart-mme`（进度条 `------!`）。

> ⚠️ **实测坑**：`MultiDataModel.deploy()` 的 predictor 返回同样依赖底层 `Model` 是否带 `predictor_cls`；不带则返回 `None`（`AttributeError`）。已在 `container=Model(...)` 上加 `predictor_cls=Predictor`。端点名固定 `sm-quickstart-mme`。

### 3. 携带不同 TargetModel 路由调用

```bash
export ENDPOINT_NAME=sm-quickstart-mme
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
for M in model-a.tar.gz model-b.tar.gz; do
  echo "TargetModel=${M}:"
  aws sagemaker-runtime invoke-endpoint --endpoint-name ${ENDPOINT_NAME} \
    --content-type text/csv --target-model ${M} --cli-binary-format raw-in-base64-out \
    --body fileb:///tmp/payload.csv --region ${AWS_REGION} /tmp/mme.json >/dev/null
  cat /tmp/mme.json; echo
done
```

**预期输出**：两次调用各返回 `[0.00047526723938062787]`（注意 MME 的 XGBoost 返回 **JSON 数组** `[...]`，与单模型端点返回裸浮点数不同）。首次访问某模型含懒加载延迟。invoke 需 `fileb://`+`raw-in-base64-out`（见 Demo04）。

---

## 验收标准

- [ ] MME 端点 `InService`
- [ ] 携带不同 `TargetModel` 均返回合法结果
- [ ] 能说清 MME 懒加载/内存卸载与成本优势

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |
| 2 | `aws s3 ls s3://${SM_BUCKET}/mme-models/ --region ${AWS_REGION} \| wc -l` | `2` |

---

## 实验总结

本实验把多个同类模型托管到单个端点，通过 `TargetModel` 头路由、按需懒加载，将 N 个模型的部署成本从 N 个端点压缩到 1 个。这是 SaaS 多租户、模型按客户细分等场景下的关键成本优化手段。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-mme --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-mme --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-model --model-name sm-quickstart-mme --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/mme-models/ --recursive --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
