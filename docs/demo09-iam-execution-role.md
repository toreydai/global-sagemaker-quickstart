# Demo09 — IAM 执行角色与权限最小化

## 实验简介

将 Demo01 起步时使用的 `AmazonSageMakerFullAccess` 收敛为按最小权限原则编写的自定义策略，覆盖训练、部署、S3 访问的最小操作集合。

**实验目标：**
- 理解 SageMaker 执行角色的最小权限边界
- 掌握自定义 IAM 策略的编写与验证方法
- 能验证权限收敛后前序 Demo 功能不受影响

**实验流程：**
1. 梳理 Demo01-08 实际用到的 API 操作（数据处理/训练/部署/伸缩）
2. 编写最小权限自定义策略
3. 替换执行角色上的策略
4. 重跑关键 Demo（训练/部署/监控）验证功能正常

**预计 AI 执行时长：** 12 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x
- **权限**：IAM（PutRolePolicy/AttachRolePolicy）
- **前提**：Demo01-08 已完成（需覆盖数据处理/训练/部署/伸缩用到的全部 API 才能准确收敛权限）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_NAME=SageMakerQuickStartExecutionRole
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

> ⚠️ 本 Demo 收敛的是**基础阶段**用到的权限。进阶 Demo（Model Monitor/Pipelines/Feature Store/JumpStart 等）会用到额外 API，届时需按需扩展本策略，而非直接套用。

---

> ✅ **已在真实账号验证**：应用本最小策略并解绑 `AmazonSageMakerFullAccess` 后，重跑了一次内置 XGBoost 训练（Training Job `Completed`），说明 ECR 拉镜像、S3 读写、CloudWatch 日志、PassRole 均被本策略覆盖。验证后按「清理」段回滚到 FullAccess，以便进阶 Demo（Feature Store/Pipelines/Model Registry 等）继续使用。

## 步骤

### 1. 编写最小权限自定义策略

```bash
cat > /tmp/sm-minimal.json <<EOF
{"Version":"2012-10-17","Statement":[
  {"Sid":"SageMakerCore","Effect":"Allow","Action":[
    "sagemaker:CreateProcessingJob","sagemaker:CreateTrainingJob",
    "sagemaker:CreateModel","sagemaker:CreateEndpointConfig","sagemaker:CreateEndpoint",
    "sagemaker:Describe*","sagemaker:List*","sagemaker:InvokeEndpoint",
    "sagemaker:UpdateEndpoint","sagemaker:DeleteEndpoint","sagemaker:DeleteEndpointConfig","sagemaker:DeleteModel"
  ],"Resource":"*"},
  {"Sid":"S3Data","Effect":"Allow","Action":["s3:GetObject","s3:PutObject","s3:ListBucket","s3:DeleteObject"],
   "Resource":["arn:aws:s3:::${SM_BUCKET}","arn:aws:s3:::${SM_BUCKET}/*"]},
  {"Sid":"Logs","Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents","cloudwatch:PutMetricData"],"Resource":"*"},
  {"Sid":"ECRPull","Effect":"Allow","Action":["ecr:GetAuthorizationToken","ecr:BatchGetImage","ecr:GetDownloadUrlForLayer"],"Resource":"*"},
  {"Sid":"PassRole","Effect":"Allow","Action":"iam:PassRole","Resource":"arn:aws:iam::${ACCOUNT_ID}:role/${SM_ROLE_NAME}",
   "Condition":{"StringEquals":{"iam:PassedToService":"sagemaker.amazonaws.com"}}}
]}
EOF
```

### 2. 附加自定义策略并解除 FullAccess

```bash
aws iam put-role-policy --role-name ${SM_ROLE_NAME} \
  --policy-name SageMakerQuickStartMinimal \
  --policy-document file:///tmp/sm-minimal.json
aws iam detach-role-policy --role-name ${SM_ROLE_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
```

**预期输出**：两条命令均无报错

### 3. 回归验证（重跑一次训练或推理）

```bash
# 用最小权限的执行角色重跑一次短训练，确认功能正常（真正验证角色权限，而非调用方权限）
python3 - <<'PY'
import os, sagemaker
from sagemaker.estimator import Estimator
from sagemaker.inputs import TrainingInput
from sagemaker.image_uris import retrieve
region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
xgb=Estimator(image_uri=retrieve("xgboost",region,version="1.7-1"), role=role,
    instance_count=1, instance_type="ml.m5.xlarge", output_path=f"s3://{bucket}/model-min/",
    sagemaker_session=sagemaker.Session())
xgb.set_hyperparameters(objective="binary:logistic", num_round=10, max_depth=3)
xgb.fit({"train":TrainingInput(f"s3://{bucket}/train/", content_type="csv"),
         "validation":TrainingInput(f"s3://{bucket}/validation/", content_type="csv")}, logs=False)
print("STATUS=", xgb.latest_training_job.describe()["TrainingJobStatus"])
PY
```

**预期输出**：`STATUS= Completed`（最小权限下训练一次通过，约 2-3 分钟，无 `AccessDenied`）。

> 💡 注意：`invoke-endpoint` 用的是**调用方**（你的 CLI 身份）权限，测不出执行角色的收敛效果；真正验证要靠让 SageMaker **用该角色**去跑任务（训练/处理/部署），如上重跑训练。

> ⚠️ 若出现 `AccessDenied`，从报错的 Action 反推补齐策略——这正是最小权限收敛的迭代过程，不要直接退回 FullAccess。

---

## 验收标准

- [ ] 角色已附加自定义最小策略，且已解绑 `AmazonSageMakerFullAccess`
- [ ] 训练/部署/推理关键路径在最小权限下仍可运行
- [ ] 能说清哪些 Action 是训练/部署所必需

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws iam list-attached-role-policies --role-name ${SM_ROLE_NAME} --query 'AttachedPolicies[].PolicyName' --output text` | 不再包含 `AmazonSageMakerFullAccess` |
| 2 | `aws iam get-role-policy --role-name ${SM_ROLE_NAME} --policy-name SageMakerQuickStartMinimal --query 'PolicyName' --output text` | `SageMakerQuickStartMinimal` |

---

## 实验总结

本实验把起步用的 `AmazonSageMakerFullAccess` 收敛为一份按实际调用范围编写的最小权限策略，并通过回归验证确认功能不受影响。最小权限是生产落地的合规底线；进阶 Demo 引入新能力时，应以同样方式增量扩展而非退回全权限。

---

## 清理

```bash
# 如需回滚到起步权限（便于继续进阶 Demo）：
aws iam attach-role-policy --role-name ${SM_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam delete-role-policy --role-name ${SM_ROLE_NAME} --policy-name SageMakerQuickStartMinimal 2>/dev/null || true
echo "已回滚到起步权限"
```
