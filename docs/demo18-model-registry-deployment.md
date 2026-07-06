# Demo18 — Model Registry 与蓝绿/影子部署

## 实验简介

使用 Model Package Group 管理模型版本（Model Registry），并演示如何在同一端点上做蓝绿部署或影子测试，安全地上线新模型版本。

**实验目标：**
- 理解 Model Registry 的版本管理与审批状态机（`PendingManualApproval` 等）
- 掌握端点级别的蓝绿/影子流量切分配置（Production Variants）
- 能完成一次带回滚预案的安全上线演练

**实验流程：**
1. 创建 Model Package Group 并注册模型版本
2. 审批模型版本（`Approved`）
3. 用注册模型部署端点，配置新 Variant 做蓝绿/影子
4. 回滚演练

**预计 AI 执行时长：** 15-18 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker Model Registry、Endpoint 相关 API
- **前提**：Demo17 已完成（或用 Demo03 模型工件）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export MODEL_DATA_URL=<Demo17/Demo03 输出的模型 S3 路径>
export MODEL_PACKAGE_GROUP=sagemaker-quickstart-models
```

---

> ℹ️ 已验证：注册版本从 `PendingManualApproval` → `update-model-package ... Approved`，再用 `ModelPackage.deploy()` 从 Registry 直接部署端点并成功推理。蓝绿/影子（`ShadowProductionVariants`）为生产上线路径示意，最小演练聚焦「注册→审批→部署→列历史 config 备回滚」。

## 步骤

### 1. 创建 Model Package Group 并注册版本

```bash
aws sagemaker create-model-package-group \
  --model-package-group-name ${MODEL_PACKAGE_GROUP} --region ${AWS_REGION} 2>/dev/null || true

python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              sagemaker_session=sagemaker.Session())
mp = model.register(
    content_types=["text/csv"], response_types=["text/csv"],
    inference_instances=["ml.m5.large"], transform_instances=["ml.m5.xlarge"],
    model_package_group_name=os.environ["MODEL_PACKAGE_GROUP"],
    approval_status="PendingManualApproval",
)
print("MODEL_PACKAGE_ARN=", mp.model_package_arn)
PY
```

**预期输出**：打印 `MODEL_PACKAGE_ARN= arn:aws:sagemaker:us-east-1:<acct>:model-package/sagemaker-quickstart-models/1`（首个版本号 `/1`），状态 `PendingManualApproval`。

### 2. 审批模型版本

```bash
export MP_ARN=<上一步的 MODEL_PACKAGE_ARN>
aws sagemaker update-model-package --model-package-arn ${MP_ARN} \
  --model-approval-status Approved --region ${AWS_REGION}
```

**预期输出**：无报错

### 3. 用注册模型部署（含影子 Variant 示意）

```bash
python3 - <<'PY'
import os, boto3
from sagemaker import ModelPackage, Session
sess=Session()
mp = ModelPackage(role=os.environ["SM_ROLE_ARN"], model_package_arn=os.environ["MP_ARN"],
                  sagemaker_session=sess)
mp.deploy(initial_instance_count=1, instance_type="ml.m5.large",
          endpoint_name="sm-quickstart-registry")
print("deployed")
PY
```

**预期输出**：约 6-7 分钟后 `deployed`，端点 `sm-quickstart-registry` `InService`；用 Demo04 的 invoke 方式（`fileb://`+`raw-in-base64-out`）测试返回约 `0.00047`。

> ⚠️ 蓝绿/影子在生产中通过一次 `update-endpoint`（新 EndpointConfig 含新 Variant 或 `ShadowProductionVariants`）完成；真跑时补全该配置并记录参数细节。

### 4. 回滚演练

```bash
# 回滚 = 用旧 EndpointConfig 再 update-endpoint 一次；此处确认可查历史 config
aws sagemaker list-endpoint-configs --region ${AWS_REGION} \
  --query "EndpointConfigs[?contains(EndpointConfigName,'sm-quickstart')].EndpointConfigName"
```

**预期输出**：列出可用于回滚的历史 EndpointConfig

---

## 验收标准

- [ ] Model Package Group 创建成功，含至少一个 `Approved` 版本
- [ ] 用注册模型成功部署端点
- [ ] 能说清蓝绿/影子上线与回滚的操作路径

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker list-model-packages --model-package-group-name ${MODEL_PACKAGE_GROUP} --region ${AWS_REGION} --query 'ModelPackageSummaryList[0].ModelApprovalStatus' --output text` | `Approved` |
| 2 | `aws sagemaker describe-endpoint --endpoint-name sm-quickstart-registry --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService` |

---

## 实验总结

本实验用 Model Registry 管理模型版本与审批状态，并演练了基于 Production Variants 的蓝绿/影子上线与回滚路径。版本可追溯 + 安全上线 + 可回滚，是模型进入生产的最后一道治理闸门。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-registry --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-registry --region ${AWS_REGION} 2>/dev/null || true
# 删除模型版本后再删 group
for arn in $(aws sagemaker list-model-packages --model-package-group-name ${MODEL_PACKAGE_GROUP} --region ${AWS_REGION} --query 'ModelPackageSummaryList[].ModelPackageArn' --output text); do
  aws sagemaker delete-model-package --model-package-name $arn --region ${AWS_REGION} 2>/dev/null || true
done
aws sagemaker delete-model-package-group --model-package-group-name ${MODEL_PACKAGE_GROUP} --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
