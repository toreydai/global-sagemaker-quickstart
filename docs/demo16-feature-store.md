# Demo16 — Feature Store 特征存储

## 实验简介

使用 SageMaker Feature Store 建立在线/离线特征组，演示特征工程产出如何被训练和实时推理复用，避免训练服务偏差（training-serving skew）。

**实验目标：**
- 理解 Feature Store 在线库（低延迟查询）与离线库（S3 + Glue）的分工
- 掌握 Feature Group 的 schema 定义与 ingest 方式
- 能验证在线/离线两条链路数据的一致性

**实验流程：**
1. 定义 Feature Group schema（含 event time / record identifier）
2. 创建同时开启在线与离线存储的 Feature Group
3. 批量 ingest 特征数据
4. 分别从在线（GetRecord）和离线（S3）验证一致性

**预计 AI 执行时长：** 15 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、sagemaker Python SDK
- **权限**：SageMaker Feature Store、Glue、Athena、S3
- **前提**：Demo01 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1   # 必须：sagemaker.Session() 经 botocore 读 AWS_DEFAULT_REGION 解析 Region，仅设 AWS_REGION 会 ValueError
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_ROLE_ARN=$(aws iam get-role --role-name SageMakerQuickStartExecutionRole --query 'Role.Arn' --output text)
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

> ⚠️ **Region 环境变量坑**：`sagemaker.Session()` 若拿不到 Region 会抛 `ValueError: Must setup local AWS configuration with a region supported by SageMaker`。本机 botocore 只认 `AWS_DEFAULT_REGION`，仅 `export AWS_REGION` 不够，务必同时设置 `AWS_DEFAULT_REGION`。

---

## 步骤

### 1. 准备特征数据并定义 Feature Group

```bash
python3 - <<'PY'
import os, time, pandas as pd, sagemaker
from sklearn.datasets import load_breast_cancer
from sagemaker.feature_store.feature_group import FeatureGroup

sess = sagemaker.Session(); bucket=os.environ["SM_BUCKET"]; role=os.environ["SM_ROLE_ARN"]
d = load_breast_cancer(as_frame=True)
df = d.data.copy(); df.columns = [c.replace(" ","_") for c in df.columns]
df.insert(0, "record_id", range(len(df)))
df["event_time"] = float(round(time.time(), 3))
df = df.astype({c: "float64" for c in df.columns if c not in ["record_id"]})

fg = FeatureGroup(name="sm-quickstart-fg", sagemaker_session=sess)
fg.load_feature_definitions(data_frame=df)
fg.create(s3_uri=f"s3://{bucket}/feature-store",
          record_identifier_name="record_id", event_time_feature_name="event_time",
          role_arn=role, enable_online_store=True)
print("creating feature group...")
PY
```

**预期输出**：
```
creating feature group...
num_features 32 record_id_dtype int64
```
（创建异步；`load_feature_definitions` 自动推断 32 个特征定义 = 30 个乳腺癌数值特征 + `record_id`(Integral) + `event_time`(Fractional)。稍后轮询 `Created`）

> ⚠️ **event_time 类型**：`event_time` 用 Unix epoch 秒（`float`）即可，SDK 推断为 `Fractional`。若改用 ISO8601 字符串需自行声明为 `String` 类型，否则 ingest 报类型不匹配。

### 2. 轮询 Feature Group 状态并 ingest

```bash
python3 - <<'PY'
import os, time, pandas as pd, sagemaker
from sklearn.datasets import load_breast_cancer
from sagemaker.feature_store.feature_group import FeatureGroup
sess=sagemaker.Session()
fg=FeatureGroup(name="sm-quickstart-fg", sagemaker_session=sess)
while fg.describe()["FeatureGroupStatus"] != "Created": time.sleep(10)
d=load_breast_cancer(as_frame=True); df=d.data.copy()
df.columns=[c.replace(" ","_") for c in df.columns]
df.insert(0,"record_id",range(len(df))); df["event_time"]=float(round(time.time(),3))
df=df.astype({c:"float64" for c in df.columns if c!="record_id"})
resp=fg.ingest(data_frame=df, max_workers=3, wait=True)
print("ingested", len(df))
print("failed_rows", resp.failed_rows)
PY
```

**预期输出**：
```
ingested 569
failed_rows []
```
（`FeatureGroup` Creating→Created 约 20 秒即完成；`ingest` 用 `IngestionManagerPandas` 多线程 PutRecord，`failed_rows` 为空表示 569 条全部写入在线库成功）

### 3. 从在线库读取单条记录

```bash
aws sagemaker-featurestore-runtime get-record \
  --feature-group-name sm-quickstart-fg \
  --record-identifier-value-as-string "0" \
  --region ${AWS_REGION} --query 'Record[0]'
```

**预期输出**：返回 `record_id=0` 那条记录（在线库低延迟查询），`Record[0]` 即：
```json
{
    "FeatureName": "record_id",
    "ValueAsString": "0"
}
```
在线库为该 record 返回全部 32 个特征（`--query 'length(Record)'` = `32`）。

### 4. 确认离线库落 S3

离线库落 S3 有延迟（本次 ingest 完成后约 **5-6 分钟**才出现 parquet），需轮询：

```bash
for i in $(seq 1 20); do
  N=$(aws s3 ls s3://${SM_BUCKET}/feature-store/ --recursive --region ${AWS_REGION} | grep -c '\.parquet$')
  echo "poll $i: parquet=$N"; [ "$N" -gt 0 ] && break; sleep 30
done
aws s3 ls s3://${SM_BUCKET}/feature-store/ --recursive --region ${AWS_REGION} | grep '\.parquet$' | head -3
```

**预期输出**：出现 Hive 分区路径下的多个 `.parquet` 文件（本次为 **54 个**，合计 **569 行**）：
```
feature-store/<account>/sagemaker/us-east-1/offline-store/sm-quickstart-fg-<ts>/data/year=2026/month=07/day=06/hour=08/20260706T0828..Z_<rand>.parquet
```

> ⚠️ **离线库延迟 + 首个 0 字节占位文件**：ingest 后立即 `ls` 只会看到一个 0 字节 `sm-quickstart-fg<ISO>.txt` 占位标记，真实 parquet 分区数据要等 5-6 分钟才由 Feature Store 后台批量写入 `.../offline-store/.../data/year=/month=/day=/hour=/` 下。用 `grep '.parquet$'` 判定落地，勿被占位 `.txt` 误导。离线 parquet 比在线多 3 个元数据列（`write_time`、`api_invocation_time`、`is_deleted`），共 35 列。

---

## 验收标准

- [ ] Feature Group 状态 `Created`；`OnlineStoreConfig.EnableOnlineStore=true` 且 `OfflineStoreConfig.S3StorageConfig.S3Uri` 指向 `s3://<bucket>/feature-store`
- [ ] `ingest` 返回 `failed_rows=[]`，569 条全部写入
- [ ] 在线库 GetRecord 对 record_id=0 返回 32 个特征；离线库 5-6 分钟后出现 `.parquet`（54 个文件 / 569 行 / 35 列）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws sagemaker describe-feature-group --feature-group-name sm-quickstart-fg --region ${AWS_REGION} --query 'FeatureGroupStatus' --output text` | `Created` |
| 2 | `aws sagemaker describe-feature-group --feature-group-name sm-quickstart-fg --region ${AWS_REGION} --query 'OnlineStoreConfig.EnableOnlineStore' --output text` | `True` |
| 3 | `aws sagemaker-featurestore-runtime get-record --feature-group-name sm-quickstart-fg --record-identifier-value-as-string "0" --region ${AWS_REGION} --query 'Record[0].ValueAsString' --output text` | `0` |
| 4 | `aws sagemaker-featurestore-runtime get-record --feature-group-name sm-quickstart-fg --record-identifier-value-as-string "0" --region ${AWS_REGION} --query 'length(Record)'` | `32` |
| 5 | `aws s3 ls s3://${SM_BUCKET}/feature-store/ --recursive --region ${AWS_REGION} \| grep -c '\.parquet$'`（离线落地后） | `54`（≥1 即通过） |

---

## 实验总结

本实验建立了同时开启在线/离线存储的 Feature Group：在线库供实时推理低延迟取特征，离线库供训练批量取特征，二者共享同一份特征定义，从根源上避免训练服务偏差。特征复用是规模化 ML 团队的关键基础设施。

---

## 清理

> ⚠️ **`delete-feature-group` 不删离线 S3**：删除 Feature Group（含在线库）后，离线库落在 S3 的 parquet **不会**被一并清除，必须显式 `aws s3 rm`，否则 S3 残留计费。删除为异步（`Deleting`→`ResourceNotFound`，本次约 10-20 秒）。

```bash
aws sagemaker delete-feature-group --feature-group-name sm-quickstart-fg --region ${AWS_REGION} 2>/dev/null || true
# 轮询确认删除完成
for i in $(seq 1 12); do
  aws sagemaker describe-feature-group --feature-group-name sm-quickstart-fg --region ${AWS_REGION} >/dev/null 2>&1 || { echo "FG 已删除"; break; }
  sleep 10
done
aws s3 rm s3://${SM_BUCKET}/feature-store/ --recursive --region ${AWS_REGION} 2>/dev/null || true
echo "清理完成"
```
