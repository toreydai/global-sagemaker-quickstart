# Demo10 — CloudWatch 全链路可观测性

## 实验简介

汇总训练任务与推理端点的 CloudWatch 指标（如 `CPUUtilization`、`ModelLatency`、`Invocations`）与日志，搭建一个最小可用的监控视图，作为「基础」阶段的收尾。

**实验目标：**
- 掌握 SageMaker 相关 CloudWatch 指标命名空间与关键指标含义
- 能搭建基础 Dashboard 覆盖训练与推理两类资源
- 理解告警阈值设置的基本思路

**实验流程：**
1. 梳理训练/端点关键指标
2. 创建 CloudWatch Dashboard
3. 为 `ModelLatency` 配置告警
4. 制造延迟升高验证告警触发

**预计 AI 执行时长：** 10 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x
- **权限**：CloudWatch（Dashboard/Alarm）
- **前提**：Demo04、Demo08 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ENDPOINT_NAME=<Demo04 创建的端点名>
```

---

## 步骤

### 1. 关键指标速览

| 指标 | 命名空间 | 维度 | 含义 |
|------|---------|------|------|
| `Invocations` | `AWS/SageMaker` | EndpointName, VariantName | 调用次数 |
| `ModelLatency` | `AWS/SageMaker` | EndpointName, VariantName | 模型推理延迟（微秒） |
| `CPUUtilization` | `/aws/sagemaker/Endpoints` | EndpointName, VariantName | 实例 CPU 使用率 |

### 2. 创建 Dashboard

```bash
cat > /tmp/dash.json <<EOF
{"widgets":[
 {"type":"metric","x":0,"y":0,"width":12,"height":6,"properties":{
   "title":"Invocations & Latency","region":"${AWS_REGION}","stat":"Average",
   "metrics":[
     ["AWS/SageMaker","Invocations","EndpointName","${ENDPOINT_NAME}","VariantName","AllTraffic"],
     ["AWS/SageMaker","ModelLatency","EndpointName","${ENDPOINT_NAME}","VariantName","AllTraffic"]]}}
]}
EOF
aws cloudwatch put-dashboard --dashboard-name sm-quickstart \
  --dashboard-body file:///tmp/dash.json --region ${AWS_REGION}
```

**预期输出**：`{"DashboardValidationMessages": []}`

### 3. 为 ModelLatency 配置告警

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name sm-quickstart-latency \
  --namespace AWS/SageMaker --metric-name ModelLatency \
  --dimensions Name=EndpointName,Value=${ENDPOINT_NAME} Name=VariantName,Value=AllTraffic \
  --statistic Average --period 60 --evaluation-periods 1 \
  --threshold 1000000 --comparison-operator GreaterThanThreshold \
  --region ${AWS_REGION}
```

**预期输出**：无报错（阈值 1,000,000 微秒 = 1s，可按实际基线调整）

### 4. 产生流量并观察

```bash
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} 2>/dev/null \
  | head -1 | cut -d, -f2- > /tmp/payload.csv
for i in $(seq 1 20); do
  aws sagemaker-runtime invoke-endpoint --endpoint-name ${ENDPOINT_NAME} \
    --content-type text/csv --cli-binary-format raw-in-base64-out \
    --body fileb:///tmp/payload.csv --region ${AWS_REGION} /dev/null >/dev/null 2>&1
done
# 指标有 2-3 分钟延迟，告警会先 INSUFFICIENT_DATA 再转 OK
for i in $(seq 1 8); do
  ST=$(aws cloudwatch describe-alarms --alarm-names sm-quickstart-latency --region ${AWS_REGION} --query 'MetricAlarms[0].StateValue' --output text)
  echo "alarm=$ST"; { [ "$ST" = "OK" ] || [ "$ST" = "ALARM" ]; } && break; sleep 30
done
```

**预期输出**：约 3-4 分钟后 `OK`（实测 `ModelLatency` 约 **5932 微秒 ≈ 5.9ms**，远低于 1s 阈值，故不触发）。要演示 `ALARM` 可把 `--threshold` 调到远低于基线（如 `1000`=1ms）。invoke 同样需 `fileb://`+`raw-in-base64-out`（见 Demo04）。

---

## 验收标准

- [ ] Dashboard 创建成功且展示 Invocations/ModelLatency
- [ ] ModelLatency 告警创建成功
- [ ] 能说清三个关键指标的命名空间与含义

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws cloudwatch get-dashboard --dashboard-name sm-quickstart --region ${AWS_REGION} --query 'DashboardName' --output text` | `sm-quickstart` |
| 2 | `aws cloudwatch describe-alarms --alarm-names sm-quickstart-latency --region ${AWS_REGION} --query 'MetricAlarms[0].MetricName' --output text` | `ModelLatency` |

---

## 实验总结

本实验汇总了训练与推理两类资源的核心 CloudWatch 指标，搭建了 Dashboard 与延迟告警，完成「基础」阶段的可观测性收尾。至此，从建环境、数据、训练、部署、弹性到监控与权限的最小生产闭环已完整；进阶阶段将在此之上叠加高级推理模式与 MLOps 能力。

---

## 清理

```bash
aws cloudwatch delete-dashboards --dashboard-names sm-quickstart --region ${AWS_REGION} 2>/dev/null || true
aws cloudwatch delete-alarms --alarm-names sm-quickstart-latency --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
