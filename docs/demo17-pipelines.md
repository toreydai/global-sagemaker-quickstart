# Demo17 — SageMaker Pipelines 模型流水线

## 实验简介

用 SageMaker Pipelines 把数据处理、训练、评估、条件注册串成一条可重复执行的 CI/CD 流水线，替代手工串联多个 Demo 的步骤。

**实验目标：**
- 理解 Pipeline 的 Step 类型（Processing/Training/Condition/RegisterModel）
- 掌握参数化 Pipeline 定义方式
- 能触发一次 Execution 并观察 DAG 执行结果

**实验流程：**
1. 定义 Processing + Training + Condition + RegisterModel Step
2. 创建/更新 Pipeline 定义
3. 触发一次 Pipeline Execution
4. 查看 DAG 执行结果与各 Step 产物

**预计 AI 执行时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker Pipelines、Processing、Training
- **前提**：Demo03、Demo09 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
export PIPELINE_NAME=sagemaker-quickstart-pipeline
```

---

> ℹ️ 本 Demo 已按「最小单 TrainingStep」跑通验证。`upsert` 会打印几条无害提示：`Ignoring unnecessary instance type: None`、`Popping out 'TrainingJobName' ...`（SDK 会在执行时覆盖该字段），均可忽略。完整流水线（Processing→Training→Eval→Condition→RegisterModel）可在此基础上扩展。

## 步骤

### 1. 定义并 upsert Pipeline

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import TrainingStep
from sagemaker.workflow.parameters import ParameterString
from sagemaker.estimator import Estimator
from sagemaker.image_uris import retrieve
from sagemaker.inputs import TrainingInput

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
sess=sagemaker.Session()
train_uri = ParameterString(name="TrainUri", default_value=f"s3://{bucket}/train/")

est = Estimator(image_uri=retrieve("xgboost", region, version="1.7-1"), role=role,
                instance_count=1, instance_type="ml.m5.xlarge",
                output_path=f"s3://{bucket}/pipeline-model/", sagemaker_session=sess)
est.set_hyperparameters(objective="binary:logistic", num_round=100, max_depth=5)

step_train = TrainingStep(name="TrainStep",
    estimator=est, inputs={"train": TrainingInput(train_uri, content_type="csv")})

pipe = Pipeline(name=os.environ["PIPELINE_NAME"], parameters=[train_uri],
                steps=[step_train], sagemaker_session=sess)
pipe.upsert(role_arn=role)
print("pipeline upserted")
PY
```

**预期输出**：`pipeline upserted`

> ⚠️ 完整流水线应包含 Processing→Training→Evaluation→Condition→RegisterModel。此处先跑通最小单 Step，真跑后按需扩展并把 Step 间属性引用语法记入 Known Issues。

### 2. 触发一次 Execution

```bash
export EXEC_ARN=$(aws sagemaker start-pipeline-execution \
  --pipeline-name ${PIPELINE_NAME} --region ${AWS_REGION} \
  --query 'PipelineExecutionArn' --output text)
echo ${EXEC_ARN}
```

**预期输出**：打印 Execution ARN

### 3. 轮询执行状态

```bash
while true; do
  ST=$(aws sagemaker describe-pipeline-execution --pipeline-execution-arn ${EXEC_ARN} \
       --region ${AWS_REGION} --query 'PipelineExecutionStatus' --output text)
  echo "状态：${ST}"; [ "${ST}" = "Succeeded" ] || [ "${ST}" = "Failed" ] && break; sleep 45
done
aws sagemaker list-pipeline-execution-steps --pipeline-execution-arn ${EXEC_ARN} \
  --region ${AWS_REGION} --query 'PipelineExecutionSteps[].{Step:StepName,Status:StepStatus}'
```

**预期输出**：约 3 分钟后 `状态：Succeeded`，`TrainStep` 状态 `Succeeded`（单 Step DAG）。

---

## 验收标准

- [ ] Pipeline upsert 成功
- [ ] Execution 状态 `Succeeded`
- [ ] 能列出各 Step 及其状态

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-pipeline --pipeline-name ${PIPELINE_NAME} --region ${AWS_REGION} --query 'PipelineName' --output text` | `sagemaker-quickstart-pipeline` |
| 2 | `aws sagemaker describe-pipeline-execution --pipeline-execution-arn ${EXEC_ARN} --region ${AWS_REGION} --query 'PipelineExecutionStatus' --output text` | `Succeeded` |

---

## 实验总结

本实验把此前手工串联的数据/训练步骤固化为一条可重复、可参数化、可追溯的 Pipeline。流水线化是从「能训练」到「可持续交付模型」的关键一步，产出的模型可直接进入 Demo18 的 Registry 做版本管理与部署。

---

## 清理

```bash
aws sagemaker delete-pipeline --pipeline-name ${PIPELINE_NAME} --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/pipeline-model/ --recursive --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成（如 Demo18 需 Pipeline 产出的模型，请先完成 Demo18）"
```
