# Demo14 — 自带容器 BYOC

## 实验简介

使用自定义训练/推理镜像（Bring Your Own Container）运行不在内置算法范围内的框架或自定义逻辑，构建镜像并推送到 ECR 后接入 SageMaker。

**实验目标：**
- 理解 BYOC 需遵循的 SageMaker 容器契约（`train`/`serve` 入口约定）
- 掌握镜像构建与 ECR 推送流程
- 能用自定义镜像完成一次训练或推理

**实验流程：**
1. 编写符合容器契约的 Dockerfile 与入口脚本
2. 本地构建镜像并推送到 ECR
3. 用自定义镜像提交 Training Job（或部署推理）
4. 验证自定义逻辑生效

**预计 AI 执行时长：** 15-18 分钟

---

## 前提条件

- **工具**：Docker、AWS CLI 2.x
- **权限**：ECR（创建仓库/推送镜像）、SageMaker
- **前提**：Demo01 已完成，本机具备 Docker 环境
- **初始化**：

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export ECR_REPO=sagemaker-quickstart-byoc
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

---

> ⚠️ **磁盘前置检查**：BYOC 需本地构建镜像。构建前先 `df -h /`；`python:3.10-slim` + scikit-learn/pandas/numpy/scipy 层解压后约 1GB。本机磁盘紧张（实测初始仅剩 ~4GB），构建前清理无用的 docker 资源：`docker image prune -f && docker builder prune -f`（只清悬空镜像和未使用的构建缓存，不影响运行中的容器/镜像）。实测清理后从 4GB 释放到 13GB。

## 步骤

### 1. 编写训练脚本与 Dockerfile

```bash
mkdir -p /tmp/byoc && cd /tmp/byoc
cat > train.py <<'PY'
# 简化：SageMaker 会以 `train` 为入口调用；数据在 /opt/ml/input/data/train
import os, pandas as pd
from sklearn.ensemble import RandomForestClassifier
import joblib, glob
files = glob.glob("/opt/ml/input/data/train/*.csv")
df = pd.concat([pd.read_csv(f, header=None) for f in files])
X, y = df.iloc[:,1:], df.iloc[:,0]
clf = RandomForestClassifier(n_estimators=50).fit(X, y)
os.makedirs("/opt/ml/model", exist_ok=True)
joblib.dump(clf, "/opt/ml/model/model.joblib")
print("train done")
PY

cat > Dockerfile <<'DOCKER'
FROM public.ecr.aws/docker/library/python:3.10-slim
RUN pip install --no-cache-dir scikit-learn pandas joblib
COPY train.py /opt/program/train.py
ENV PATH="/opt/program:${PATH}"
# SageMaker 以 `train` 命令启动：提供同名可执行入口
RUN printf '#!/usr/bin/env python3\nimport runpy; runpy.run_path("/opt/program/train.py", run_name="__main__")\n' > /usr/local/bin/train \
    && chmod +x /usr/local/bin/train
DOCKER
```

> ⚠️ **依赖不固定版本**：`pip install scikit-learn pandas joblib` 会拉最新版（实测装到 `scikit-learn 1.7.2 / pandas 2.3.3 / numpy 2.2.6 / scipy 1.15.3 / joblib 1.5.3`）。演示可接受，但生产 BYOC 应固定版本（如 `scikit-learn==1.7.2`）以保证可复现，且训练/推理镜像的 sklearn 版本须一致，否则 `joblib.load` 反序列化模型可能告警或失败。

### 2. 构建并推送镜像到 ECR

```bash
aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION} 2>/dev/null || true
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
docker build -t ${ECR_REGISTRY}/${ECR_REPO}:latest /tmp/byoc
docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
```

**预期输出**：`docker login` 显示 `Login Succeeded`；`docker build` 成功（实测基础层已缓存时约 11 秒，首次拉取基础镜像会更久）；`docker push` 结束显示 `latest: digest: sha256:... size: 1785`。

### 3. 用自定义镜像提交 Training Job

```bash
python3 - <<'PY'
import os, sagemaker
from sagemaker.estimator import Estimator
from sagemaker.inputs import TrainingInput
reg=os.environ["ECR_REGISTRY"]; repo=os.environ["ECR_REPO"]; role=os.environ["SM_ROLE_ARN"]; bucket=os.environ["SM_BUCKET"]
est = Estimator(image_uri=f"{reg}/{repo}:latest", role=role,
                instance_count=1, instance_type="ml.m5.xlarge",
                output_path=f"s3://{bucket}/byoc-model/",
                sagemaker_session=sagemaker.Session())
est.fit({"train": TrainingInput(f"s3://{bucket}/train/", content_type="text/csv")})
print("MODEL_DATA=", est.model_data)
print("JOB_NAME=", est.latest_training_job.name)
PY
```

**预期输出**：作业日志出现自定义脚本的 `train done`，随后 `Completed - Training job completed`、`Billable seconds: 39`（实测），并打印
`MODEL_DATA= s3://<bucket>/byoc-model/sagemaker-quickstart-byoc-<ts>/output/model.tar.gz`。

> ⚠️ **本机 Python 3.9 提示**：`fit()` 全程约 1 分钟（Starting→Downloading→Training→Uploading→Completed，实测计费 39 秒）。本机为 Python 3.9，`boto3` 会打印一条 `PythonDeprecationWarning`，非阻断可忽略。若需确认自定义脚本 `print` 是否落地，查 CloudWatch 日志组 `/aws/sagemaker/TrainingJobs`、流名 `<job-name>/algo-1-<ts>`，可见 `train done`。

---

## 验收标准

- [ ] 自定义镜像成功推送到 ECR
- [ ] 用自定义镜像的 Training Job 状态 `Completed`
- [ ] 能说清 SageMaker 容器契约（`train` 入口、`/opt/ml/` 目录约定）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecr describe-images --repository-name ${ECR_REPO} --region ${AWS_REGION} --query 'imageDetails[0].imageTags' --output text` | `latest` |
| 2 | `aws s3 ls s3://${SM_BUCKET}/byoc-model/ --recursive --region ${AWS_REGION} \| grep -c model.tar.gz` | `1` |
| 3 | `aws sagemaker describe-training-job --training-job-name ${JOB_NAME} --region ${AWS_REGION} --query TrainingJobStatus --output text` | `Completed` |
| 4 | 训练产出 `model.tar.gz` 大小 | 约 32 KB（RandomForest 50 棵树，实测 32842 字节） |

---

## 实验总结

本实验用自带容器跑通了一次训练：遵循 SageMaker 的容器契约（`train` 入口 + `/opt/ml/` 目录约定），把任意框架/自定义逻辑接入托管训练。当内置算法与框架容器都不满足需求时，BYOC 是终极的灵活性出口。

---

## 清理

```bash
aws ecr delete-repository --repository-name ${ECR_REPO} --force --region ${AWS_REGION} 2>/dev/null || true
aws s3 rm s3://${SM_BUCKET}/byoc-model/ --recursive --region ${AWS_REGION} 2>/dev/null || true
docker rmi ${ECR_REGISTRY}/${ECR_REPO}:latest 2>/dev/null || true
echo "清理完成"
```
