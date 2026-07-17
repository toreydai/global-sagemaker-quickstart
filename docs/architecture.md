# 架构文档

本仓库包含 20 个 Demo，这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Demo 提供架构图；其余 Demo 请直接看对应的 `docs/demoXX-*.md`。

以下 5 个 Demo 涉及多组件编排、异步作业链路、自定义镜像集成或多阶段状态流转，复杂度明显高于其余以"创建单个资源 + 调用验证"为主的 Demo，因此单独画图：

- **Demo13 — Model Monitor 数据漂移检测**：端点 Data Capture + 训练数据建 baseline + Processing 作业做监控执行的多阶段异步链路
- **Demo14 — BYOC 自带容器**：本地构建镜像 → ECR 推送 → SageMaker 按容器契约调度训练的完整链路
- **Demo16 — Feature Store 特征存储**：同一份特征定义拆分到在线（低延迟）/离线（S3 Parquet）两条并行存储链路
- **Demo17 — SageMaker Pipelines**：Processing/Training/Evaluation/Condition/RegisterModel 串联成 DAG 的流水线编排
- **Demo18 — Model Registry 与蓝绿/影子部署**：模型版本审批状态机 + 从 Registry 部署端点 + 历史 EndpointConfig 回滚

---

## Demo13 — Model Monitor 数据漂移检测

为 Demo04 的端点开启 Data Capture，用训练数据建立 baseline 统计量与约束，再用 on-demand 方式手动触发一次监控作业，检测线上流量与训练数据的分布漂移。关键点：定时 Monitoring Schedule 最小粒度为每小时，演示走 `CronExpressionGenerator.now()` 的一次性执行路径（实测约 11 分钟：排队 + Processing 作业）。

```mermaid
flowchart TB
  Train["S3: train/train.csv\n(训练数据)"]
  Endpoint["SageMaker Endpoint\nsm-quickstart-mon\n(Data Capture: enable=true, sampling=100%)"]
  Invoke["invoke-endpoint\n(30 次推理流量)"]
  Capture["S3: datacapture/\n(捕获的 endpointInput/Output, jsonl)"]

  BaselineJob["DefaultModelMonitor.suggest_baseline\n(Processing Job)"]
  Baseline["S3: monitor/baseline/\nstatistics.json + constraints.json"]

  MonSchedule["create_monitoring_schedule\nCronExpressionGenerator.now()\n(一次性 on-demand)"]
  MonJob["Monitoring Execution\n(Processing Job，对比 capture vs baseline)"]
  Report["S3: monitor/reports/\nconstraint_violations.json"]

  Train --> BaselineJob --> Baseline
  Endpoint --> Invoke --> Capture
  Baseline --> MonSchedule
  Capture --> MonSchedule
  MonSchedule --> MonJob --> Report
```

```mermaid
sequenceDiagram
  participant U as 执行者
  participant EP as Endpoint(Data Capture)
  participant S3 as S3
  participant Mon as DefaultModelMonitor

  U->>EP: 部署端点并开启 Data Capture
  U->>Mon: suggest_baseline(train.csv)
  Mon->>S3: 写入 statistics.json / constraints.json
  U->>EP: 30 次 invoke-endpoint（制造流量）
  EP->>S3: 落盘 capture jsonl（有延迟，需轮询）
  U->>Mon: create_monitoring_schedule(now())
  Mon->>S3: 读取 baseline + capture 数据
  Note over Mon: 约 11 分钟（排队约5分钟 + Processing约5分钟）
  Mon->>S3: 写入 constraint_violations.json
  U->>S3: 读取违规报告，定位漂移字段
```

---

## Demo14 — BYOC 自带容器

当内置算法与框架容器都不满足需求时，BYOC 是终极的灵活性出口：本地编写符合 SageMaker 容器契约（`train` 入口 + `/opt/ml/` 目录约定）的 Dockerfile，构建镜像推送到 ECR，SageMaker Training Job 直接引用该 ECR 镜像调度训练，产出模型工件写回 S3。

```mermaid
flowchart LR
  Local["本地构建环境\ntrain.py + Dockerfile\n(train 入口 = /opt/program/train.py)"]
  ECR["Amazon ECR\nsagemaker-quickstart-byoc:latest"]
  TrainingJob["SageMaker Training Job\nEstimator(image_uri=ECR镜像)\n instance: ml.m5.xlarge"]
  S3Train["S3: train/\n(RandomForest 训练数据)"]
  S3Model["S3: byoc-model/.../model.tar.gz\n(joblib 序列化模型)"]

  Local -->|docker build + docker push| ECR
  ECR -->|拉取自定义镜像| TrainingJob
  S3Train -->|TrainingInput| TrainingJob
  TrainingJob -->|"容器内 /opt/ml/input/data/train\n→ /opt/ml/model"| S3Model
```

> ⚠️ 训练/推理镜像的依赖版本必须锁定一致（如 `scikit-learn==1.7.2`），否则跨镜像 `joblib.load` 反序列化可能失败——生产 BYOC 的常见坑点。

---

## Demo16 — Feature Store 特征存储

建立同时开启在线库与离线库的 Feature Group：在线库供实时推理低延迟取特征，离线库（S3 Parquet，Hive 分区）供训练批量取特征，二者共享同一份特征定义（32 个特征 + record_id + event_time），从根源上避免训练服务偏差。

```mermaid
flowchart TB
  Src["breast_cancer 数据集\n(sklearn, 569 行 x 30 特征)"]
  Schema["FeatureGroup.load_feature_definitions\n(自动推断 32 个特征定义)"]
  FG["Feature Group: sm-quickstart-fg\n(enable_online_store=true)"]

  Ingest["fg.ingest()\nIngestionManagerPandas 多线程 PutRecord"]

  OnlineStore["在线库\n(低延迟 KV 存储)"]
  OfflineStore["离线库: S3\nfeature-store/.../offline-store/\nyear=/month=/day=/hour=/*.parquet"]

  GetRecord["sagemaker-featurestore-runtime\nget-record API"]

  Src --> Schema --> FG
  FG --> Ingest
  Ingest -->|同步写入| OnlineStore
  Ingest -.异步批量落盘\n5-6 分钟延迟.-> OfflineStore
  OnlineStore --> GetRecord
```

---

## Demo17 — SageMaker Pipelines 模型流水线

用 SageMaker Pipelines 把数据处理、训练、评估、条件注册串成一条可重复执行、可参数化、可追溯的 DAG，替代手工串联多个 Demo 的步骤。下图为完整流水线的目标形态（Processing → Training → Evaluation → Condition → RegisterModel）；当前 Demo 已跑通验证的是最小化的单 TrainingStep（`ParameterString` 化训练数据路径），其余 Step 类型待在此基础上扩展。

```mermaid
flowchart LR
  Param["ParameterString: TrainUri\n(s3://.../train/)"]
  Processing["ProcessingStep\n(数据处理，规划中)"]
  Training["TrainingStep\nEstimator(XGBoost 1.7-1)\n★ 当前已跑通"]
  Eval["EvaluationStep\n(模型评估，规划中)"]
  Condition["ConditionStep\n(按评估指标判断是否注册，规划中)"]
  Register["RegisterModelStep\n(注册到 Demo18 Model Registry，规划中)"]

  Param --> Training
  Processing -.规划中.-> Training
  Training --> Eval -.规划中.-> Condition -.规划中.-> Register
  Register -.-> ModelPkg["Model Package\n(供 Demo18 审批/部署)"]
```

```mermaid
sequenceDiagram
  participant U as 执行者
  participant Pipe as SageMaker Pipeline
  participant Exec as Pipeline Execution
  participant CW as CloudWatch/Steps

  U->>Pipe: pipeline.upsert(role_arn)
  Pipe-->>U: pipeline upserted
  U->>Pipe: start-pipeline-execution
  Pipe->>Exec: 触发 Execution（当前 DAG：TrainStep）
  loop 轮询状态
    U->>Exec: describe-pipeline-execution
    Exec-->>U: PipelineExecutionStatus
  end
  Exec-->>U: Succeeded
  U->>CW: list-pipeline-execution-steps 查看各 Step 状态
```

---

## Demo18 — Model Registry 与蓝绿/影子部署

用 Model Package Group 管理模型版本，经过审批状态机（`PendingManualApproval` → `Approved`）后直接从 Registry 部署端点，并演练基于历史 EndpointConfig 的回滚路径。蓝绿/影子（`ShadowProductionVariants`）为生产上线路径示意。

```mermaid
flowchart TB
  ModelData["S3: 模型工件\n(Demo17 Pipeline / Demo03 输出)"]
  Group["Model Package Group\nsagemaker-quickstart-models"]
  Register["model.register()\napproval_status=PendingManualApproval"]
  Package["Model Package v1\n(PendingManualApproval)"]
  Approve["update-model-package\n--model-approval-status Approved"]
  Deploy["ModelPackage.deploy()"]
  Endpoint["Endpoint: sm-quickstart-registry\n(InService)"]
  Configs["历史 EndpointConfig 列表\n(回滚候选)"]

  ModelData --> Register --> Group
  Register --> Package
  Package --> Approve
  Approve --> Deploy --> Endpoint
  Endpoint -. "update-endpoint 切换 Variant(生产: 蓝绿/影子)" .-> Configs
  Configs -. "回滚: 用旧 Config 再次 update-endpoint" .-> Endpoint
```

```mermaid
sequenceDiagram
  participant U as 执行者
  participant Reg as Model Registry
  participant EP as Endpoint

  U->>Reg: create-model-package-group
  U->>Reg: model.register(PendingManualApproval)
  Reg-->>U: ModelPackageArn (v1, PendingManualApproval)
  U->>Reg: update-model-package --model-approval-status Approved
  U->>Reg: ModelPackage.deploy()
  Reg->>EP: 创建端点 sm-quickstart-registry
  EP-->>U: InService，invoke-endpoint 验证推理结果
  U->>EP: list-endpoint-configs（查历史配置，备回滚）
  Note over U,EP: 回滚 = 用旧 EndpointConfig 再次 update-endpoint
```
