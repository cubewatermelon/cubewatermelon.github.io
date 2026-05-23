# 知识图谱基本知识与创建过程

经典知识图谱的构建方式是先创建本体，再依据这个本体填充实例，这样生成的知识图谱较为稳定。

但是像 LightRAG 不依赖于一开始的人工本体构建，更简易，但可能存在不稳定的问题。

## 一、知识图谱基本知识

### 1. 什么是本体？

**本体 Ontology**，在知识工程里可以理解为：

> 对某个领域中"有哪些概念、概念之间有什么关系、每类概念有哪些属性、需要遵守什么约束"的形式化定义。

换成人话说，本体就是先回答这些问题：

- 这个领域里有哪些"东西"？
- 这些东西分成哪些类别？
- 这些类别之间是什么关系？
- 每类东西有哪些字段？
- 哪些关系是允许的？
- 哪些规则是必须满足的？

比如在光网络领域，如果我们要建设一个传统知识图谱，就不会一开始直接往里面塞文本，而是先设计：

**实体类型：**

- 网元 NetworkElement
- 板卡 Board
- 端口 Port
- 光放大器 Amplifier
- WSS
- 波长 Wavelength
- 业务 Service
- 告警 Alarm
- 故障 Fault

**关系类型：**

- 网元 → 包含 → 板卡
- 板卡 → 包含 → 端口
- 端口 → 承载 → 波长
- 波长 → 承载 → 业务
- 告警 → 发生于 → 网元/板卡/端口

**这套"类型、关系、属性、约束"的定义，就是本体。**

---

### 2. 本体和知识图谱是什么关系？

可以这样区分：

- **本体** = 模型层 / Schema 层 / 设计规范
- **知识图谱** = 数据层 / 实例层 / 具体知识

例如本体中定义：

```
实体类型：板卡 Board
属性：板卡名称、槽位号、板卡类型、所属网元
关系：板卡 属于 网元
```

知识图谱里填入具体实例：

```
Board_01
  类型：OTU业务板
  槽位：Slot 3
  所属网元：NE_A

Board_01 ──属于── NE_A
```

再比如本体中定义：

```
故障 Fault 可以导致 告警 Alarm
```

图谱实例里填入：

```
低输入光功率 ──导致── LOS告警
WSS衰减配置异常 ──导致── 波长功率异常
EDFA增益异常 ──导致── OSNR下降
```

所以，**本体规定"能表达什么"，知识图谱记录"具体发生了什么/具体有哪些知识"。**

---

### 3. 为什么传统知识图谱要先设计本体？

因为传统知识图谱强调几个目标：

- 结构稳定
- 语义清晰
- 可验证
- 可推理
- 可治理
- 可长期维护

如果没有本体，大家随便抽实体关系，就可能变成：

```
GNN ──用于── RSA
图神经网络 ──应用于── 路由频谱分配
Graph Neural Network ──solves── Routing and Spectrum Assignment
```

这些表达语义类似，但名字、关系、方向都不同，后续很难检索、统计、推理。

有了本体，就可以统一成：

```
Algorithm: GNN
Task: RSA
Relation: used_for
```

这样后续就可以稳定查询：

- 查询所有用于 RSA 的算法
- 查询所有会导致 LOS 告警的故障原因
- 查询某个 MML 命令可以操作哪些对象
- 查询某类板卡支持哪些业务能力

> 需要注意的是，这套构建知识图谱的过程类似类、实例、继承等代码规则。
> 实体类似于一个类，会包含各种属性。

比如：

- "ROADM" 可以是一个**类**
- "北京站点的 NE_001" 是一个**具体实例**

**继承的意义**是：子类继承父类属性和约束。

例如：

```
OpticalDevice
  ├── NetworkElement
  ├── Board
  ├── Port
  ├── Amplifier
  └── WSS
```

**约束**是本体中非常重要但经常被忽视的一部分。它规定什么关系是合法的、属性取值是什么范围。

例如：

```
contains 关系：
domain = NetworkElement
range = Board
```

意思是：

```
NetworkElement 可以 contains Board
但 Alarm 不应该 contains Board
```

有了约束，图谱质量才能被校验。

**规则**比约束更进一步，可以用于推理。

例如：

```
如果某业务经过的所有波长都出现低功率，
且上游站点输出正常，
且本站输入异常，
则可能存在跨段光纤衰耗异常。
```

简化成规则：

```
LowInputPower(Port_A)
AND NormalOutputPower(Upstream_Port)
AND SameWavelength(λ_i)
→ SuspectedFiberLoss(Span_X)
```

---

## 二、知识图谱创建过程

下面详细展开。

### 第一步：需求定义

传统知识图谱不是为了"存所有知识"，而是围绕业务问题建设。

比如你的光网络场景，要先明确目标：

- 目标一：支撑 MML 命令智能查询
- 目标二：支撑故障根因定位
- 目标三：支撑设备对象关系管理
- 目标四：支撑论文技术调研

不同目标决定本体怎么设计。

例如，如果目标是 MML 智能调用，本体重点是：

- MMLCommand
- CommandParameter
- ManagedObject
- Board
- Port
- Precondition
- Postcondition
- RiskLevel
- RollbackCommand

如果目标是故障诊断，本体重点是：

- Alarm
- Fault
- RootCause
- Symptom
- NetworkElement
- PerformanceMetric
- Location
- ServiceImpact
- DiagnosisRule

所以本体设计不是越全越好，而是要服务业务查询和推理。

---

### 第二步：本体设计

本体设计通常要回答 5 个问题：

#### 问题一：有哪些实体类型？

以光网络故障诊断为例：

- NetworkElement
- Board
- Port
- OpticalChannel
- Wavelength
- Alarm
- PerformanceMetric
- FaultSymptom
- RootCause
- DiagnosisCase
- MMLCommand

#### 问题二：实体之间有哪些关系？

- NetworkElement → contains → Board
- Board → contains → Port
- Port → carries → Wavelength
- Alarm → occurs_on → Port
- PerformanceMetric → measured_on → Port
- RootCause → causes → Alarm
- DiagnosisCase → has_symptom → FaultSymptom
- MMLCommand → queries → NetworkElement
- MMLCommand → configures → Port

#### 问题三：每类实体有哪些属性？

Alarm:

- alarm_id
- alarm_name
- severity
- timestamp
- location
- probable_cause

Port:

- port_id
- board_id
- port_type
- direction
- status

PerformanceMetric:

- metric_name
- value
- unit
- timestamp

#### 问题四：有哪些约束？

属性约束：

```
Alarm.severity ∈ {CR, MJ, MN, WR}
Port.direction ∈ {input, output}
Power.unit = dBm
```

关系约束：

```
occurs_on:
  domain = Alarm
  range = OpticalDevice

contains:
  domain = NetworkElement
  range = Board
```

#### 问题五：有哪些推理规则？

```
如果某站点所有波长输入功率同步下降，
则优先怀疑线路段衰耗或光纤问题。

如果单波长异常而其他波长正常，
则优先怀疑 WSS 单波衰减、波长配置或通道级问题。
```

---

### 第三步：数据源梳理

传统知识图谱要有稳定数据源。常见来源包括：

**结构化数据：**

- 资源台账
- 网管配置数据
- 告警数据库
- 性能数据库

**半结构化数据：**

- XML
- YAML
- MML命令输出
- 设备配置文件

**非结构化数据：**

- PDF手册
- Word文档
- 运维案例
- 论文
- 故障报告
- 专家经验

对于企业知识图谱，结构化和半结构化数据通常更可靠；非结构化文档需要 NLP 或 LLM 辅助抽取。

---

### 第四步：实体抽取

实体抽取就是从数据中识别出具体对象。

例如从 MML 输出里抽：

```
NE: NE_001
Board: Slot_3_OTU
Port: 3-1-1
Wavelength: λ39
Alarm: LOS
```

从论文中抽：

```
Algorithm: GNN
Algorithm: PPO
Task: RSA
Metric: Blocking Probability
Dataset: NSFNET
```

传统方式一般包括：

- 规则抽取
- 词典匹配
- 正则表达式
- 机器学习模型
- 人工标注校验

例如光网络里很多对象有固定格式，很适合规则抽取：

```
端口：3-1-1
槽位：Slot 3
波长：λ39 / 193.1 THz
功率：-12.3 dBm
告警：LOS / LOF / BDI / ODU-AIS
```

---

### 第五步：关系抽取

关系抽取是把实体之间连起来。

例如从配置数据里得到：

```
NE_001 contains Board_03
Board_03 contains Port_3_1_1
Port_3_1_1 carries Wavelength_39
```

从告警数据里得到：

```
Alarm_001 occurs_on Port_3_1_1
Alarm_001 has_severity Critical
```

从专家规则里得到：

```
LowInputPower causes LOS
WSS_Attenuation_Error causes Single_Wavelength_Power_Abnormal
```

传统关系抽取可以来自：

- 数据库外键
- 配置文件层级
- 规则模板
- 人工维护
- NLP模型
- 专家标注

企业图谱里最可靠的关系通常来自已有系统数据，例如资源拓扑、告警表、业务路径表，而不是纯文本自动抽取。

---

### 第六步：实体对齐与消歧

这是传统知识图谱非常关键的一步。

**同一个对象可能有多个名字：**

```
NE001
NE_001
北京东站NE1
Beijing-East-NE-1
```

它们可能其实是同一个网元。

**同一个概念也可能有多个表达：**

```
GNN
Graph Neural Network
图神经网络
```

实体对齐就是把它们归一到一个标准实体。

例如：

```
Graph Neural Network
图神经网络
GNN
→ Algorithm:GNN
```

**消歧**则是区分同名不同物。

例如 `LOS` 可能是：

- Loss of Signal 告警
- 在别的领域表示 Line of Sight

在光网络场景中必须根据上下文判断。

传统方法通常包括：

- 唯一ID
- 标准命名规范
- 别名表
- 规则匹配
- 相似度匹配
- 人工审核

---

### 第七步：图谱入库

#### 属性图 Property Graph

这是工程中很常用的图数据库形式，例如 Neo4j。

节点和边都可以有属性。

例如：

```
节点：
(:Port {id:"3-1-1", type:"line", status:"active"})

边：
(:Board)-[:HAS_PORT {source:"inventory"}]->(:Port)
```

查询语言常见是 Cypher。

---

### 第八步：质量校验

传统知识图谱强调质量治理，否则图越大越乱。

常见校验包括：

- **类型校验：** Alarm 是否错误连到了 Paper？
- **关系约束校验：** contains 的主语是否一定是 NetworkElement？
- **属性取值校验：** severity 是否在合法枚举中？
- **重复实体校验：** GNN 和 Graph Neural Network 是否重复？
- **孤立节点检查：** 是否存在没有任何关系的实体？
- **冲突检查：** 同一个端口是否同时属于两个不同板卡？

图谱后续可能支撑自动化决策，错图谱会导致错操作。

---

### 第九步：推理与应用

有了本体、实例和规则后，图谱可以用于推理。

例如本体里定义：

```
OTUBoard 是 Board 的子类
Board 是 OpticalDevice 的子类
```

如果图谱里有：

```
Board_03 rdf:type OTUBoard
```

系统可以推理出：

```
Board_03 也是 Board
Board_03 也是 OpticalDevice
```

再比如：

```
如果 Alarm occurs_on Port
Port belongs_to Board
Board belongs_to NetworkElement
```

可以推理：

```
Alarm impacts NetworkElement
```

在光网络诊断里，推理可以用于：

- 告警传播分析
- 业务影响分析
- 根因候选排序
- 配置一致性检查
- MML命令前置条件校验