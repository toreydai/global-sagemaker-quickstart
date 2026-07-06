# Demo13 — Model Monitor 数据漂移检测

## 实验简介

为 Demo04 的端点开启 Data Capture，基于训练数据建立基线（baseline）统计与约束，检测线上流量与训练数据的分布漂移。属于进阶专题——概念密度高（两阶段流程 + Processing 作业 + 约束文件），且定时监控有最小调度粒度限制。

**实验目标：**
- 理解 Model Monitor「建基线 → 监控作业」两阶段流程
- 掌握 Data Capture 配置方式
- 能读懂漂移检测报告并定位违规字段

**实验流程：**
1. 为端点开启 Data Capture
2. 用训练数据生成 baseline 统计量与约束文件
3. 制造漂移流量后，**手动触发一次监控作业**（on-demand）以立即拿到结果
4. 查看违规（violations）报告；如改用 Monitoring Schedule，说明其最小粒度为每小时

**预计 AI 执行时长：** 约 20 分钟（走手动触发监控作业路径）

> ⚠️ Monitoring Schedule 的最小调度粒度是**每小时**，20 分钟内看不到定时作业出报告。演示要即时看到 violations，应走「手动触发一次 monitoring job」路径；定时调度仅作说明或留给学员课后验证。

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker（Model Monitor 相关 API）、S3
- **前提**：Demo04 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export ENDPOINT_NAME=<Demo04 创建的端点名>
export MODEL_DATA_URL=<Demo03 输出的模型 S3 路径>
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

> ✅ **已在真实账号验证要点**：
> - **定时调度最小粒度确为每小时**：SDK `CronExpressionGenerator` 只提供 `hourly()`（`cron(0 * ? * * *)`）、`daily()`、`daily_every_x_hours()`，**没有**分钟级选项；无法配置亚小时调度。
> - **on-demand 路径用 `CronExpressionGenerator.now()`**（一次性立即执行），但**并非"秒级即时"**：实测从创建 schedule 到执行 `Completed` 约 **11 分钟**（约 5 分钟排队启动 + 约 5 分钟 Processing 作业）。仍在 20 分钟内，但要有心理预期，不是同步返回。

## 步骤

### 1. 重新部署端点并开启 Data Capture

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.model_monitor import DataCaptureConfig

region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              sagemaker_session=sagemaker.Session())
model.deploy(initial_instance_count=1, instance_type="ml.m5.large",
             endpoint_name="sm-quickstart-mon",
             data_capture_config=DataCaptureConfig(
                 enable_capture=True, sampling_percentage=100,
                 destination_s3_uri=f"s3://{bucket}/datacapture"))
print("deployed")
PY
export ENDPOINT_NAME=sm-quickstart-mon
```

> ⚠️ 本 Demo 需 `SM_ROLE_ARN` 与 `MODEL_DATA_URL`（同 Demo03/04）。若已有开启 Data Capture 的端点可跳过重建。

**预期输出**：`deployed`，端点 `InService`

### 2. 用训练数据生成 baseline

```bash
python3 - <<'PY'
import os
from sagemaker.model_monitor import DefaultModelMonitor
from sagemaker.model_monitor.dataset_format import DatasetFormat
bucket=os.environ["SM_BUCKET"]; role=os.environ["SM_ROLE_ARN"]
mon = DefaultModelMonitor(role=role, instance_count=1, instance_type="ml.m5.xlarge")
mon.suggest_baseline(
    baseline_dataset=f"s3://{bucket}/train/train.csv",
    dataset_format=DatasetFormat.csv(header=False),
    output_s3_uri=f"s3://{bucket}/monitor/baseline",
)
print("baseline done")
PY
```

**预期输出**：生成 `statistics.json` 与 `constraints.json` 到 `s3://.../monitor/baseline`

### 3. 制造流量并用 `now()` 触发一次性监控执行（on-demand）

```bash
# 先打一批推理产生 capture 数据（invoke 需 fileb+raw-in-base64-out，见 Demo04）
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
for i in $(seq 1 30); do
  aws sagemaker-runtime invoke-endpoint --endpoint-name ${ENDPOINT_NAME} \
    --content-type text/csv --cli-binary-format raw-in-base64-out \
    --body fileb:///tmp/payload.csv --region ${AWS_REGION} /dev/null >/dev/null 2>&1
done
# capture 数据有几分钟落盘延迟
sleep 60
aws s3 ls s3://${SM_BUCKET}/datacapture/ --recursive --region ${AWS_REGION} | tail -3
```

**预期输出**：`datacapture/sm-quickstart-mon/AllTraffic/<Y>/<M>/<D>/<H>/*.jsonl` 捕获文件（每条含 `captureData.endpointInput/endpointOutput`）。

```bash
# 用 now() 创建一次性监控执行（on-demand），基于 baseline 检查 capture 数据
python3 - <<'PY'
import os
from sagemaker.model_monitor import DefaultModelMonitor, CronExpressionGenerator
bucket=os.environ["SM_BUCKET"]; role=os.environ["SM_ROLE_ARN"]
mon=DefaultModelMonitor(role=role, instance_count=1, instance_type="ml.m5.xlarge")
mon.create_monitoring_schedule(
    monitor_schedule_name="sm-quickstart-mon-now",
    endpoint_input="sm-quickstart-mon",
    statistics=f"s3://{bucket}/monitor/baseline/statistics.json",
    constraints=f"s3://{bucket}/monitor/baseline/constraints.json",
    schedule_cron_expression=CronExpressionGenerator.now(),
    output_s3_uri=f"s3://{bucket}/monitor/reports",
    data_analysis_start_time="-PT2H", data_analysis_end_time="-PT0H",
)
print("now() 一次性监控已创建")
PY
# 轮询执行状态（约 11 分钟：排队启动 + Processing 作业）
for i in $(seq 1 30); do
  ST=$(aws sagemaker list-monitoring-executions --monitoring-schedule-name sm-quickstart-mon-now \
       --region ${AWS_REGION} --query 'MonitoringExecutionSummaries[0].MonitoringExecutionStatus' --output text)
  echo "exec: $ST"
  case "$ST" in Completed|CompletedWithViolations|Failed) break;; esac; sleep 30
done
```

**预期输出**：约 11 分钟后 `CompletedWithViolations`（前 ~5 分钟 `list` 返回 `None`/排队，然后 `InProgress`）。

### 4. 查看违规报告

```bash
aws s3 ls s3://${SM_BUCKET}/monitor/reports/ --recursive --region ${AWS_REGION} | grep violations
aws s3 cp "$(aws s3 ls s3://${SM_BUCKET}/monitor/reports/ --recursive --region ${AWS_REGION} | grep constraint_violations | awk '{print "s3://'${SM_BUCKET}'/"$NF}')" - --region ${AWS_REGION}
```

**预期输出**：生成 `constraint_violations.json`。本例因 capture 的是**去掉 label 的特征负载**（首列 `_c0` 为浮点特征），而 baseline 建于 `train.csv`（首列 label 为整数），故触发一条真实的 `data_type_check` 违规：
```json
{"violations":[{"feature_name":"_c0","constraint_check_type":"data_type_check",
 "description":"Data type match requirement is not met. Expected data type: Integral, ... Observed: Only 0.0% of data is Integral."}]}
```

---

## 验收标准

- [ ] 端点已开启 Data Capture 且 capture 数据落 S3
- [ ] baseline 的 `statistics.json` / `constraints.json` 生成成功
- [ ] 能通过 on-demand 路径（而非等每小时调度）拿到监控结果
- [ ] 能读懂 violations 报告定位漂移字段

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws s3 ls s3://${SM_BUCKET}/monitor/baseline/constraints.json --region ${AWS_REGION} \| wc -l` | `1` |
| 2 | `aws s3 ls s3://${SM_BUCKET}/datacapture/ --recursive --region ${AWS_REGION} \| wc -l` | `>= 1` |

---

## 实验总结

本实验走通了 Model Monitor 的核心两阶段：用训练数据建 baseline（统计量 + 约束），再对端点 capture 的线上流量做漂移检测。关键认知是——定时监控最小粒度为每小时，演示与快速验证应走 on-demand 路径。数据漂移检测是模型上线后质量保障的第一道防线。

---

## 清理

```bash
aws sagemaker delete-monitoring-schedule --monitoring-schedule-name sm-quickstart-mon-sched --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-mon --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-mon --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/datacapture/ --recursive --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/monitor/ --recursive --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
