# SLB-FPE API 接口技术文档

> **版本**：V2.0（完整版）| **适用系统**：SLB-FPE 教育诊断与干预生态系统 V12.0
> **基地址**：`https://api.slb-fpe.edu/v1` | **协议**：HTTPS / JSON / RESTful
> **编码**：UTF-8 | **日期格式**：ISO 8601

---

## 目录

1. [认证与鉴权](#1-认证与鉴权)
2. [核心接口定义](#2-核心接口定义)
   - 2.1 评估提交 | 2.2 报告生成 | 2.3 学习者画像 | 2.4 干预匹配
   - 2.5 联邦学习 | 2.6 多模态上传 | 2.7 数据导出 | 2.8 批量评估
   - 2.9 预警推送 | 2.10 量表管理
3. [Webhook 回调规范](#3-webhook-回调规范)
4. [错误码全集](#4-错误码全集)
5. [SDK 与快速集成](#5-sdk-与快速集成)
6. [部署架构与性能指标](#6-部署架构与性能指标)
7. [安全合规清单](#7-安全合规清单)

---

（第1-2.6节保持原有内容不变，此处省略，接续第2.7节）

---

### 2.7 数据导出接口（合规导出）

支持用户/家长导出个人数据，满足《个人信息保护法》的可携带权要求。

| 属性 | 值 |
|------|-----|
| **接口** | `POST /api/v1/data/export` |
| **权限** | `data:export` |
| **延迟要求** | P99 < 30秒（小数据）/ < 5分钟（全量数据） |

**请求体**：

```json
{
  "user_id": "stu_20240801001",
  "scope": "all",
  "format": "json",
  "include_children": ["assessment_history", "intervention_log", "behavior_log", "reports"],
  "date_range": {"from": "2023-09-01", "to": "2024-06-15"},
  "verification": {
    "method": "sms",
    "code": "123456"
  }
}
```

**响应体**：

```json
{
  "success": true,
  "data": {
    "export_id": "exp_20240615_001",
    "status": "processing",
    "estimated_size_mb": 12.5,
    "expires_at": "2024-06-22T10:05:12Z",
    "download_url": "https://api.slb-fpe.edu/v1/data/export/download?export_id=exp_20240615_001",
    "formats_available": ["json", "csv", "pdf"]
  }
}
```

**删除请求（被遗忘权）**：

```http
DELETE /api/v1/data/purge
Authorization: Bearer {TOKEN}
Content-Type: application/json

{"user_id": "stu_20240801001", "reason": "user_request", "verification_code": "789012"}
```

---

### 2.8 批量评估接口

用于学校/区域级大规模统一施测场景。

| 属性 | 值 |
|------|-----|
| **接口** | `POST /api/v1/assessment/batch` |
| **权限** | `assessment:write` + `batch:admin` |
| **频率限制** | 5次/天/学校 |

**请求体**：

```json
{
  "batch_id": "batch_schoolA_2024fall",
  "school_id": "sch_bj_hd_001",
  "assessment_config": {
    "type": "student_standard",
    "version": "v2024.09",
    "modules": ["M1","M2","M3","M4","M5","M6","M7","M8","M9","M10","M11","M12"],
    "schedule": {"start": "2024-09-15T08:00:00Z", "end": "2024-09-20T17:00:00Z"},
    "settings": {"allow_resume": true, "time_limit_minutes": 50}
  },
  "participants": [
    {"user_id": "stu_001", "grade": "P5", "class": "3"},
    {"user_id": "stu_002", "grade": "P5", "class": "3"}
  ]
}
```

**响应体**：

```json
{
  "success": true,
  "data": {
    "batch_id": "batch_schoolA_2024fall",
    "total_participants": 320,
    "status_counts": {"pending": 320, "in_progress": 0, "completed": 0},
    "monitor_url": "/api/v1/assessment/batch/status?batch_id=batch_schoolA_2024fall",
    "expected_completion": "2024-09-20T17:00:00Z"
  }
}
```

---

### 2.9 预警推送接口

主动向学校管理者推送高风险学生预警。

| 属性 | 值 |
|------|-----|
| **接口** | `POST /api/v1/alert/push` |
| **权限** | `alert:write`（系统内部调用） |
| **触发方式** | 评估提交后自动触发 / Webhook 回调 |

**推送体（系统→学校）**：

```json
{
  "alert_id": "alt_20240615_042",
  "severity": "critical",
  "type": "self_harm_risk",
  "user_id": "stu_20240801001",
  "triggered_by": {
    "session_id": "sess_abc123",
    "rules": ["M4_Q081=5", "M12_OPEN='我不想活了'"],
    "timestamp": "2024-06-15T10:05:12Z"
  },
  "required_actions": ["immediate_contact_parent", "counselor_notify", "principal_notify"],
  "recipients": [
    {"role": "counselor", "contact": "138****6789", "channel": "sms"},
    {"role": "principal", "contact": "principal@school.edu", "channel": "email"}
  ]
}
```

---

### 2.10 量表管理接口

管理量表题库的增删改查，支持IRT参数更新。

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/items` | GET | 查询题库（支持模块/年级/学科筛选） |
| `/api/v1/items/{item_id}` | PUT | 更新题目内容和参数 |
| `/api/v1/items/batch` | POST | 批量导入新题目（AI生成+专家审核流程） |
| `/api/v1/norms` | GET | 查询常模数据（分区域/年级/性别） |
| `/api/v1/norms/update` | POST | 触发常模更新（管理员） |

**题目更新示例**：

```json
{
  "item_id": "M1_Q027",
  "content": "我每天放学后会先列出当晚要完成的学习任务清单",
  "options": {"type": "likert_5", "labels": ["完全不符合","较不符合","一般","较符合","完全符合"]},
  "irt_params": {"a": 1.42, "b": -0.35, "c": 0.12},
  "dif_check": {"gender_bias": 0.08, "urban_rural_bias": 0.15},
  "status": "active"
}
```

---

## 3. Webhook 回调规范

### 3.1 可用事件

| 事件 | 触发时机 | 回调延迟 |
|------|---------|---------|
| `assessment.completed` | 学生/家长完成全部答题 | < 5秒 |
| `report.generated` | 报告生成完毕（含异步PDF） | < 10秒 |
| `alert.triggered` | 系统触发风险预警 | < 30秒 |
| `intervention.assigned` | 为学生匹配了干预方案 | < 5秒 |
| `batch.completed` | 批量施测全部完成 | < 60秒 |
| `data.export_ready` | 数据导出文件就绪 | < 5分钟 |

### 3.2 回调格式

```json
{
  "event": "assessment.completed",
  "timestamp": "2024-06-15T10:05:12Z",
  "data": {
    "session_id": "sess_abc123",
    "user_id": "stu_20240801001",
    "assessment_type": "student_standard",
    "total_time_seconds": 1482
  },
  "signature": "sha256=abcd1234..."
}
```

### 3.3 签名验证

接收方应验证签名以防止伪造回调：

```python
import hmac, hashlib

def verify_signature(payload, signature, secret):
    expected = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# 使用
payload = json.dumps(data)  # 原始JSON字符串
is_valid = verify_signature(payload, request.headers['X-SLB-Signature'], WEBHOOK_SECRET)
```

### 3.4 重试策略

| 尝试 | 延迟 | 策略 |
|------|------|------|
| 第1次 | 即时 | — |
| 第2次 | 30秒后 | — |
| 第3次 | 5分钟后 | — |
| 第4次 | 30分钟后 | — |
| 第5次 | 2小时后 | 最后一次，失败后标记为 dead letter |

连续失败3次后自动禁用回调，需管理员手动重新激活。

---

## 4. 错误码全集

### 4.1 HTTP 状态码约定

| 状态码 | 含义 | 典型场景 |
|--------|------|---------|
| 200 | 成功 | GET/PUT 成功 |
| 201 | 已创建 | POST 创建资源成功 |
| 202 | 已接受 | 异步处理中（如PDF生成） |
| 204 | 无内容 | DELETE 成功 |
| 400 | 请求错误 | 参数校验失败 |
| 401 | 未认证 | Token 缺失或无效 |
| 403 | 无权限 | 权限不足 |
| 404 | 未找到 | 资源不存在 |
| 409 | 冲突 | 重复提交、状态冲突 |
| 429 | 限流 | 超过频率限制 |
| 500 | 服务器错误 | 内部异常 |
| 503 | 服务不可用 | 维护中或过载 |

### 4.2 业务错误码

| 错误码 | HTTP | 含义 | 建议处理 |
|--------|------|------|---------|
| `AUTH_TOKEN_EXPIRED` | 401 | Token 过期 | 使用 Refresh Token 刷新 |
| `AUTH_TOKEN_REVOKED` | 401 | Token 已被撤销 | 重新登录 |
| `AUTH_INSUFFICIENT_SCOPE` | 403 | Token 权限不足 | 申请更高权限 |
| `ASSESSMENT_SESSION_EXPIRED` | 400 | 测评会话超时 | 重新开始测评 |
| `ASSESSMENT_DUPLICATE` | 409 | 重复提交 | 检查 session_id |
| `ASSESSMENT_ITEM_NOT_FOUND` | 404 | 题目不存在 | 检查 item_id |
| `REPORT_NOT_READY` | 202 | 报告生成中 | 轮询 status 接口 |
| `REPORT_STALE_DATA` | 400 | 评估数据已过期 | 重新施测 |
| `PROFILE_NOT_FOUND` | 404 | 无评估记录 | 引导用户完成首次评估 |
| `BATCH_EXCEEDS_LIMIT` | 400 | 批量人数超限 | 单批≤5000人 |
| `DATA_EXPORT_IN_PROGRESS` | 409 | 导出已在进行 | 等待或取消 |
| `RATE_LIMIT_EXCEEDED` | 429 | 请求频率超限 | 等待 Retry-After 秒 |
| `MULTIMODAL_FILE_TOO_LARGE` | 400 | 上传文件超限 | 音频≤5MB，视频≤20MB |
| `FEDERATED_NODE_UNAUTHORIZED` | 403 | 节点未授权 | 联系管理员注册节点 |
| `INTERNAL_ERROR` | 500 | 服务器内部错误 | 联系技术支持 |

---

## 5. SDK 与快速集成

### 5.1 Python SDK

```bash
pip install slb-fpe-sdk
```

```python
from slb_fpe import SLBClient

client = SLBClient(
    base_url="https://api.slb-fpe.edu/v1",
    client_id="your_client_id",
    client_secret="your_client_secret"
)

# 学生施测
session = client.assessment.start(
    user_id="stu_001",
    assessment_type="student_standard"
)

# 提交单题
result = client.assessment.submit(
    session_id=session.id,
    item_id="M1_Q001",
    value=4,
    time_ms=3500
)

# 获取报告
report = client.report.get(
    user_id="stu_001",
    format="json",
    audience="teacher"
)

# 匹配干预
interventions = client.intervention.match(
    user_id="stu_001",
    target_modules=["M3", "M4"]
)
```

### 5.2 JavaScript/Node.js SDK

```bash
npm install @slb-fpe/api-client
```

```typescript
import { SLBClient } from '@slb-fpe/api-client';

const client = new SLBClient({
  baseURL: 'https://api.slb-fpe.edu/v1',
  clientId: 'your_client_id',
  clientSecret: 'your_client_secret'
});

// 启动自适应测试
const session = await client.assessment.startAdaptive({
  userId: 'stu_001',
  grade: 'P5'
});

// 订阅预警
client.webhooks.subscribe('alert.triggered', async (event) => {
  console.log(`预警: ${event.data.userId} - ${event.data.type}`);
  // 集成学校IM/短信通知
});
```

### 5.3 第三方平台集成（REST）

```bash
# 获取学校批量报告
curl -X POST "https://api.slb-fpe.edu/v1/assessment/batch" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @batch_config.json

# 查看批量进度
curl "https://api.slb-fpe.edu/v1/assessment/batch/status?batch_id=batch_schoolA_2024fall" \
  -H "Authorization: Bearer $TOKEN"
```

---

## 6. 部署架构与性能指标

### 6.1 部署拓扑

```
                   [CDN / WAF]
                        │
                   [API Gateway] (Kong / Nginx)
                        │
            ┌───────────┼───────────┐
            │           │           │
       [评估引擎]   [报告引擎]   [分析引擎]
       (Gunicorn)  (Celery)    (Spark/Flink)
            │           │           │
            └───────────┼───────────┘
                        │
              [PostgreSQL + Redis + S3]
                        │
              [联邦学习节点 × 区域]
```

### 6.2 性能指标

| 指标 | 目标值 | 测量方式 |
|------|--------|---------|
| API 可用性 | ≥ 99.9% | 每月 uptime 统计 |
| 评估提交 P99延迟 | < 200ms | Prometheus + Grafana |
| 报告生成 P99延迟 | < 3秒（JSON）/ < 5秒（PDF） | 同上 |
| 画像查询 P99延迟 | < 500ms | 同上 |
| 并发连接数 | 10,000+ | 压测验证 |
| 单次批量评估 | ≤ 5,000人 | 压力测试 |
| 数据存储冗余 | 3副本（异地） | 运维审计 |
| RTO（恢复时间） | < 30分钟 | 灾备演练 |
| RPO（数据丢失） | < 5分钟 | 每日增量+每周全量备份 |

### 6.3 容量规划

| 规模 | 学校数 | 学生数 | 并发峰值 | 服务器配置 |
|------|--------|--------|---------|-----------|
| 试点 | 10 | 5,000 | 100 | 4C8G × 3 |
| 区级 | 100 | 50,000 | 1,000 | 8C16G × 10 |
| 市级 | 500 | 500,000 | 10,000 | 16C32G × 30 + K8s 自动伸缩 |
| 省级 | 5,000 | 5,000,000 | 100,000 | 微服务 + 多云部署 |

---

## 7. 安全合规清单

### 7.1 数据传输安全

- [x] 全站 HTTPS（TLS 1.3），禁用 TLS 1.0/1.1
- [x] API 鉴权：OAuth 2.0 + JWT（RS256签名）
- [x] 敏感数据传输使用国密 SM4 加密
- [x] 联邦学习梯度传输使用 Paillier 同态加密
- [x] 请求级全链路追踪（X-Request-ID）

### 7.2 数据存储安全

- [x] 数据库加密：AES-256（静态）+ SM4（传输）
- [x] 密钥管理：HSM（硬件安全模块）或云 KMS
- [x] 数据分级：敏感（心理评估）> 内部（行为日志）> 公开（聚合统计）
- [x] 数据保留：学生数据保留至毕业后5年，届时自动删除
- [x] 备份：每日增量 + 每周全量，异地存储，保留90天

### 7.3 访问控制

- [x] RBAC（角色权限） + ABAC（属性权限）双模型
- [x] 最小权限原则：默认拒绝，逐项授权
- [x] 操作审计：所有敏感操作记录日志，保留不少于6个月
- [x] 多因素认证（MFA）用于管理员操作

### 7.4 隐私合规

- [x] 《个人信息保护法》合规：知情同意、最小必要、可携带、可删除
- [x] 《数据安全法》合规：分类分级、风险评估、应急响应
- [x] 《未成年人保护法》合规：监护人授权、数据采集限制
- [x] 等保 2.0 三级评测通过
- [x] 每年第三方安全渗透测试
- [x] IRB（机构伦理审查委员会）年度审查

### 7.5 安全响应 SLA

| 事件等级 | 响应时间 | 通知范围 | 善后措施 |
|---------|---------|---------|---------|
| P0 数据泄露 | 15分钟内启动应急 | 监管机构+受影响用户+学校 | 根因分析+安全加固+保险理赔 |
| P1 系统入侵 | 30分钟内隔离 | CISO+运维团队 | 漏洞修复+攻击溯源 |
| P2 漏洞发现 | 24小时内评估 | 技术团队 | 补丁+验证 |
| P3 配置错误 | 72小时内修复 | 内部 | 配置审计+自动化检查 |

---

> **完整版说明**：本文档 V2.0 覆盖了原 V1.0 的 6 个核心接口 + 新增的 4 个接口（批量评估、预警推送、量表管理、数据导出）+ Webhook 规范 + SDK 示例 + 部署架构 + 安全合规清单。与 SLB-FPE 完整版手册 V12.0 配套使用。
