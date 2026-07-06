# Demo15 — 超参数调优 Tuning Job

## 实验简介

使用 Hyperparameter Tuning Job 对 Demo03 的 XGBoost 训练做自动超参数搜索，对比最优模型与基线模型的效果差异。

**实验目标：**
- 理解贝叶斯搜索/随机搜索策略的选择依据
- 掌握超参数范围与目标指标（objective metric）的配置方式
- 能从多个 Training Job 中选出最优模型并说明依据

**实验流程：**
1. 定义超参数搜索空间与目标指标
2. 提交 Tuning Job（内部并行/串行多个 Training Job）
3. 轮询至 `Completed`
4. 对比最优模型与 Demo03 基线的指标差异

**预计 AI 执行时长：** 20-25 分钟（含并行训练等待）

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（CreateHyperParameterTuningJob）
- **前提**：Demo03 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

> ✅ **已验证**：内置 XGBoost 的 HPO 目标指标名 `validation:auc` 有效，但**必须给 Estimator 设 `eval_metric="auc"`**，否则 binary:logistic 默认不发出 AUC 指标、HPO 拿不到目标值。已在下方代码补上。

## 步骤

### 1. 定义搜索空间并提交 Tuning Job

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.estimator import Estimator
from sagemaker.image_uris import retrieve
from sagemaker.inputs import TrainingInput
from sagemaker.tuner import HyperparameterTuner, IntegerParameter, ContinuousParameter

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
est = Estimator(image_uri=retrieve("xgboost", region, version="1.7-1"), role=role,
                instance_count=1, instance_type="ml.m5.xlarge",
                output_path=f"s3://{bucket}/hpo-model/", sagemaker_session=sagemaker.Session())
est.set_hyperparameters(objective="binary:logistic", num_round=100,
                        eval_metric="auc")   # ⚠️ 必须显式发出 auc，HPO 才能取到 validation:auc

tuner = HyperparameterTuner(
    estimator=est,
    objective_metric_name="validation:auc",   # ✅ 已验证有效（配合上面的 eval_metric=auc）
    objective_type="Maximize",
    hyperparameter_ranges={"max_depth": IntegerParameter(3, 8),
                           "eta": ContinuousParameter(0.05, 0.4),
                           "subsample": ContinuousParameter(0.6, 1.0)},
    max_jobs=6, max_parallel_jobs=3, strategy="Bayesian",
)
tuner.fit({"train": TrainingInput(f"s3://{bucket}/train/", content_type="csv"),
           "validation": TrainingInput(f"s3://{bucket}/validation/", content_type="csv")}, wait=False)
print("TUNING_JOB=", tuner.latest_tuning_job.name)
PY
```

**预期输出**：打印 `TUNING_JOB= sagemaker-xgboost-<时间戳>`，内部并行跑 6 个训练任务（`wait=False` 立即返回；两处 `No finished training job found ...` 是无害提示）。

### 2. 轮询至完成

```bash
export TUNING_JOB=<上一步打印的 TUNING_JOB>
while true; do
  ST=$(aws sagemaker describe-hyper-parameter-tuning-job --hyper-parameter-tuning-job-name ${TUNING_JOB} \
       --region ${AWS_REGION} --query 'HyperParameterTuningJobStatus' --output text)
  echo "状态：${ST}"; [ "${ST}" = "Completed" ] && break; sleep 60
done
```

**预期输出**：最终 `状态：Completed`（数据集小，实测约 **4 分钟**即 6 个训练全部完成，远快于草稿估计的 15-20 分钟）

### 3. 查看最优任务

```bash
aws sagemaker describe-hyper-parameter-tuning-job --hyper-parameter-tuning-job-name ${TUNING_JOB} \
  --region ${AWS_REGION} \
  --query 'BestTrainingJob.{Job:TrainingJobName,Metric:FinalHyperParameterTuningJobObjectiveMetric.Value,HP:TunedHyperParameters}'
```

**预期输出**：返回最优任务名、`MetricName=validation:auc`、最优值与超参。实测：最优 `validation:auc ≈ 0.9927`，对应 `max_depth=6, eta≈0.119, subsample≈0.868`（优于 Demo03 基线的手工超参）。

---

## 验收标准

- [ ] Tuning Job 状态 `Completed`，内部子任务全部结束
- [ ] 能读出 `BestTrainingJob` 及其目标指标值
- [ ] 最优模型指标不劣于 Demo03 基线

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-hyper-parameter-tuning-job --hyper-parameter-tuning-job-name ${TUNING_JOB} --region ${AWS_REGION} --query 'HyperParameterTuningJobStatus' --output text` | `Completed` |
| 2 | `aws sagemaker describe-hyper-parameter-tuning-job --hyper-parameter-tuning-job-name ${TUNING_JOB} --region ${AWS_REGION} --query 'TrainingJobStatusCounters.Completed' --output text` | `6` |

---

## 实验总结

本实验用贝叶斯策略在超参数空间自动搜索，从 6 个训练任务中选出最优模型，并与 Demo03 手工设定的基线对比。自动调优把「拍脑袋调参」变成「有目标指标驱动的系统搜索」，是提升模型效果的标准 MLOps 手段。

---

## 清理

```bash
aws s3 rm s3://${SM_BUCKET}/hpo-model/ --recursive --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成（Tuning Job 元数据保留，不计费）"
```
