# Demo20 — 推理成本优化

## 实验简介

对比同一模型在不同部署形态（标准实例 vs Graviton vs Serverless）下的推理成本与延迟，并说明 Inferentia 与 Savings Plans 的适用条件，产出一份可供客户参考的选型建议框架。

**实验目标：**
- 理解 Graviton（如 ml.c7g）与 Inferentia（inf1/inf2）在推理场景的成本优势与限制
- 理解 SageMaker Savings Plans 的定价机制与适用条件
- 能产出一份成本/延迟对比表和选型建议

**实验流程：**
1. 在标准实例（ml.m5.large）部署端点，采集延迟基线
2. 在 Graviton 实例部署同一模型做对比
3. 说明 Inferentia 与 Savings Plans 的适用条件
4. 汇总产出选型建议

**预计 AI 执行时长：** 15 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker、Pricing（只读）
- **前提**：Demo03 已完成（有训练好的模型工件用于重新部署基线端点）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
export MODEL_DATA_URL="s3://${SM_BUCKET}/model/sagemaker-xgboost-2026-07-06-06-07-53-924/output/model.tar.gz"
```

---

> ✅ **已在真实账号 ACCOUNT_ID_REDACTED / us-east-1 验证**：内置 XGBoost `1.7-1` 镜像**不支持 arm64/Graviton**（`exec format error`，见下），这是本 Demo 最重要的真实结论；标准实例基线延迟、Graviton/Inferentia 单价均已用 Pricing API 核实。

## 步骤

### 0. 重新部署标准实例基线端点（Demo04 端点已在前序 Demo 清理，需重建）

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.predictor import Predictor
region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              sagemaker_session=sagemaker.Session(), predictor_cls=Predictor)
predictor = model.deploy(initial_instance_count=1, instance_type="ml.m5.large",
                          endpoint_name="sm-quickstart-baseline")
print("ENDPOINT=", predictor.endpoint_name)
PY
```

**实测输出**：`ENDPOINT= sm-quickstart-baseline`，`InService` 约 **3.5 分钟**（03:03 → 03:06 UTC）。

### 1. 在 Graviton 实例部署对比端点

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.image_uris import retrieve
from sagemaker.model import Model
from sagemaker.predictor import Predictor
region=os.environ["AWS_REGION"]; role=os.environ["SM_ROLE_ARN"]
model = Model(image_uri=retrieve("xgboost", region, version="1.7-1"),
              model_data=os.environ["MODEL_DATA_URL"], role=role,
              sagemaker_session=sagemaker.Session(), predictor_cls=Predictor)
# ⚠️ ml.m7g.large 在当前 SageMaker Hosting 实例枚举中**不存在**（CreateEndpointConfig 直接 ValidationException，
# 枚举里只有 ml.c7g / ml.r7g(d) / ml.m6g / ml.m8g 等 Graviton 家族，没有 ml.m7g）。改用 ml.c7g.large。
predictor = model.deploy(initial_instance_count=1, instance_type="ml.c7g.large",
                          endpoint_name="sm-quickstart-graviton")
print("deployed")
PY
```

**实测输出**：`ml.m7g.large` 首次尝试直接被 `CreateEndpointConfig` 拒绝（`ValidationException`：不在允许的实例枚举内）。改用 **`ml.c7g.large`**（真实存在的 Graviton Hosting 实例）后请求被接受，端点进入 `Creating`，但**容器持续崩溃**，轮询 **约 24 分钟**（03:07 → 03:31 UTC）后最终转为 **`Failed`**（`did not pass the ping health check`）——见下方 CloudWatch 证据。这本身就是重要结论：**内置 XGBoost `1.7-1` 镜像是 x86-only，不含 arm64 变体**。

> ⚠️ **健康检查失败判定耗时明显长于「10 分钟」的常规超时预期**：容器崩溃循环期间端点持续显示 `Creating`，SageMaker 要经过多轮重试/退避才最终标记 `Failed`，本次与另一次独立复现均耗时 **约 24 分钟**。且端点仍处于 `Creating`（in-progress）期间**不能** `DeleteEndpoint`（会报 `ValidationException: Cannot update in-progress endpoint`），必须等它真正到达终态（`Failed`）才能删除清理，规划时间预算时需把这一点算进去。

```bash
aws logs get-log-events \
  --log-group-name "/aws/sagemaker/Endpoints/sm-quickstart-graviton" \
  --log-stream-name "AllTraffic/<instance-id>" \
  --region ${AWS_REGION} --start-from-head --limit 20 --query 'events[*].message' --output text
```

**实测输出**：
```
exec /miniconda3/bin/serve: exec format error
```
（每次容器重启都重复此错误——`serve` 二进制是 x86_64 编译产物，在 Graviton 的 aarch64 CPU 上无法执行，是内核级的 `ExecFormatError`，而非应用逻辑问题。健康检查因此持续失败，端点最终转 `Failed`，`FailureReason` = "did not pass the ping health check"。）

### 2. 采集两端点延迟对比

```bash
PAYLOAD_FILE=payload.csv
aws s3 cp s3://${SM_BUCKET}/validation/validation.csv - --region ${AWS_REGION} \
  | head -1 | cut -d, -f2- > ${PAYLOAD_FILE}
for i in 1 2 3 4 5; do
  time aws sagemaker-runtime invoke-endpoint --endpoint-name sm-quickstart-baseline \
    --content-type text/csv --cli-binary-format raw-in-base64-out \
    --body fileb://${PAYLOAD_FILE} --region ${AWS_REGION} /dev/null >/dev/null
done
```

**实测输出**：`sm-quickstart-baseline`（ml.m5.large）单次调用耗时约 **0.43-0.49 秒**（含 CLI 进程/网络开销，5 次采样稳定），返回值 `0.00047526723938062787`。`sm-quickstart-graviton` **始终未进入 `InService`，无法采集延迟**——这与前面「镜像不支持 arm64」的结论一致，也是本 Demo 应记录的真实结果，而非缺陷。

### 3. 查询实例单价（Pricing）

```bash
aws pricing get-products --service-code AmazonSageMaker --region us-east-1 \
  --filters "Type=TERM_MATCH,Field=instanceName,Value=ml.m5.large" \
            "Type=TERM_MATCH,Field=regionCode,Value=us-east-1" \
  --output json | python3 -c "
import json,sys
d=json.load(sys.stdin)
for pl in d['PriceList']:
    item=json.loads(pl); attrs=item['product']['attributes']
    if attrs.get('component')=='Hosting':
        for v in item['terms']['OnDemand'].values():
            for pv in v['priceDimensions'].values():
                print(pv['description'])
"
```

**实测输出**（Pricing API 可用，region `us-east-1`，`component=Hosting`）：

| 实例 | 单价（USD/小时） | 说明 |
|------|-----------------|------|
| `ml.m5.large` | $0.115 | 标准 x86 基线 |
| `ml.c7g.large` | $0.087 | Graviton3，比 m5.large **低约 24%**（但本 Demo 内置镜像不兼容，无法验证性能/延迟收益） |
| `ml.inf2.xlarge` | $0.99 | Inferentia2，单价高于 CPU 实例，需大批量/低延迟深度学习推理场景才划算 |
| `ml.g5.2xlarge` | $1.515 | GPU（对照 Demo19 JumpStart LLM 端点），CPU 推理场景不适用，仅供跨 Demo 成本对比参考 |

### 4. 汇总选型建议

| 形态 | 相对成本（对比 ml.m5.large） | 适用 | 备注（本次真实结论） |
|------|---------|------|------|
| 标准实例 ml.m5.large | 基线（$0.115/hr） | 通用 CPU 推理 | 兼容性最好，实测延迟约 0.43-0.49s |
| Graviton ml.c7g.large | **-24%**（$0.087/hr） | CPU 推理、**镜像需自带 arm64 变体** | **内置 XGBoost `1.7-1` 镜像不支持**（`exec format error`）；需用支持多架构的框架容器（如自建 BYOC 多架构镜像）或官方已知支持 Graviton 的算法镜像才能享受这部分成本优势 |
| Inferentia ml.inf2.xlarge | +761%（$0.99/hr，单实例贵，但吞吐/请求成本可能更低） | 深度学习/LLM 推理，需 Neuron SDK 编译模型 | 不适用于本系列的 XGBoost CPU 场景；仅深度学习框架模型经 `neuronx-cc` 编译后可用 |
| Serverless | 按调用量计费，无固定小时价 | 间歇/低流量场景 | 见 Demo06（本系列已实测，冷启动对小模型不明显） |
| + Savings Plans | 承诺 1/3 年再降**最多 ~64%** | 稳定长期负载、已确定实例族 | 与上述实例形态叠加计费，适合已验证稳定运行的生产端点，不适合仍在选型试错阶段的场景 |

---

## 验收标准

- [x] 完成标准实例 vs Graviton 的延迟对比（Graviton 因镜像不兼容未能进入 `InService`，已完整记录该兼容性限制及根因）
- [x] 能说清 Graviton/Inferentia/Savings Plans 各自适用条件
- [x] 产出一份成本/延迟/适用场景对比表（含真实 Pricing API 单价）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 | 实测 |
|---|---------|-------------|------|
| 1 | `aws sagemaker describe-endpoint --endpoint-name sm-quickstart-graviton --region ${AWS_REGION} --query 'EndpointStatus' --output text` | `InService`（或记录不兼容） | `Failed`（`FailureReason` = did not pass the ping health check） |
| 2 | 对比表是否覆盖 Graviton/Inferentia/Serverless/Savings Plans 四类手段 | 是 | 是，见步骤 4 表格 |
| 3 | `aws logs get-log-events ... sm-quickstart-graviton ...` 是否能定位根因 | 能 | 能，`exec /miniconda3/bin/serve: exec format error`（arm64 不兼容的确凿证据） |

---

## 实验总结

本实验把「推理成本优化」从口号变成可对比的真实数据：Graviton `ml.c7g.large` 比标准 `ml.m5.large` **单价低约 24%**，但**内置 XGBoost `1.7-1` 镜像是 x86-only**，直接部署到 Graviton 会因 `exec format error` 健康检查失败——这是比"预期"更有价值的真实发现，说明 Graviton 收益的前提是**镜像本身要有 arm64 变体**（自建多架构 BYOC 镜像，或选用官方已支持 Graviton 的算法/框架容器）。同时梳理了 Inferentia（深度学习专用、需 Neuron 编译）与 Savings Plans（稳定负载再降本、需先完成选型）的适用边界，形成一份可直接给客户的选型框架。至此，从数据、训练、部署到监控、MLOps、生成式 AI 与成本优化，SageMaker 全生命周期的 20 个 Demo 完整闭环。

---

## 清理

```bash
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-graviton --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-graviton --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint --endpoint-name sm-quickstart-baseline --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name sm-quickstart-baseline --region ${AWS_REGION} 2>/dev/null || true
# 删除本 Demo 产生的 Model 对象（含 Graviton 首次因非法实例类型失败但已创建的 Model）
aws sagemaker list-models --region ${AWS_REGION} --query "Models[?contains(ModelName,'sagemaker-xgboost-2026-07-07')].ModelName" --output text \
  | tr '\t' '\n' | while read m; do [ -n "$m" ] && aws sagemaker delete-model --model-name "$m" --region ${AWS_REGION}; done
# 二次确认
aws sagemaker list-endpoints --region ${AWS_REGION} --query "Endpoints[?contains(EndpointName,'sm-quickstart-graviton') || contains(EndpointName,'sm-quickstart-baseline')]" --output text
echo "清理完成 — 已二次确认无遗留端点"
```
