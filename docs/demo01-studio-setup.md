# Demo01 — 准备实验环境与 SageMaker Domain

## 实验简介

创建本系列所有 Demo 依赖的基础环境：数据桶、执行角色和 SageMaker Domain/Studio。作为第一个 Demo，重点是建立起可复用、可复制的最小环境基线。

**实验目标：**
- 理解 SageMaker Domain / User Profile / Studio 三者的关系
- 创建具备起步权限的执行角色（后续 Demo09 再收敛为最小权限）
- 能够独立打开 Studio 并验证 SDK 可用

**实验流程：**
1. 创建 S3 数据桶与 SageMaker 执行角色
2. 创建 SageMaker Domain（默认 VPC，IAM 认证）
3. 创建 User Profile 并启动 Studio
4. 在 Studio 内验证 boto3 / sagemaker SDK 可正常调用

**预计 AI 执行时长：** 10-15 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x
- **权限**：IAM（创建角色）、SageMaker（创建 Domain/UserProfile）、S3（创建桶）
- **前提**：无（本系列第一个 Demo）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SM_DOMAIN_NAME=demo-domain
export SM_ROLE_NAME=SageMakerQuickStartExecutionRole
export SM_BUCKET=sagemaker-quickstart-${ACCOUNT_ID}-${AWS_REGION}
```

---

## 步骤

### 1. 创建 S3 数据桶

```bash
aws s3 mb s3://${SM_BUCKET} --region ${AWS_REGION}
```

**预期输出**：`make_bucket: sagemaker-quickstart-...`

### 2. 创建执行角色并附加起步权限

```bash
cat > /tmp/sm-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sagemaker.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name ${SM_ROLE_NAME} \
  --assume-role-policy-document file:///tmp/sm-trust.json
aws iam attach-role-policy --role-name ${SM_ROLE_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

export SM_ROLE_ARN=$(aws iam get-role --role-name ${SM_ROLE_NAME} --query 'Role.Arn' --output text)
echo ${SM_ROLE_ARN}
```

**预期输出**：打印形如 `arn:aws:iam::<account>:role/SageMakerQuickStartExecutionRole`

> ⚠️ ARN 前缀必须是 `arn:aws:`（全球区），不要用 `arn:aws-cn:`。

### 3. 创建 SageMaker Domain（默认 VPC）

> ⚠️ **实测坑（默认 VPC 子网）**：本账号默认 VPC 里除了 6 个标准可用区子网外，还带一个 **Wavelength 区**子网（AZ `us-east-1-wl1-atl-wlz-1`）。若把它一并传给 `create-domain`，Domain 会在 `Pending → Failed` 后报 `AvailabilityZoneUnavailableError: ... subnet is in an Availability Zone that does not support EFS mount targets`。因此**必须过滤掉 Wavelength/Local Zone 子网**，只保留 `ZoneType==availability-zone` 的子网（下面命令用 `default-for-az=true` 且排除含 `wl` 的 AZ）。

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text --region ${AWS_REGION})
# 只取标准可用区（排除 Wavelength/Local Zone），每个 AZ 一个默认子网
export SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=${VPC_ID} Name=default-for-az,Values=true \
  --query 'Subnets[?starts_with(AvailabilityZone,`us-east-1`) && !contains(AvailabilityZone,`wl`)].SubnetId' \
  --output text --region ${AWS_REGION})
echo "使用子网：${SUBNET_IDS}"   # 应为 6 个标准 AZ 子网，不含 wl 区

aws sagemaker create-domain \
  --domain-name ${SM_DOMAIN_NAME} \
  --auth-mode IAM \
  --default-user-settings ExecutionRole=${SM_ROLE_ARN} \
  --vpc-id ${VPC_ID} \
  --subnet-ids ${SUBNET_IDS} \
  --region ${AWS_REGION}

# 轮询直到 InService
export DOMAIN_ID=$(aws sagemaker list-domains --region ${AWS_REGION} \
  --query "Domains[?DomainName=='${SM_DOMAIN_NAME}'].DomainId" --output text)
while true; do
  ST=$(aws sagemaker describe-domain --domain-id ${DOMAIN_ID} --region ${AWS_REGION} --query 'Status' --output text)
  echo "Domain 状态：${ST}"; [ "${ST}" = "InService" ] && break; sleep 30
done
```

**预期输出**：状态经历 `Pending → Pending → ... → InService`（实测约 2-3 分钟，5 次轮询内到 `InService`）

### 4. 创建 User Profile

```bash
aws sagemaker create-user-profile \
  --domain-id ${DOMAIN_ID} --user-profile-name demo-user \
  --region ${AWS_REGION}
```

**预期输出**：返回含 `UserProfileArn` 的 JSON

### 5. 验证 SDK

```bash
python3 -c "import sagemaker, boto3; print('sagemaker', sagemaker.__version__); print('caller', boto3.client('sts').get_caller_identity()['Account'])"
```

**预期输出**：打印 `sagemaker 2.235.2`（或更新版本）与当前账号 ID。

> ⚠️ 在 Python 3.9 上 `import boto3` 会打印一条 `PythonDeprecationWarning`（Boto3 2026-04-29 后不再支持 3.9），属正常告警，不影响执行。Studio 内置内核为 Python 3.10+，无此告警。

---

## 验收标准

- [ ] S3 数据桶创建成功
- [ ] 执行角色存在且已附加 `AmazonSageMakerFullAccess`
- [ ] Domain 状态为 `InService`
- [ ] User Profile 创建成功，SDK 可正常 import 与调用

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws s3 ls s3://${SM_BUCKET} --region ${AWS_REGION} >/dev/null && echo OK` | `OK` |
| 2 | `aws iam list-attached-role-policies --role-name ${SM_ROLE_NAME} --query 'AttachedPolicies[].PolicyName' --output text` | 包含 `AmazonSageMakerFullAccess` |
| 3 | `aws sagemaker describe-domain --domain-id ${DOMAIN_ID} --region ${AWS_REGION} --query 'Status' --output text` | `InService` |

---

## 实验总结

本实验搭建了后续所有 Demo 的公共底座：数据桶承载 raw/train/model/output 各阶段数据，执行角色为训练与推理提供身份，Domain/User Profile 提供 Studio 交互环境。接下来 Demo02 将在此基础上做数据预处理。

---

## 清理

```bash
# 注意：如需继续后续 Demo，请勿在此清理 Domain 与角色
aws sagemaker delete-user-profile --domain-id ${DOMAIN_ID} --user-profile-name demo-user --region ${AWS_REGION}
aws sagemaker delete-domain --domain-id ${DOMAIN_ID} --retention-policy HomeEfsFileSystem=Delete --region ${AWS_REGION}
aws iam detach-role-policy --role-name ${SM_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam delete-role --role-name ${SM_ROLE_NAME}
aws s3 rb s3://${SM_BUCKET} --force --region ${AWS_REGION}
echo "清理完成"
```
