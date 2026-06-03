# BDCSPM-70937 — Epic: RDGC MQTT Protocol Support (Ported from Probe Server)

## Capability Reference

| Field | Value |
|---|---|
| **Capability** | BDCSPM-70937: RDGC has the ability to monitor devices using MQTT |
| **Parent** | BDCSPM-6550: RDGC monitoring protocol parity with BLSS |
| **Fix Version** | BLSS 8.2.0 (2027-01-15) |
| **Priority** | High |
| **Dependency** | BDCSPM-6541: RDGC device multi-protocol auto discovery |
| **Labels** | Customer-SustainingEngineering, DIT, RDG |
| **Reference Implementation** | Probe Server — MQTT protocol handler (已上线) |

---

### EPIC: BDCSPM-70937-RDGC — Port MQTT Protocol Monitoring from Probe Server to RDGC

**BLSS Product(s)**: Distributed IT Performance Management (RDGC component)

**Summary**
将现有 Probe Server 上已实现的 MQTT 协议监控能力移植到 RDGC（Remote Data Gateway Client），复用现有的协议处理架构、Monitor Configuration 模型和 Polling Groups 设置模式，使 RDGC 管理的设备也能通过 MQTT 协议进行数据采集和监控。

**Description**

- **Problem statement**: Probe Server 已经实现了 MQTT 协议监控能力（如截图所示，MQTT 已在 Monitor Configuration 的协议列表中），但 RDGC 尚不支持 MQTT。DIT（Distributed IT）场景下的边缘站点和远程站点通常使用 RDGC 而非 Probe Server，这些站点中的 MQTT 设备当前无法被监控。
- **Proposed solution**: 将 Probe Server 的 MQTT 协议处理器（Protocol Handler）移植至 RDGC 架构，复用以下已有技术：
  - Protocol Handler 插件架构（与 SNMP、Modbus、BACnet、OPC UA 等共存）
  - Monitor Configuration 模型（IP Address、Probe/RDG 关联、协议选择）
  - Monitoring Templates 模板机制
  - Polling Groups Settings（Baseline Interval、Priority 倍率、Retries、Timeout）
  - 设备属性面板与协议参数面板的 UI 模式
- **User personas**: IT/OT Administrators, Operations Teams, System Integrators
- **Business value**: 
  - 实现 RDGC 与 Probe Server 的协议能力对等（Protocol Parity）
  - 复用已验证的实现大幅降低开发风险和工时
  - 为 DIT 边缘场景客户提供 MQTT 设备监控能力
- **Key workflows** (对齐 Probe Server 已有流程):
  1. 管理员在 Device Monitor Configuration 中选择 MQTT 协议
  2. 配置 MQTT 特有参数（Broker Host/Port、Topic、QoS、认证方式、TLS 等）
  3. 关联 Monitoring Template 定义采集点位
  4. 配置 Polling Groups Settings（Baseline Interval、Priority 倍率）
  5. RDGC 建立 MQTT 连接并订阅 Topic 接收数据
  6. 采集数据纳入现有设备监控流程（Graph、Alarm、Trap 等）
- **Cross-product touchpoints**: 与 Probe Server 共享 Protocol Handler 代码库；Monitor Configuration 模型一致性
- **UX/UI references**: 
  - 参考已有 Probe Server Monitor Config 页面（见附图）
  - 协议列表中新增 MQTT 选项（已在 Probe Server 中存在）
  - MQTT 协议参数面板：类似 SNMP 参数面板的布局模式
- **On-prem operational impact**:
  - Install/upgrade impact: RDGC 安装包新增 MQTT Protocol Handler 模块；配置 schema 扩展
  - Sizing impact: 与 Probe Server 同等负载下资源消耗一致；RDGC 边缘硬件需验证
  - Network impact: 出站端口 8883 (MQTTS) / 1883 (MQTT)；防火墙文档更新
  - Backup impact: MQTT 配置数据纳入 RDGC 标准备份

---

## 已有实现参考（Probe Server）

从截图可知 Probe Server 已实现以下能力，RDGC 应直接复用：

| 已有能力 | Probe Server 实现 | RDGC 移植策略 |
|---|---|---|
| 协议列表 | Monitor Config → Protocol List 包含 MQTT | 复用协议注册机制，在 RDGC 中注册 MQTT Handler |
| 协议配置面板 | 右侧参数面板（参考 SNMP 面板：Port/Protocol/Version/Community/Security） | 实现 MQTT 专属面板：Broker Host/Port/ClientID/QoS/Topic/Auth/TLS |
| Monitoring Templates | 支持为每种协议定义采集模板 | 复用 Template 引擎，新增 MQTT 类型模板 |
| Polling Groups | Baseline Interval + Priority 倍率 + Retries + Timeout | MQTT 为推送模式，适配为 "Expected Update Interval" + 超时告警 |
| 设备绑定 | Device → Monitor Config → Probe/RDG Server 关联 | 替换 Probe 为 RDGC；保持 Device → Monitor Config 关联模型 |
| 连接管理 | Probe Server 管理协议连接 | RDGC 本地管理 MQTT 连接（无需经 Probe 中转） |

---

## Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Security | MQTT 连接加密 | TLS 1.2+；与 Probe Server 一致 | P0 |
| Security | 凭证存储 | 复用 RDGC 现有加密存储机制 | P0 |
| Reliability | 断线自动重连 | 30s 内首次重连；指数退避至 5min | P0 |
| Scalability | 单 RDGC 实例支持 MQTT 设备数 | ≥ 50 MQTT 设备（RDGC 硬件限制） | P1 |
| Scalability | Topic 订阅数 | ≥ 200 Topic/RDGC 实例 | P1 |
| Performance | MQTT 模块额外 CPU 开销 | < 10% (RDGC 边缘硬件基准) | P1 |
| Performance | 额外内存占用 | < 128 MB | P1 |
| Compatibility | MQTT 协议版本 | v3.1.1 和 v5.0（与 Probe Server 一致） | P1 |
| Compatibility | 与现有协议共存 | SNMP/Modbus/BACnet 等不受影响 | P0 |
| Observability | 连接状态 | 设备 Dashboard 可见连接状态 | P1 |

---

## Acceptance Criteria (Epic-level)

- [ ] AC1: RDGC Monitor Configuration 协议列表中出现 MQTT 选项（与 Probe Server 一致）
- [ ] AC2: 管理员可通过 UI 配置 MQTT 协议参数（Broker、Port、Topic、Auth、TLS）
- [ ] AC3: RDGC 可建立 TLS 加密 MQTT 连接并订阅 Topic 接收设备遥测数据
- [ ] AC4: MQTT 采集的数据纳入现有监控流程（Graph、Alarm Panel、Traps）
- [ ] AC5: Monitoring Templates 支持 MQTT 类型设备模板
- [ ] AC6: Polling Groups Settings 适配 MQTT 推送模式（Expected Update Interval + Timeout）
- [ ] AC7: 设备 Dashboard 显示 MQTT 连接状态（Connected/Disconnected/Error）
- [ ] AC8: MQTT 监控不影响现有 SNMP/Modbus/BACnet 设备监控性能
- [ ] AC9: 与 Probe Server MQTT 实现功能对等（Feature Parity）

---

## Scope

**In scope**:
- MQTT Protocol Handler 移植至 RDGC（代码复用/适配）
- MQTT 协议配置面板 UI（参考 SNMP 面板布局）
- MQTT 设备 Monitoring Templates 支持
- Polling Groups 适配（推送模式 → Expected Update Interval）
- TLS 加密连接（TLS 1.2+）
- 认证方式：Username/Password、Client Certificate (mTLS)
- Topic 订阅与消息接收
- JSON payload 解析（复用 Probe Server 解析逻辑）
- 设备连接状态显示
- 与现有 RDGC 监控管道集成（Graphs、Alarms、Traps）
- 设备发现集成（BDCSPM-6541 multi-protocol auto discovery）

**Out of scope**:
- MQTT 消息发布（设备控制）
- MQTT Broker 托管
- 非 JSON payload 格式（Protobuf、MessagePack — 未来迭代）
- Sub-30s 轮询保证
- Probe Server 端 MQTT 功能的修改或增强
- 高级分析或 ML 处理

**Assumptions**:
1. ⚠️ ASSUMPTION: Probe Server MQTT Protocol Handler 代码可提取为独立模块复用于 RDGC
2. ⚠️ ASSUMPTION: RDGC 与 Probe Server 共享 Protocol Handler 接口规范（Plugin Interface）
3. ⚠️ ASSUMPTION: RDGC 边缘硬件（ARM/x86 小型设备）可运行 MQTT 客户端库
4. ⚠️ ASSUMPTION: 现有 Monitor Configuration 数据模型可扩展支持 MQTT 参数，无需 Breaking Change
5. ⚠️ ASSUMPTION: Monitoring Templates 引擎在 RDGC 上与 Probe Server 实现一致 — [TBD — confirm with architecture team]

---

## Dependencies & Risks

| Item | Type | Owner | Mitigation | Status |
|---|---|---|---|---|
| BDCSPM-6541: Multi-protocol auto discovery | Dep | TBD | 协调 MQTT 设备发现机制 | Ready for Development |
| Probe Server MQTT 代码模块化程度 | Risk | Architecture | 评估代码耦合度；必要时重构为共享库 | ⚠️ NEEDS VALIDATION |
| RDGC 边缘硬件资源限制 | Risk | QA | 性能测试在目标硬件上验证 | Open |
| Probe Server 与 RDGC Protocol Handler 接口差异 | Risk | Dev team | 抽象统一接口层；适配器模式 | Open |
| MQTT 客户端库跨平台兼容性 (ARM/x86) | Risk | Dev team | 确认 Paho C 库支持目标平台 | Open |
| Monitor Configuration 数据模型兼容性 | Risk | Architecture | Schema migration 确保向后兼容 | Open |
| Polling Groups 语义适配（Poll → Push） | Risk | PO + Dev | 定义 "Expected Update Interval" 语义；与 QA 确认告警行为 | Open |

---

## Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook 更新（RDGC MQTT 模块部署步骤）
- [ ] Sizing guide 更新（RDGC 边缘硬件 + MQTT 负载基准）
- [ ] Firewall/port 文档更新（8883/TLS, 1883/非加密）
- [ ] Backup/restore 流程更新（MQTT 配置数据）
- [ ] Release notes 条目
- [ ] Lifecycle Services training 更新（MQTT 配置流程培训）
- [ ] 用户手册更新（MQTT 设备添加与监控配置）

---

## Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | 提取 Probe Server MQTT Protocol Handler 为共享模块 | 8 | Sprint 1 |
| S2 | RDGC Protocol Handler 插件注册 MQTT | 5 | Sprint 1 |
| S3 | MQTT 协议配置面板 UI（Monitor Config） | 8 | Sprint 2 |
| S4 | MQTT 连接管理（TLS + Auth + Reconnect） | 8 | Sprint 2 |
| S5 | Topic 订阅与消息接收管道 | 5 | Sprint 3 |
| S6 | JSON Payload 解析与参数映射（复用 Probe 逻辑） | 5 | Sprint 3 |
| S7 | Monitoring Templates 支持 MQTT 设备类型 | 5 | Sprint 3 |
| S8 | Polling Groups 适配（Expected Update Interval） | 5 | Sprint 4 |
| S9 | 设备连接状态 & 通信健康显示 | 5 | Sprint 4 |
| S10 | 与现有监控管道集成（Graph/Alarm/Trap） | 8 | Sprint 5 |
| S11 | 性能验证 & 协议共存测试（RDGC 硬件） | 5 | Sprint 5 |
| S12 | 端到端集成测试 & Probe Server Feature Parity 验证 | 5 | Sprint 6 |

**总估算**: 72 Story Points

---

## Story Details

---

### STORY: S1 — 提取 Probe Server MQTT Protocol Handler 为共享模块

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (共享组件)

**Description**
As a Developer,
I want the Probe Server's MQTT Protocol Handler to be extracted into a reusable shared module,
So that RDGC can leverage the same battle-tested implementation without code duplication.

**Context & notes**:
- 分析 Probe Server MQTT Handler 当前代码结构和依赖
- 识别与 Probe Server 基础设施的耦合点（日志、配置、连接池等）
- 提取为独立库/模块，定义清晰的 Protocol Handler Interface
- 保持 Probe Server 原有功能不变（回归测试）

**Acceptance Criteria**

AC1:
  Given: Probe Server MQTT Protocol Handler 代码被提取为独立模块
  When: Probe Server 引用新模块构建和运行
  Then: Probe Server MQTT 功能与提取前完全一致（回归测试通过）

AC2:
  Given: 独立 MQTT Handler 模块
  When: 查看模块依赖
  Then: 不直接依赖 Probe Server 特有的基础设施（通过接口抽象）

AC3:
  Given: 独立模块接口文档
  When: RDGC 团队评审
  Then: 接口满足 RDGC 集成需求，无阻塞性问题

**Tasks**
- [ ] Task 1: 分析 Probe Server MQTT Handler 代码结构和依赖图
- [ ] Task 2: 定义 Protocol Handler Interface（抽象层）
- [ ] Task 3: 重构 MQTT Handler 为独立模块/库
- [ ] Task 4: 替换 Probe Server 中原有代码为模块引用
- [ ] Task 5: 执行 Probe Server 回归测试
- [ ] Task 6: 编写模块 API 文档和集成指南

**Definition of Ready checklist**
- [ ] Probe Server MQTT 源码可访问
- [ ] Architecture team 同意模块化方案
- [ ] 回归测试集准备就绪
- [ ] Estimated by team

---

### STORY: S2 — RDGC Protocol Handler 插件注册 MQTT

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a Developer,
I want to register the MQTT Protocol Handler in RDGC's plugin architecture,
So that MQTT appears as an available protocol option in the Monitor Configuration.

**Context & notes**:
- RDGC 已有 Protocol Handler 注册机制（用于 SNMP、Modbus 等）
- 将 S1 产出的共享 MQTT 模块注册为 RDGC Protocol Plugin
- 注册后 MQTT 应出现在 Monitor Config → Protocol List 中
- 实现 RDGC 特有的适配器（Adapter）连接共享模块与 RDGC 运行时

**Acceptance Criteria**

AC1:
  Given: MQTT Protocol Handler 模块已集成到 RDGC 构建中
  When: RDGC 服务启动
  Then: MQTT 出现在可用协议列表中（与 SNMP、Modbus 并列）

AC2:
  Given: RDGC 加载 MQTT Protocol Handler
  When: 查看系统日志
  Then: 日志显示 "MQTT Protocol Handler registered successfully"

AC3:
  Given: MQTT Handler 注册在 RDGC 中
  When: 在 Monitor Configuration UI 选择 MQTT
  Then: 协议特有的配置面板被激活（即使内容尚为空壳）

**Tasks**
- [ ] Task 1: 在 RDGC Protocol Plugin Registry 中注册 MQTT Handler
- [ ] Task 2: 实现 RDGC → 共享模块的 Adapter 层
- [ ] Task 3: 配置 MQTT Handler 启动/停止生命周期
- [ ] Task 4: 验证协议列表 UI 显示 MQTT 选项
- [ ] Task 5: 编写单元测试

**Definition of Ready checklist**
- [ ] S1 模块化完成
- [ ] RDGC Plugin Interface 文档确认
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S3 — MQTT 协议配置面板 UI（Monitor Config）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an IT/OT Administrator,
I want an MQTT protocol configuration panel in the Monitor Configuration page,
So that I can configure MQTT connection parameters for a device (similar to the existing SNMP panel).

**Context & notes**:
- 参考 Probe Server 已有的 SNMP 面板布局（Port、Protocol、Version、Community、Security Level 等）
- MQTT 面板参数：
  - Broker Host / Port (默认 8883)
  - Client ID
  - Protocol Version (v3.1.1 / v5.0)
  - QoS Level (0 / 1)
  - Topic Pattern(s)
  - Auth Type (None / Username+Password / Client Certificate)
  - Username / Password (masked)
  - TLS Enabled (toggle)
  - CA Certificate (file upload or path)
  - Keep Alive Interval (seconds)
- 面板出现在 Monitor Config 右侧，选择 MQTT 协议后显示
- "Verify" 按钮测试连接（与现有 Verify 按钮行为一致）

**Acceptance Criteria**

AC1:
  Given: 管理员在 Monitor Configuration 中选择 MQTT 协议
  When: 协议选中
  Then: 右侧显示 MQTT 配置面板，包含所有必填参数字段

AC2:
  Given: MQTT 配置面板中填入有效参数
  When: 点击 "Verify" 按钮
  Then: 系统测试 MQTT 连接并显示成功/失败结果

AC3:
  Given: MQTT 配置填写完成
  When: 点击 "Submit" 保存
  Then: 配置持久化；Password 字段加密存储，UI 显示 "•••••••"

AC4:
  Given: 已保存的 MQTT 配置
  When: 管理员重新打开 Monitor Configuration
  Then: 所有参数正确回显（密码字段显示掩码）

**Tasks**
- [ ] Task 1: 实现 MQTT 配置面板 UI 组件（参考 SNMP 面板布局）
- [ ] Task 2: 实现面板字段：Broker Host、Port、Client ID、Protocol Version
- [ ] Task 3: 实现面板字段：QoS、Topic Pattern(s)（支持多 Topic）
- [ ] Task 4: 实现认证配置：Auth Type selector + Username/Password + Certificate
- [ ] Task 5: 实现 TLS toggle + CA Certificate 配置
- [ ] Task 6: 实现 "Verify" 按钮连接测试
- [ ] Task 7: 实现配置持久化 API 调用
- [ ] Task 8: 编写 UI 组件测试
- [ ] Task 9: 编写 E2E 测试

**Definition of Ready checklist**
- [ ] UX 确认面板布局（参考 SNMP 面板）
- [ ] 配置 API 接口定义完成
- [ ] RBAC 角色确认（admin 可编辑）
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S4 — MQTT 连接管理（TLS + Auth + Reconnect）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to establish and maintain secure MQTT connections with auto-reconnect,
So that device monitoring is reliable and secure.

**Context & notes**:
- 复用 Probe Server 已有的连接管理逻辑
- 适配 RDGC 运行环境（可能资源更受限）
- TLS 1.2+、Username/Password、mTLS
- 断线自动重连（指数退避）
- 连接状态对接内部状态总线

**Acceptance Criteria**

AC1:
  Given: MQTT 配置已保存且设备 "Monitored" 开关打开
  When: RDGC 启动连接
  Then: TLS 加密连接建立成功，CONNACK 收到

AC2:
  Given: 活跃的 MQTT 连接
  When: Broker 重启或网络中断
  Then: RDGC 在 30s 内尝试首次重连；使用指数退避（最大 5 分钟）

AC3:
  Given: 重连成功后
  When: 连接恢复
  Then: 之前所有 Topic 订阅自动恢复

AC4:
  Given: 无效凭证配置
  When: RDGC 尝试连接
  Then: 连接被拒；错误日志记录；不使用相同凭证重试

**Tasks**
- [ ] Task 1: 集成共享 MQTT 模块的连接管理器到 RDGC
- [ ] Task 2: 适配 RDGC TLS/证书存储
- [ ] Task 3: 实现重连策略（指数退避，可配置）
- [ ] Task 4: 实现连接状态发布至 RDGC 内部消息总线
- [ ] Task 5: 实现与 "Monitored" 开关的联动（开=连接，关=断开）
- [ ] Task 6: 编写集成测试（正常连接、断线重连、无效凭证）

**Definition of Ready checklist**
- [ ] S1 + S2 完成
- [ ] RDGC 证书存储 API 文档
- [ ] 测试 Broker 环境就绪
- [ ] Estimated by team

---

### STORY: S5 — Topic 订阅与消息接收管道

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a System Integrator,
I want RDGC to subscribe to configured MQTT topics and receive messages through the data pipeline,
So that device telemetry flows into the monitoring system.

**Context & notes**:
- Topic 配置来源于 S3 的 Monitor Configuration
- 支持通配符 (+, #)
- 消息进入 RDGC 内部数据管道
- QoS 0 和 QoS 1 支持

**Acceptance Criteria**

AC1:
  Given: MQTT 连接已建立
  When: 配置的 Topic 订阅执行
  Then: 对应 Topic 的消息被接收并传入数据管道

AC2:
  Given: 通配符 Topic（如 `site/+/telemetry`）已配置
  When: 匹配的消息发布
  Then: 所有匹配消息均被接收

AC3:
  Given: Topic 订阅失败（ACL 拒绝等）
  When: SUBACK 返回错误码
  Then: 错误记录日志；设备状态反映订阅失败

**Tasks**
- [ ] Task 1: 实现 Topic 订阅管理器（从 Monitor Config 读取 Topic 列表）
- [ ] Task 2: 实现消息接收与内部队列路由
- [ ] Task 3: 支持通配符 Topic
- [ ] Task 4: 实现 SUBACK 错误处理
- [ ] Task 5: 编写集成测试

**Definition of Ready checklist**
- [ ] S4 连接管理完成
- [ ] 数据管道接口定义确认
- [ ] Estimated by team

---

### STORY: S6 — JSON Payload 解析与参数映射（复用 Probe 逻辑）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to parse MQTT JSON payloads and map them to device monitoring parameters,
So that device telemetry data is stored in the standard parameter model.

**Context & notes**:
- 复用 Probe Server 已有的 JSON 解析逻辑
- 支持 JSONPath 表达式提取嵌套值
- 映射到 RDGC 参数模型（name、value、unit、timestamp、quality）
- 错误 payload 不影响其他设备

**Acceptance Criteria**

AC1:
  Given: MQTT 消息包含 JSON payload `{"temperature": 23.5, "humidity": 45}`
  When: 解析器处理该消息
  Then: 参数 "temperature" = 23.5、"humidity" = 45 被正确提取和存储

AC2:
  Given: 配置了 JSONPath 映射规则
  When: 嵌套 JSON payload 到达
  Then: 按映射规则正确提取目标值

AC3:
  Given: 格式错误的 JSON payload
  When: 解析失败
  Then: 错误记录日志；消息跳过；其他设备数据处理不受影响

**Tasks**
- [ ] Task 1: 集成共享模块中的 JSON 解析器
- [ ] Task 2: 适配 RDGC 参数模型
- [ ] Task 3: 实现 JSONPath 映射配置
- [ ] Task 4: 实现错误隔离（per-device 处理）
- [ ] Task 5: 编写单元测试（多种 payload 格式）

**Definition of Ready checklist**
- [ ] 目标设备的样本 JSON payload 收集
- [ ] Probe Server 解析逻辑文档化
- [ ] Estimated by team

---

### STORY: S7 — Monitoring Templates 支持 MQTT 设备类型

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want to use Monitoring Templates to define monitoring points for MQTT devices,
So that I can quickly onboard new MQTT devices using pre-configured templates (same workflow as SNMP/Modbus).

**Context & notes**:
- Monitoring Templates 是 Probe Server / RDGC 已有功能（见截图 "Monitoring Templates" tab）
- 为 MQTT 设备定义 Template：Topic Pattern + Parameter Mappings + 预期更新频率
- 支持 Template 分配给 Device Type
- 管理员可创建自定义 MQTT Template

**Acceptance Criteria**

AC1:
  Given: 管理员在 Monitoring Templates 中创建 MQTT 类型模板
  When: 定义 Topic Pattern 和参数映射
  Then: Template 保存成功且可分配给 MQTT 设备

AC2:
  Given: MQTT Template 已关联到设备
  When: 该设备的 MQTT 消息到达
  Then: 按 Template 定义的映射规则提取和存储参数

AC3:
  Given: 一个 MQTT Template 关联多个同类设备
  When: 所有设备上线
  Then: 所有设备使用相同映射规则，数据正确采集

**Tasks**
- [ ] Task 1: 扩展 Monitoring Template 引擎支持 MQTT 协议类型
- [ ] Task 2: 实现 MQTT Template 配置字段（Topic、JSONPath Mapping）
- [ ] Task 3: 实现 Template 与设备的关联逻辑
- [ ] Task 4: 在 Monitoring Templates UI Tab 中支持 MQTT 模板创建/编辑
- [ ] Task 5: 编写集成测试

**Definition of Ready checklist**
- [ ] Monitoring Templates 现有架构文档确认
- [ ] MQTT Template 字段定义确认
- [ ] Estimated by team

---

### STORY: S8 — Polling Groups 适配（Expected Update Interval）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want to configure the expected update interval for MQTT devices using the familiar Polling Groups Settings,
So that the system can detect communication loss based on message arrival patterns.

**Context & notes**:
- Probe Server 的 Polling Groups Settings（见截图）：Baseline Interval、High/Medium/Low Priority 倍率、Retries、Timeout
- MQTT 为推送模式，不适用传统 "Polling"
- 适配策略：Baseline Interval → "Expected Update Interval"（设备预期发送频率）
- Timeout 语义保留：超过 N 倍 Expected Interval 未收到消息 → Communication Lost
- Retries 不适用（推送无重试概念），可隐藏或标记 N/A

**Acceptance Criteria**

AC1:
  Given: MQTT 设备配置了 Expected Update Interval = 60s
  When: 设备持续每 60s 发送数据
  Then: 设备状态保持 "Connected"

AC2:
  Given: Expected Update Interval = 60s，Timeout = 3x
  When: 超过 180s 未收到消息
  Then: 设备状态变为 "Communication Lost"；告警触发

AC3:
  Given: 管理员查看 MQTT 设备的 Polling Groups Settings
  When: 配置面板显示
  Then: "Baseline Interval" 标签显示为 "Expected Update Interval"；Retries 字段隐藏或置灰

**Tasks**
- [ ] Task 1: 适配 Polling Groups 数据模型（Interval 语义从 Poll 变为 Expected Push）
- [ ] Task 2: 实现超时检测逻辑（N × Expected Interval → Communication Lost）
- [ ] Task 3: 适配 UI 标签和字段可见性（隐藏 Retries）
- [ ] Task 4: 集成告警引擎（Communication Lost 事件）
- [ ] Task 5: 编写单元测试和集成测试

**Definition of Ready checklist**
- [ ] 语义适配方案与 PO 确认
- [ ] 告警引擎集成点确认
- [ ] Estimated by team

---

### STORY: S9 — 设备连接状态 & 通信健康显示

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want to see MQTT device connection status and communication health in the device Dashboard,
So that I can quickly identify devices with connectivity issues.

**Context & notes**:
- 在设备 Dashboard（见截图顶部 Tab）中显示连接状态
- 状态：Connected / Disconnected / Reconnecting / Communication Lost
- 通信健康指标：最后消息时间、重连次数、错误计数
- 与现有 SNMP/Modbus 设备的状态显示方式一致

**Acceptance Criteria**

AC1:
  Given: MQTT 设备正常通信
  When: 查看设备 Dashboard
  Then: 显示 "Connected" 状态（绿色指示）

AC2:
  Given: MQTT 连接断开
  When: 断连事件发生
  Then: 设备状态 10s 内更新为 "Disconnected"（红色指示）

AC3:
  Given: 设备超时未报数据
  When: 超过 Timeout 阈值
  Then: 显示 "Communication Lost"，并显示最后通信时间

AC4:
  Given: 设备 Dashboard 通信面板
  When: 管理员查看详情
  Then: 显示：最后消息时间、重连次数、错误计数、连接时长

**Tasks**
- [ ] Task 1: 实现连接状态 Badge 组件
- [ ] Task 2: 集成到设备 Dashboard 视图
- [ ] Task 3: 实现通信健康详情面板
- [ ] Task 4: 实现实时状态刷新（≤10s）
- [ ] Task 5: 编写 UI 测试

**Definition of Ready checklist**
- [ ] UX mockup 确认
- [ ] 内部状态 API 可用（S4 完成）
- [ ] Estimated by team

---

### STORY: S10 — 与现有监控管道集成（Graph/Alarm/Trap）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want MQTT device data to flow into existing Graphs, Alarm Panel, and Traps workflows,
So that MQTT devices are monitored with the same tools and processes as other devices.

**Context & notes**:
- 参考截图顶部 Tab：Dashboard、Graphs、Ports、Alarm Panel、Traps、Calendar、Attributes、Monitor 等
- MQTT 采集的参数应可在 Graphs 中绘图
- 参数越阈时触发 Alarm（纳入 Alarm Panel）
- 通信异常事件生成 Trap 条目
- 不需要修改现有 Graph/Alarm/Trap 引擎，只需正确输入数据

**Acceptance Criteria**

AC1:
  Given: MQTT 设备参数数据已入库
  When: 管理员为 MQTT 设备配置 Graph
  Then: Graph 正确显示 MQTT 参数的历史和实时趋势

AC2:
  Given: MQTT 参数配置了阈值告警
  When: 参数值越阈
  Then: Alarm Panel 中出现对应告警条目

AC3:
  Given: MQTT 设备通信异常（Communication Lost）
  When: 超时事件触发
  Then: Traps / Events 中记录通信故障事件

AC4:
  Given: MQTT 设备与 SNMP 设备混合
  When: 管理员查看 Alarm Panel
  Then: MQTT 和 SNMP 设备告警统一显示，可按协议类型过滤

**Tasks**
- [ ] Task 1: 验证 MQTT 参数数据格式与 Graph 引擎兼容
- [ ] Task 2: 验证 MQTT 参数与 Alarm 引擎集成（阈值触发）
- [ ] Task 3: 实现通信异常事件 → Trap/Event 记录
- [ ] Task 4: 验证 Alarm Panel 过滤支持协议类型
- [ ] Task 5: 编写端到端集成测试

**Definition of Ready checklist**
- [ ] Graph/Alarm/Trap 引擎输入接口确认
- [ ] 无需修改现有引擎（仅数据集成）确认
- [ ] Estimated by team

---

### STORY: S11 — 性能验证 & 协议共存测试（RDGC 硬件）

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a QA Engineer,
I want to validate MQTT performance and coexistence with other protocols on RDGC target hardware,
So that we confirm the implementation meets NFRs in the production environment.

**Context & notes**:
- RDGC 通常运行在边缘/远程站点硬件（资源受限）
- 测试场景：MQTT + SNMP + Modbus 同时运行
- 验证 CPU、内存、网络在 NFR 范围内
- 验证现有协议不受 MQTT 影响

**Acceptance Criteria**

AC1:
  Given: RDGC 运行 50 MQTT 设备 + 100 SNMP 设备 + 50 Modbus 设备
  When: 测量系统资源
  Then: MQTT 额外 CPU < 10%，额外 RAM < 128 MB

AC2:
  Given: MQTT 运行中
  When: 测量 SNMP/Modbus 轮询准确性
  Then: 轮询间隔偏差 < 5%（与无 MQTT 基线对比）

AC3:
  Given: MQTT Broker 断线/重连风暴
  When: 频繁重连发生
  Then: 其他协议和 Web UI 不受影响

**Tasks**
- [ ] Task 1: 定义性能测试计划和 RDGC 目标硬件配置
- [ ] Task 2: 建立无 MQTT 基线性能数据
- [ ] Task 3: 执行最大 MQTT 负载测试
- [ ] Task 4: 执行混合协议共存测试
- [ ] Task 5: 执行重连风暴测试
- [ ] Task 6: 产出性能测试报告，更新 Sizing Guide

**Definition of Ready checklist**
- [ ] RDGC 目标硬件环境就绪
- [ ] 性能测试工具准备
- [ ] 基线数据采集完成
- [ ] Estimated by team

---

### STORY: S12 — 端到端集成测试 & Feature Parity 验证

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a QA Engineer,
I want to verify RDGC MQTT functionality matches Probe Server's MQTT capability,
So that we confirm feature parity and customer experience consistency.

**Context & notes**:
- 对比 Probe Server 和 RDGC 的 MQTT 功能矩阵
- 端到端场景：设备配置 → MQTT 连接 → 数据采集 → Graph/Alarm → 断线恢复
- 覆盖所有 Jira 中定义的 Test Cases (TC-MQTT-01 ~ TC-MQTT-07)
- 输出 Feature Parity 对比报告

**Acceptance Criteria**

AC1:
  Given: Probe Server 与 RDGC 配置相同 MQTT 设备
  When: 对比功能行为
  Then: 所有功能项表现一致（Feature Parity 达成）

AC2:
  Given: RDGC MQTT 端到端测试
  When: 执行 TC-MQTT-01 到 TC-MQTT-07
  Then: 所有测试场景 PASS

AC3:
  Given: Feature Parity 对比中发现差异
  When: 差异项被记录
  Then: 产出 Gap Analysis 报告，标注是 "设计差异" 还是 "实现缺陷"

**Tasks**
- [ ] Task 1: 编写 RDGC MQTT Feature Parity 对比矩阵
- [ ] Task 2: 执行端到端功能测试（设备全生命周期）
- [ ] Task 3: 执行 Jira 定义的 TC-MQTT-01 ~ TC-MQTT-07
- [ ] Task 4: 对比 Probe Server 行为差异
- [ ] Task 5: 产出测试报告和 Gap Analysis

**Definition of Ready checklist**
- [ ] S1 ~ S11 全部完成
- [ ] Probe Server 功能基线文档化
- [ ] 测试环境（RDGC + Probe Server 平行）就绪
- [ ] Estimated by team

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (TLS 1.2+, 30s reconnect, 50 devices, 200 topics, <10% CPU, <128MB RAM)
- [x] BLSS product explicitly identified (Distributed IT Performance Management — RDGC)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (Probe Server shared module)
- [x] On-prem operational impact assessed (new module, port 8883, config backup)
- [x] Dependencies identified (BDCSPM-6541, Probe Server code modularity)
- [x] No invented features — based on existing Probe Server implementation
- [x] Terminology matches existing UI (Monitor Config, Polling Groups, Monitoring Templates)
- [x] Placeholders marked with [TBD] where applicable
- [x] Operational deliverables included
- [x] Leverages existing Probe Server implementation to reduce risk and effort
