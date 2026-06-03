# BDCSPM-74932: DAC 分析算法 MQTT 输出数据通道

> **Parent**: BDCSPM-66856 — BLSS Analytics Plugin Platform Foundation (DAC Framework)  
> **Status**: Analyzing  
> **Fix Version**: BLSS 8.2.0  
> **Product**: BLSS (DCPM)

## 标题

**DAC 分析算法 MQTT 输出数据通道 — 基于 SparkplugB 格式通过 BLSS 监控基础设施回传分析结果**

## 描述

使分析算法插件能够通过 BLSS 内置 MQTT Broker，以标准 SparkplugB 载荷格式和 BLSS 标准 MQTT 监控模板，将分析结果数据回传给 BLSS 平台。BLSS 利用现有的监控数据接收和处理机制来处理算法输出，无需部署额外 Broker 或构建新的数据处理管道。

## 问题陈述

分析算法插件产生的分析结果需要被 BLSS 接收并用于后续处理、可视化和告警。需要一个标准化、受治理的输出通道，使得：

- 无论算法类型如何，插件均以统一方式传递结果
- BLSS 复用其现有的监控数据接收和处理机制（无需新建数据管道）
- 数据格式和传输方式在插件注册时已预先声明，平台可据此进行校验和治理

## 核心设计原则

### 1. 注册驱动声明（Registration-Driven Declaration）

当一个新的分析算法插件注册到 BLSS 时，以下元数据作为注册契约的一部分被声明：

- **算法标识** — 算法名称、版本、适用的设备型号（Device Model）
- **输出数据模板** — 该算法输出数据所遵循的 BLSS 标准 MQTT 监控模板（Monitoring Template），定义数据的 schema
- **数据传输方式** — 声明以 MQTT 作为传输协议（MQTT 是标准方法；未来可能支持其他方式）
- **适用设备型号** — 该算法适用于哪些设备型号

所有声明在注册时即持久化到 BLSS 系统，使平台能够自动验证、路由和处理数据。

### 2. 复用 BLSS 内置 MQTT Broker

- 算法插件直接连接 **BLSS 内部 MQTT Broker**（已作为 BLSS 平台基础设施的一部分部署）
- 无需额外的 Broker 部署、配置或维护
- 连接凭证和 Topic 分配由平台在插件注册过程中自动配置

### 3. SparkplugB 标准载荷格式

- 所有算法输出数据**必须**使用 **SparkplugB** 协议格式
- SparkplugB 提供自描述载荷（包含 metrics、时间戳、数据类型），确保互操作性
- 与 BLSS 现有数据解析基础设施兼容

### 4. BLSS 标准 MQTT 监控模板

- 输出数据的 schema 由 **BLSS 标准 MQTT 监控模板** 定义
- 模板声明算法将发布的指标（metrics）、数据类型和结构
- BLSS 使用现有的基于监控模板的数据处理管道接收数据 — 与设备监控数据使用相同的机制

### 5. 复用现有监控机制

- BLSS 从"虚拟设备"角度对待算法输出数据
- 平台现有的监控数据接收、解析、存储和告警机制无需修改即可复用
- 无需构建新的数据处理管道

## 范围内（In Scope）

- 定义算法输出声明的注册 schema（算法标识、输出模板、适用设备型号、传输方式）
- 实现插件注册时对输出声明的平台侧校验
- 实现已注册算法插件的 MQTT Topic 分配和凭证配置
- 实现入站算法结果消息的 SparkplugB 载荷校验
- 将算法输出映射到其声明的 BLSS MQTT 监控模板
- 将算法输出数据路由到现有 BLSS 监控数据管道进行处理
- 实现每插件 Topic 隔离和 ACL 访问控制
- 实现传输通道可观测性（消息计数、错误率、投递状态）

## 范围外（Out of Scope）

- 向插件传输数据（输入通道 — 独立关注点）
- 新的 MQTT Broker 部署或配置（复用现有）
- 非 MQTT 输出方式（未来范围）
- 算法逻辑或算法执行运行时
- 插件生命周期管理（属于 Plugin Manager 能力）
- 插件清单签名和安全验证（属于 Plugin Contract）
- 基于算法输出的告警/可视化规则（下游关注点）

## 依赖

| 依赖项 | 负责方 | 状态 | 延迟风险 |
| --- | --- | --- | --- |
| 插件注册契约（Point 1） | DAC Platform | — | 高 — 注册 schema 必须支持输出声明 |
| 运行时插件管理器（Point 2） | DAC Platform | — | 高 — 生命周期事件触发 MQTT 通道配置/拆除 |
| BLSS 内部 MQTT Broker 可用性 | BLSS Infra | 已可用 | 低 — 直接复用现有内部 Broker |
| BLSS MQTT 监控模板框架 | BLSS Monitoring | 已可用 | 中 — 输出模板须与现有模板引擎兼容 |

## 数据流概览

```
┌─────────────────────┐       SparkplugB         ┌──────────────────────────┐
│  分析算法插件        │ ──── MQTT Publish ──────► │  BLSS 内置 MQTT Broker    │
│  (算法输出结果)      │   (按声明的监控模板)       │                          │
└─────────────────────┘                           └───────────┬──────────────┘
                                                              │
                                                              ▼
                                                  ┌──────────────────────────┐
                                                  │  BLSS 监控数据管道（现有） │
                                                  │  - 基于模板的解析          │
                                                  │  - 数据存储               │
                                                  │  - 告警/后续处理          │
                                                  └──────────────────────────┘
```

## 注册时元数据示例（概念性）

```json
{
  "algorithm": {
    "id": "thermal-anomaly-detector",
    "version": "1.2.0",
    "applicableDeviceModels": ["UPS-9395", "UPS-93PM"]
  },
  "output": {
    "transmissionMethod": "MQTT",
    "payloadFormat": "SparkplugB",
    "monitoringTemplate": "blss-standard-analytics-output-v1",
    "publishFrequency": "on-event"
  }
}
```

## 验收标准（Acceptance Criteria）

### AC-1: 插件注册时声明 MQTT 输出通道

**Given** 一个分析算法插件提交注册请求到 BLSS  
**When** 注册契约中包含输出声明（传输方式=MQTT、载荷格式=SparkplugB、监控模板名称、适用设备型号列表）  
**Then** BLSS 平台成功验证并持久化该输出声明，返回注册成功确认

### AC-2: 注册时校验输出声明合法性

**Given** 一个插件提交的注册契约中包含输出声明  
**When** 声明中引用的监控模板不存在，或适用设备型号不在系统已知范围内，或载荷格式非 SparkplugB  
**Then** BLSS 拒绝注册并返回明确的校验错误信息，指明失败字段和原因

### AC-3: 注册成功后自动配置 MQTT Topic 和凭证

**Given** 一个插件已成功注册且输出声明已通过校验  
**When** 注册流程完成  
**Then** 平台为该插件分配专属 MQTT Topic、生成连接凭证，并配置对应的 ACL 规则（仅允许该插件向其专属 Topic 发布消息）

### AC-4: 算法插件通过 MQTT 发布 SparkplugB 格式数据

**Given** 一个已注册的算法插件使用分配的凭证连接到 BLSS 内置 MQTT Broker  
**When** 插件向其专属 Topic 发布符合 SparkplugB 格式的分析结果消息  
**Then** BLSS 成功接收该消息，并通过 SparkplugB 载荷校验

### AC-5: SparkplugB 载荷格式校验

**Given** BLSS 从插件专属 Topic 接收到一条消息  
**When** 消息载荷不符合 SparkplugB 格式规范  
**Then** BLSS 拒绝处理该消息，记录错误日志（包含插件 ID、Topic、错误详情），并递增该插件的错误计数器

### AC-6: 基于监控模板解析和处理数据

**Given** BLSS 接收到一条通过 SparkplugB 校验的算法输出消息  
**When** 消息内容与该插件注册时声明的 BLSS 标准 MQTT 监控模板匹配  
**Then** BLSS 使用现有监控数据管道对数据进行解析、存储，数据可在监控界面中被查询和展示

### AC-7: 监控模板不匹配处理

**Given** BLSS 接收到一条通过 SparkplugB 格式校验的消息  
**When** 消息中的 metrics 结构与注册时声明的监控模板不匹配（如字段缺失、类型不一致）  
**Then** BLSS 记录模板不匹配告警日志，消息进入异常队列而非正常处理管道

### AC-8: Topic 隔离和 ACL 强制执行

**Given** 多个算法插件已注册并分配了各自的专属 Topic  
**When** 插件 A 尝试向插件 B 的 Topic 发布消息  
**Then** MQTT Broker 拒绝该发布操作，并记录 ACL 违规日志

### AC-9: 传输通道可观测性

**Given** 一个或多个算法插件正在通过 MQTT 通道发送数据  
**When** 管理员查看通道监控仪表板  
**Then** 可查看每个插件的消息计数、错误率、最近消息时间戳和投递状态

### AC-10: 插件注销后通道清理

**Given** 一个已注册的算法插件被从 BLSS 注销（取消注册）  
**When** 注销流程完成  
**Then** 平台撤销该插件的 MQTT 凭证、删除 ACL 规则、关闭 Topic 订阅，后续该插件的连接请求被拒绝

### AC-11: 无需额外 Broker 部署

**Given** BLSS 系统已部署并运行内置 MQTT Broker  
**When** 新的分析算法插件注册并开始通过 MQTT 回传数据  
**Then** 整个过程无需管理员手动部署、配置或维护任何额外的 MQTT Broker 实例

---

## 与当前 Jira 版本的主要差异

| # | 当前版本 | 修订版本 |
|---|---------|---------|
| 1 | 定位为数据"传入插件"的输入通道 | 明确为算法结果"回传 BLSS"的输出通道 |
| 2 | 未提及载荷格式 | 明确 SparkplugB 为标准载荷格式 |
| 3 | 未提及监控模板 | 明确以 BLSS 标准 MQTT 监控模板定义数据 schema |
| 4 | 未强调注册时声明 | 核心设计以注册时声明为驱动 |
| 5 | 暗示需要新的 proxy/管道 | 明确复用现有监控数据接收处理机制，无需新管道 |
