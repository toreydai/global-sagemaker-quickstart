# Demo05 — 端点健康检查与故障排查

## 实验简介

模拟几种常见端点故障（权限错误、镜像错误、资源不足），演示如何通过端点状态字段和 CloudWatch 日志定位并修复问题。

**实验目标：**
- 掌握端点 `Failed`/异常状态的排查路径
- 理解 `/aws/sagemaker/Endpoints/<name>` CloudWatch 日志组的作用
- 能独立完成从报错现象到根因定位的排查流程

**实验流程：**
1. 故意用错误镜像触发一次部署失败
2. 从 `describe-endpoint` 的 `FailureReason` 字段定位问题类别
3. 查看 CloudWatch 日志组获取详细错误堆栈
4. 修复配置后重新部署，验证恢复为 `InService`

**预计 AI 执行时长：** 10 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x
- **权限**：SageMaker、CloudWatch Logs 只读
- **前提**：Demo04 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export ENDPOINT_NAME=<Demo04 创建的端点名>
```

---

> ⚠️ **实测重要变化**：现在 SageMaker 在 **`CreateModel` 阶段就同步校验镜像可拉取性与 `ModelDataUrl` 是否存在**。因此「不存在的镜像」「不存在的模型数据」这两类错误**不会**再让端点进入 `Failed`，而是 `CreateModel` 直接抛 `ValidationException`（fail-fast）。要复现「端点级失败 + CloudWatch 日志」的排查路径，需用**合法镜像 + 存在但损坏的模型**，让容器启动后崩溃。本 Demo 分两个场景演示。

## 步骤

### 1. 场景 A：错误镜像 / 缺失模型数据 —— `CreateModel` 同步报错

```bash
# A1. 不存在的镜像
python3 - <<'PY'
import os, boto3
sm = boto3.client("sagemaker", region_name=os.environ["AWS_REGION"])
try:
    sm.create_model(ModelName="broken-image", ExecutionRoleArn=os.environ["SM_ROLE_ARN"],
        PrimaryContainer={"Image":"000000000000.dkr.ecr.us-east-1.amazonaws.com/does-not-exist:latest",
                          "ModelDataUrl": os.environ["MODEL_DATA_URL"]})
except Exception as e:
    print("CreateModel 报错:", str(e).split(":")[-1].strip())
PY
```

**预期输出**（fail-fast，端点根本不会被创建）：
```
CreateModel 报错: ... Role ... cannot pull 000000000000.dkr.ecr.us-east-1.amazonaws.com/does-not-exist:latest. Ensure that the role exists and the image was granted pull permission.
```

若把 `ModelDataUrl` 指向不存在的 S3 对象，则报 `Could not find model data at s3://...`。**结论**：镜像/模型数据类错误在 `CreateModel` 就被拦截，无需等端点。

### 2. 场景 B：模型损坏 —— 端点级失败，走 CloudWatch 排查

```bash
# B1. 造一个存在但格式非法的 model.tar.gz（能过 CreateModel 校验，容器加载时崩溃）
mkdir -p /tmp/badmodel && echo "not a real xgboost model" > /tmp/badmodel/xgboost-model
tar -czf /tmp/bad-model.tar.gz -C /tmp/badmodel xgboost-model
aws s3 cp /tmp/bad-model.tar.gz s3://${SM_BUCKET}/model/bad/model.tar.gz --region ${AWS_REGION}

# B2. 用正确的 xgboost 镜像 + 损坏模型部署
python3 - <<'PY'
import os, boto3
from sagemaker.image_uris import retrieve
sm = boto3.client("sagemaker", region_name=os.environ["AWS_REGION"])
img = retrieve("xgboost", os.environ["AWS_REGION"], version="1.7-1")
bad = "s3://%s/model/bad/model.tar.gz" % os.environ["SM_BUCKET"]
sm.create_model(ModelName="broken-model", ExecutionRoleArn=os.environ["SM_ROLE_ARN"],
                PrimaryContainer={"Image": img, "ModelDataUrl": bad})
sm.create_endpoint_config(EndpointConfigName="broken-cfg",
    ProductionVariants=[{"VariantName":"v1","ModelName":"broken-model",
                         "InitialInstanceCount":1,"InstanceType":"ml.m5.large"}])
sm.create_endpoint(EndpointName="broken-ep", EndpointConfigName="broken-cfg")
print("submitted")
PY
```

**预期输出**：`submitted`。CreateModel 通过（模型数据存在），端点进入 `Creating`。

### 3. 观察状态并查 CloudWatch 日志（排查核心）

```bash
# 等 3-5 分钟让容器起来并崩溃，然后看日志（不必等到 Failed）
aws sagemaker describe-endpoint --endpoint-name broken-ep --region ${AWS_REGION} --query 'EndpointStatus' --output text

LS=$(aws logs describe-log-streams --log-group-name /aws/sagemaker/Endpoints/broken-ep \
     --region ${AWS_REGION} --query 'logStreams[0].logStreamName' --output text)
aws logs get-log-events --log-group-name /aws/sagemaker/Endpoints/broken-ep \
  --log-stream-name "$LS" --region ${AWS_REGION} --limit 20 --query 'events[].message' --output text
```

**预期输出**（关键排查信号）：
- 状态长时间停留在 `Creating`（容器**崩溃重启循环**，SageMaker 反复重试健康检查；实测 25 分钟仍 `Creating`，可能到 1 小时才转 `Failed`）。
- **日志组存在**（说明容器已启动，失败在容器内部而非镜像拉取阶段），日志含：
  ```
  RuntimeError: Model /opt/ml/model/xgboost-model cannot be loaded: ... BoostLearner: wrong model format
  [ERROR] Worker failed to boot.
  "GET /ping HTTP/1.1" 502
  ```

> 💡 **排查要点**：不要死等 `Failed`——崩溃循环时端点会长期 `Creating`，**根因早已在 CloudWatch 日志里**。日志组「存在与否」是关键分诊信号：存在 = 容器已启动（模型/代码问题）；不存在 = 更早的镜像拉取阶段失败（但那类错误现在多在 CreateModel 就被拦截）。

### 4. 修复方向

将 `ModelDataUrl` 换回 Demo03 产出的合法 `model.tar.gz`（`${MODEL_DATA_URL}`）重新部署即恢复 `InService`（即 Demo04 的正常路径）。诊断到「wrong model format」后，修复就是替换为正确的模型工件。

---

## 验收标准

- [ ] 理解镜像/模型数据类错误现在在 `CreateModel` 就 fail-fast（`ValidationException`）
- [ ] 能对损坏模型场景从 CloudWatch 日志读出根因（`wrong model format` / `Worker failed to boot` / `/ping 502`）
- [ ] 理解「日志组存在 = 容器已启动」这一分诊信号，且崩溃循环时端点长期停留 `Creating`
- [ ] 能说出正确修复方向（替换为合法模型工件）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | 场景 A CreateModel（错误镜像） | 抛 `ValidationException: ... cannot pull ...`（端点不被创建） |
| 2 | `aws logs describe-log-streams --log-group-name /aws/sagemaker/Endpoints/broken-ep --region ${AWS_REGION} --query 'logStreams[0].logStreamName' --output text` | 非空（如 `v1/i-xxxx`），证明容器已启动 |
| 3 | CloudWatch 日志内容 | 含 `BoostLearner: wrong model format` 与 `Worker failed to boot` |

---

## 实验总结

本实验通过刻意制造镜像错误，走了一遍「状态 → FailureReason → CloudWatch 日志」的标准排查路径，并认识到失败阶段不同（镜像拉取 vs 容器启动 vs 健康检查）对应不同的日志可见性。掌握这条链路后，后续任何端点异常都能快速定位。

---

## 清理

> ⚠️ **重要**：崩溃循环的端点会长期停留 `Creating` 并持续计费（ml.m5.large），排查到根因后**立即删除**，不要等它自己变 `Failed`。

```bash
aws sagemaker delete-endpoint --endpoint-name broken-ep --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-endpoint-config --endpoint-config-name broken-cfg --region ${AWS_REGION} 2>/dev/null || true
aws sagemaker delete-model --model-name broken-model --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/model/bad/model.tar.gz --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
