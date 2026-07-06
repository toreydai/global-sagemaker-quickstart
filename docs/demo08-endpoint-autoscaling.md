# Demo08 — Endpoint 自动伸缩

## 实验简介

为 Demo04 的实时端点配置 Application Auto Scaling，基于调用量指标（`InvocationsPerInstance`）自动增减实例数量，观察完整的扩缩容生命周期。

**实验目标：**
- 理解 SageMaker 端点与 Application Auto Scaling 的集成方式
- 掌握目标追踪（Target Tracking）伸缩策略配置
- 能观察从压测触发扩容到流量下降后缩容的全过程

**实验流程：**
1. 将端点 Variant 注册为可伸缩目标
2. 配置目标追踪伸缩策略
3. 压测触发扩容并观察实例数变化
4. 停止压测，观察自动缩容（缩容受冷却窗口控制，默认约 5-10 分钟才会真正缩回）

**预计 AI 执行时长：** 15-20 分钟（扩容约 3-5 分钟即可观察到；完整看到缩容需额外等待冷却窗口）

> ⚠️ scale-out 通常几分钟内可见，但 scale-in 有冷却期（`ScaleInCooldown`，默认 300s 起）。演示时可聚焦扩容验证，缩容部分说明"约 5-10 分钟后自动完成"即可，不必阻塞等待。

---

## 前提条件

- **工具**：AWS CLI 2.x
- **权限**：Application Auto Scaling、SageMaker、CloudWatch
- **前提**：Demo04 已完成（端点 `InService`）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ENDPOINT_NAME=<Demo04 创建的端点名>
```

---

## 步骤

### 1. 注册可伸缩目标

```bash
export RES_ID="endpoint/${ENDPOINT_NAME}/variant/AllTraffic"
aws application-autoscaling register-scalable-target \
  --service-namespace sagemaker \
  --resource-id ${RES_ID} \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount \
  --min-capacity 1 --max-capacity 4 \
  --region ${AWS_REGION}
```

**预期输出**：无报错（返回空）

### 2. 配置目标追踪策略

```bash
cat > /tmp/scaling.json <<EOF
{"TargetValue": 5.0,
 "PredefinedMetricSpecification": {"PredefinedMetricType": "SageMakerVariantInvocationsPerInstance"},
 "ScaleInCooldown": 300, "ScaleOutCooldown": 60}
EOF
aws application-autoscaling put-scaling-policy \
  --policy-name ${ENDPOINT_NAME}-tt \
  --service-namespace sagemaker \
  --resource-id ${RES_ID} \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file:///tmp/scaling.json \
  --region ${AWS_REGION}
```

**预期输出**：返回含 `PolicyARN` 与**两个** CloudWatch 告警 ARN（`...-AlarmHigh-...` 与 `...-AlarmLow-...`）的 JSON。

### 3. 压测触发扩容

```bash
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
echo "持续压测 5 分钟..."
end=$((SECONDS+300)); n=0
while [ $SECONDS -lt $end ]; do
  aws sagemaker-runtime invoke-endpoint --endpoint-name ${ENDPOINT_NAME} \
    --content-type text/csv --cli-binary-format raw-in-base64-out \
    --body fileb:///tmp/payload.csv /dev/null >/dev/null 2>&1
  n=$((n+1))
done
echo "共发送 ${n} 次调用"
# 观察扩容（可能需在压测期间/结束后再等 1-2 分钟）
aws sagemaker describe-endpoint --endpoint-name ${ENDPOINT_NAME} --region ${AWS_REGION} \
  --query 'ProductionVariants[0].{Current:CurrentInstanceCount,Desired:DesiredInstanceCount}'
aws cloudwatch describe-alarms \
  --alarm-name-prefix "TargetTracking-endpoint/${ENDPOINT_NAME}/variant/AllTraffic-AlarmHigh" \
  --query 'MetricAlarms[0].StateValue' --output text
```

**预期输出**：实测约 646 次调用（~130 次/分钟，远超 TargetValue=5）。CloudWatch 指标有 2-3 分钟延迟，`AlarmHigh` 先为 `INSUFFICIENT_DATA`，约 5 分钟后转 `ALARM`，`DesiredInstanceCount` **直接从 1 跳到上限 4**（因每实例调用量远超目标值，目标追踪一次性扩到 max）。

> ⚠️ **实测坑（invoke 计入指标）**：压测循环必须能真正调用成功，`--body` 用 `--cli-binary-format raw-in-base64-out` + `fileb://`（见 Demo04），否则 415 报错的调用不会产生正常的 `InvocationsPerInstance` 指标，扩容不触发。
> ⚠️ **成本**：扩到 4 台 ml.m5.large 会 4 倍计费。验证到扩容后应尽快清理伸缩策略并把实例缩回 1（见清理段），不要放着不管。

### 4. 停止压测，说明缩容

停止上面的循环后，实例会在 `ScaleInCooldown`（约 5-10 分钟）后自动缩回 1，无需阻塞等待。

---

## 验收标准

- [ ] 可伸缩目标注册成功（min=1, max=4）
- [ ] 目标追踪策略创建成功并生成 CloudWatch 告警
- [ ] 压测期间实例数自动扩容 >1

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws application-autoscaling describe-scalable-targets --service-namespace sagemaker --resource-id ${RES_ID} --region ${AWS_REGION} --query 'ScalableTargets[0].MaxCapacity' --output text` | `4` |
| 2 | `aws application-autoscaling describe-scaling-policies --service-namespace sagemaker --resource-id ${RES_ID} --region ${AWS_REGION} --query 'ScalingPolicies[0].PolicyType' --output text` | `TargetTrackingScaling` |

---

## 实验总结

本实验为实时端点接入 Application Auto Scaling，用目标追踪策略基于每实例调用量自动扩缩容。理解 scale-out 快、scale-in 受冷却窗口约束这一非对称性，是生产端点容量规划的关键。

---

## 清理

```bash
aws application-autoscaling delete-scaling-policy --policy-name ${ENDPOINT_NAME}-tt \
  --service-namespace sagemaker --resource-id ${RES_ID} \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount --region ${AWS_REGION} 2>/dev/null || true
aws application-autoscaling deregister-scalable-target \
  --service-namespace sagemaker --resource-id ${RES_ID} \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成（端点本身由 Demo04 清理）"
```
