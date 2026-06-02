# Reliability Risk Review Skills

> **AI 驱动的器件可靠性评审系统 · AI-Powered Component Reliability Review System**

---

<p align="center">
  <b>中文</b> · <a href="#english-version">English</a>
</p>

---

## 📌 概述 / Overview

本仓库包含一套 **OpenClaw Agent Skills**，用于对智能终端产品的各类器件进行自动化可靠性评审。系统覆盖从品类识别、检查项评估、风险量化、测试方案生成到报告输出的完整五阶段流水线。

This repository contains a set of **OpenClaw Agent Skills** for automated reliability review of smart terminal product components. The system covers a complete five-stage pipeline: category recognition, checklist evaluation, risk quantification, test plan generation, and report output.

---

## 🏗️ 系统架构 / System Architecture

```
用户输入 (器件名称、材料、工艺、规格书)
    │
    ▼
┌─────────────┐    ┌──────────────┐     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Phase 1    │───▶│   Phase 2    │───▶│   Phase 3   │───▶│   Phase 4    │───▶│   Phase 5    │
│  classify   │    │  checklist   │     │  risk-eval   │    │  test-plan   │    │   report     │
│  品类识别    │    │  检查项评估   │     │  风险评估    │    │  测试方案     │    │  报告生成     │
└─────────────┘    └──────────────┘     └──────────────┘    └──────────────┘    └──────────────┘
```

五个 Skill 串联执行，前一阶段的输出作为下一阶段的输入，最终生成标准化的飞书多维表格评审报告。

Five Skills execute in a pipeline where each stage's output feeds into the next, culminating in a standardized Feishu Bitable review report.

---

## 📦 核心 Skills / Core Skills

### 1. `/classify` — 品类自动识别 · Category Auto-Recognition

| 项目 | 说明 |
|------|------|
| **功能** | 根据器件名称、材料、工艺自动识别所属品类 |
| **触发词** | `品类识别` · `classify` · `识别器件` · `这是什么品类` |
| **覆盖品类** | 结构件 · 辅料 · 电声器件 · 光学器件 · 包材 · 安规器件 |

**核心能力 / Core Capabilities**

- **名称关键词匹配** — 内置覆盖 6 大品类的关键词映射表，支持直接命中
- **材料/工艺辅助判定** — 名称未命中时，通过材料（如 PC+GF、铝合金）和工艺（如注塑、CNC、阳极氧化）推断品类
- **多组件复合体识别** — 自动识别如"电池盖+TP+miniLED"等三合一/多合一组件，标注"多品类复合"并在后续流程中对子组件分别处理
- **置信度输出** — 输出 high / medium / low 三档置信度 + 推理依据 + 备选品类

**输入输出示例 / I/O Example**

```json
{
  "category": "结构件",
  "confidence": "high",
  "reasoning": "器件名'电池盖'直接命中结构件关键词，材料复合板和工艺UV转印均为结构件典型特征",
  "alternatives": []
}
```

---

### 2. `/checklist` — 自动 Checklist 评估 · Automated Checklist Evaluation

| 项目 | 说明 |
|------|------|
| **功能** | 加载品类对应的 Checklist 模板，逐项自动评估器件可靠性风险 |
| **触发词** | `checklist评估` · `逐项检查` · `checklist` · `可靠性检查` |
| **覆盖维度** | 器件测试履历 · 材料与工艺 · 表面处理 · 结构强度 · 环境可靠性 · 装配匹配 |

**核心能力 / Core Capabilities**

- **知识库优先加载** — 优先读取 `memory/checklist-templates/<品类>-checklist.md` 模板，无模板时从失效数据库反推生成
- **六维度覆盖** — 每个维度至少 3 项检查，涵盖从历史客诉到装配匹配的全链路
- **多组件复合体扩展** — 自动补充组件间贴合界面、新增连接器、ESD/热设计、供应商协同等维度
- **五级信息源优先级** — 规格书 P0 > 失效数据库 P1 > 工艺对照表 P2 > 规则引擎 P3 > 历史案例 P4 > 工程通用原则 P5
- **三档置信度标注** — 确认（知识库明确匹配）/ 推定（同品类普遍规律）/ 待确认（信息不足）

**输出格式 / Output Format**

每项输出独立评估，不跳项、不合并：

| # | 检查项 | AI判断 | 依据 | 置信度 |
|---|--------|--------|------|--------|
| 1 | 上代机型同位置是否有客诉 | ✅ 建议关注 | 未提供上代机型信息，默认建议追溯 | 待确认 |
| 2 | 复合板层间结合力是否达标 | ✅ 建议关注 | failure-modes-database.md: 转印层弯折裂纹风险 | 确认 |

---

### 3. `/risk-eval` — 风险自动评估 · Automated Risk Evaluation

| 项目 | 说明 |
|------|------|
| **功能** | 失效模式匹配 + S/O/D/RPN 量化评分 + 规则引擎交叉验证 + 安全红线检查 |
| **触发词** | `风险评估` · `risk eval` · `SOD评分` · `RPN计算` · `失效模式匹配` |
| **评分标准** | 可定制企业官方打分机制|

**核心能力 / Core Capabilities**

- **失效模式匹配** — 从知识库检索对应品类的全部失效模式，按材料匹配 / 工艺匹配 / 历史案例匹配 / 通用适用四种方式判断适用性
- **S/O/D 量化评分** — 严格遵循企业官方标准：
  - **S (Severity 严重度)**: 5=无法生产/测试 → 1=用户无法感知
  - **O (Occurrence 频度)**: 5=>20% 极高 → 1=<1/2000 极低
  - **D (Detection 探测度)**: 5=无测试方案 → 1=全部就绪/几乎确定
- **RPN 计算与定级** — `RPN = S × O × D`
  | 等级 | 符号 | 阈值 | 颜色 |
  |------|------|------|------|
  | 致命缺陷 | S | ≥ 125 | 🔴 红色 |
  | 高风险 | A | 75 < RPN < 125 | 🔴 红色 |
  | 中风险 | B | 27 < RPN ≤ 75 | 🟡 黄色 |
  | 低风险 | C | ≤ 27 | 🟢 绿色 |

- **规则引擎交叉验证** — 读取 `memory/rules.md`，将规则结论与 Agent 自主评估交叉比对（一致→确认增强 / Agent有规则无→AI推定 / 规则有Agent无→重新审视补充）
- **安全红线检查** — 五条红线强制检查：人身安全风险 / 认证阻断风险 / 历史重复失效 / 安规器件核心项 / 无测试覆盖

**输出示例 / Output Example**

| # | 失效模式 | 匹配方式 | S | O | D | RPN | 等级 | 来源 |
|---|---------|---------|---|---|---|-----|------|------|
| 1 | 【安全红线】FM-001 涂层附着力不足 | 直接匹配(材料+工艺) | 5 | 4 | 3 | 60 | 中风险(B) | Agent+规则 |

---

### 4. `/test-plan` — 测试方案自动生成 · Automated Test Plan Generation

| 项目 | 说明 |
|------|------|
| **功能** | 基于风险评估结果，自动匹配并生成结构化测试方案；如用户提供已排方案则自动比对输出差异分析 |
| **触发词** | `测试方案` · `test plan` · `生成测试` · `测试计划` |
| **覆盖范围** | RPN ≥ 27 的全部风险项（中风险及以上） |

**核心能力 / Core Capabilities**

- **智能测试检索** — 从失效数据库（拦截测试字段）、测试 SOP、测试项目库三重知识库检索匹配的测试项
- **参数自动填充** — 严格按优先级链填充：规格书要求 > SOP默认参数 > 历史测试参数 > 工程推定
- **样本量推荐** — 致命缺陷(S) ≥ 10pcs，高风险(A) ≥ 5pcs，中风险(B) ≥ 3pcs，低风险(C) ≥ 1pcs
- **去重合并** — 同一测试覆盖多风险时保留最高优先级，可合并试验项并注明覆盖的风险
- **用户方案比对** — 自动比对用户已排方案与 AI 方案，按五种差异类型分类（一致 / 参数差异 / 等效替换 / 用户独有 / 【建议补充】）
- **测试顺序建议** — 非破坏性 → 环境测试 → 破坏性，符合工程最佳实践

**输出格式 / Output Format**

| 优先级 | 测试名称 | 测试参数 | 样品数 | 接收标准 | 关联失效模式 | 标准参考 |
|--------|---------|---------|--------|---------|-------------|---------|
| 关键 | 百格附着力 | 百格刀(1mm), 3M 600#胶带 | 5pcs | ≥4B (≤5%脱落) | FM-001 | GB/T 9286 |

---

### 5. `/report` — 报告生成 · Report Generation

| 项目 | 说明 |
|------|------|
| **功能** | 汇总全部评审结果，生成标准化 6-Sheet xlsx 报告并上传至飞书云空间 |
| **触发词** | `生成报告` · `输出报告` · `评审报告` · `report` |

**六大 Sheet / Six Sheets**

| Sheet | 名称 | 内容 |
|-------|------|------|
| 1 | 基本信息 | 器件名、品类、材料、工艺、供应商、项目名、规格书版本、评审日期、器件本质、结构叠层、新增模具、关键阻断项 |
| 2 | Checklist | 每项检查项的编号(C#)、名称、AI判断(✅/❌)、依据、置信度 |
| 3 | 风险评估 | 失效模式、失效机理、S/O/D 分值及依据(1-5传音标准)、RPN、等级、来源(Agent/规则引擎/Agent+规则) |
| 4 | 差异分析 | AI推定 vs 知识库确认的对照，待确认项清单 |
| 5 | 测试方案 | 测试名称、优先级(P0-P3)、参数、样本量、接收标准、标准参考 |
| 6 | 总体意见 | 关键发现(带🔴/🟠/🟡分级)、高风险统计、结论(建议通过/建议有条件通过/不建议通过)、后续动作清单 |

**格式规范 / Formatting Standards**

- 表头：深蓝底(RGB:68,114,196) + 白字 + 粗体
- 条件格式着色：按风险等级 / 判断结果 / 置信度自动着色
- 冻结首行，列宽自适应（最小12字符），全表细边框
- 风险评估按 RPN 降序排列

**智能信息提取 / Intelligent Information Extraction**

> ⚠️ **核心规则**：用户提供 PPTX/规格书/图纸时，必须主动提取所有可用信息填入报告，禁止留"未提供"。
>
> **核心规则**: When users provide PPTX/specs/drawings, all available information must be proactively extracted into the report. Never leave fields as "not provided."

---

## 🔄 流水线数据流 / Pipeline Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          用户输入 / User Input                          │
│  器件名称 · 材料 · 工艺 · 规格书(PPTX/PDF/图纸) · 项目名称              │
│  Component Name · Material · Process · Spec Document · Project Name     │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    Phase 1: classify   │
                    │   品类识别 · 置信度     │
                    │ Category · Confidence  │
                    └───────────┬───────────┘
                                │ category + confidence
                    ┌───────────▼───────────┐
                    │   Phase 2: checklist   │
                    │ 六维度逐项评估 · 置信度 │
                    │ 6-dimension evaluation │
                    └───────────┬───────────┘
                                │ checklist results
                    ┌───────────▼───────────┐
                    │   Phase 3: risk-eval   │
                    │ S/O/D评分 · RPN · 红线  │
                    │ S/O/D scoring · RPN    │
                    └───────────┬───────────┘
                                │ risk table
                    ┌───────────▼───────────┐
                    │   Phase 4: test-plan   │
                    │ 测试方案 · 用户方案比对  │
                    │ Test plan · Diff analysis│
                    └───────────┬───────────┘
                                │ test plan
                    ┌───────────▼───────────┐
                    │    Phase 5: report     │
                    │ 6-Sheet xlsx · 飞书上传 │
                    │ 6-Sheet xlsx · Upload  │
                    └────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   飞书多维表格链接      │
                    │  Feishu Bitable Link   │
                    └────────────────────────┘
```

---

## 📁 知识库结构 / Knowledge Base Structure

```
memory/
├── rules.md                          # 全局规则库 · Global Rules
├── test-items.md                     # 全局测试项目库 · Global Test Items
├── material-process.md              # 材料-工艺-失效关联 · Material-Process-Failure Mapping
├── standards.md                      # 标准文献库 · Standards Library
│
├── failure-database/
│   ├── INDEX.md                      # 索引 · Index
│   ├── failure-modes-database.md     # 失效模式数据库 · Failure Modes DB
│   ├── testing-sop.md               # 测试SOP · Testing SOP
│   ├── material-catalog.md          # 材料目录 · Material Catalog
│   └── structural-process-table.md  # 结构工艺对照表 · Structural Process Table
│
├── checklist-templates/
│   └── <品类>-checklist.md          # 各品类Checklist模板 · Per-category Checklist Templates
│
├── history/
│   └── <品类>/                      # 历史评审记录 · Historical Review Records
│
└── learned-content/
    ├── review-reports/
    │   ├── failure-patterns.md      # 高频失效模式 · High-frequency Failure Patterns
    │   ├── test-gaps.md            # 测试盲区对照 · Test Gaps
    │   └── process-rules.md        # 流程/标准管理规则 · Process Rules
    │
    └── issue-tracker/
        ├── material-failure-modes.md  # 物料类失效模式 · Material Failure Modes
        ├── root-cause-patterns.md     # 根因模式 · Root Cause Patterns
        ├── test-lessons.md           # 测试经验教训 · Test Lessons Learned
        └── supplier-risks.md         # 供应商风险 · Supplier Risks
```

---

## 🔧 技术栈 / Tech Stack

| 组件 | 说明 | Component | Description |
|------|------|-----------|-------------|
| **OpenClaw** | Agent 运行时 · Agent Runtime | 任务编排与 Skill 路由 | Task orchestration and skill routing |
| **openpyxl** | xlsx 报告生成 · xlsx Generation | 带格式多Sheet Excel | Formatted multi-sheet Excel |
| **飞书 Bitable API** | 报告输出 · Report Output | 多维表格创建与上传 | Bitable creation and upload |
---

## 🚀 使用方式 / Usage

### 自动触发 / Auto-Trigger

当用户消息包含以下任一模式时，系统**自动启动**五阶段评审流水线：

The system **automatically launches** the five-stage review pipeline when a user message contains any of these patterns:

| 触发模式 / Trigger Pattern | 示例 / Example |
|---|---|
| 器件名 + 材料 + 工艺 / Component + Material + Process | `帮我评审XX中框，PC+GF注塑` |
| "评审" + 器件信息 / "Review" + Component Info | `评审扬声器A123，PU+PET，热压成型` |
| "评估" + 可靠性 / "Evaluate" + Reliability | `评估这个电池盖的可靠性，铝合金阳极氧化` |
| "分析" + 器件 / "Analyze" + Component | `分析摄像头模组的失效风险` |
| 提供规格书/图纸 / Provide specs or drawings | `这个器件的规格书在桌面，帮忙看下` |

### 手动触发 / Manual Trigger

各 Skill 支持独立调用（通过触发词），但完整评审建议使用自动流水线：

Each Skill supports independent invocation (via trigger words), but a complete review is recommended via the automatic pipeline:

```
/classify          # 仅品类识别 · Category only
/checklist         # 仅Checklist评估 · Checklist only
/risk-eval         # 仅风险评估 · Risk evaluation only
/test-plan         # 仅测试方案 · Test plan only
/report            # 仅报告生成 · Report generation only
```

---

## 📊 评分标准参考 / Scoring Standards Reference

### RPN 风险等级 / RPN Risk Levels

```
RPN = S × O × D  (企业 1-5 分制 · Transsion 1-5 scale)

  1   5   9   13   17   21   25   27        75        125   150
  │───│───│───│────│────│────│────│─────────│─────────│──────│
  🟢 低风险(C)  │     🟡 中风险(B)      │  🔴 高风险(A)   │ 🔴 致命缺陷(S)
```

### S/O/D 评分速查 / S/O/D Quick Reference

| 分值 | S 严重度 · Severity | O 频度 · Occurrence | D 探测度 · Detection |
|:----:|---|---|---|
| **5** | 无法生产/测试 · Cannot produce/test | >20% 极高 · Very High | 无测试方案 · No test plan |
| **4** | 严重影响功能 · Severe impact | >1% ~ 20% 高 · High | 方案基本制定 · Plan drafted |
| **3** | 有缺陷但可继续 · Defective but workable | >1/500 ~ 1% 中 · Medium | 方案完成 · Plan ready |
| **2** | 外观类/非功能 · Cosmetic/Non-functional | >1/2000 ~ 1/500 低 · Low | 设备人员就绪 · Ready |
| **1** | 用户无法感知 · User cannot perceive | <1/2000 极低 · Very Low | 全部就绪/几乎确定 · Fully ready |

---

## 📝 示例输出 / Example Output

以下为评审报告六大 Sheet 的实际输出示例：

Below are examples of actual output from the six-sheet review report:

### Sheet 1: 基本信息 · Basic Information

| 字段 | 内容 |
|------|------|
| 器件名称 | 敲击流光共鸣三合一组件（电池盖+TP+miniLED） |
| 器件品类 | 结构件/电声器件/光学器件（多品类复合） |
| 材料 | PC/PMMA复合盖板 + PET薄膜(TP sensor) + miniLED + MCU/Driver IC |
| 工艺 | TP贴合+miniLED贴合+Deco CNC+辅料贴合 |
| 评审日期 | 2026-06-01 |
| 器件本质 | 不是普通电池盖，而是三合一组件 |

### Sheet 3: 风险评估 · Risk Assessment (节选)

| # | 失效模式 | S | O | D | RPN | 等级 |
|---|---------|---|---|---|-----|------|
| R1 | 【安全红线】BTB连接器无固定导致跌落失效 | 5 | 4 | 3 | 60 | 中风险(B) |
| R2 | 涂层附着力不足 | 4 | 3 | 2 | 24 | 低风险(C) |
| R3 | TP贴合气泡导致触控失灵 | 3 | 3 | 2 | 18 | 低风险(C) |

### Sheet 6: 总体意见 · Overall Opinion (节选)

| # | 关键发现 |
|---|---------|
| 🔴 F1 | **阻断项**: BTB连接器无固定，跌落可靠性无法实现，建议修改设计 |
| 🟠 F2 | **高风险**: 三合一组件涉及多方供应商协同，需建立联合验收标准 |
| 🟡 F3 | **中风险**: 复合板层间结合力需通过弯折+恒温恒湿双重验证 |
| 📊 | **统计**: 致命缺陷(S) 0项 · 高风险(A) 0项 · 中风险(B) 5项 · 低风险(C) 12项 |
| ✅ | **结论**: 建议有条件通过 — 阻断项解决后可进入下一阶段 |

---

## 🔐 安全红线 / Safety Red Lines

以下情况**必须**在评审报告中红色高亮：

The following conditions **must** be highlighted in red in the review report:

| 红线 / Red Line | 触发条件 / Trigger |
|------|---------|
| 人身安全风险 · Personal Safety Risk | 失效模式涉及触电/起火/爆炸/化学泄漏/机械伤人 · Electric shock / fire / explosion / chemical leak / mechanical injury |
| 认证阻断风险 · Certification Blocking Risk | 可能导致 CCC/UL/CE/SRRC/PSE 等强制认证失败 · May cause mandatory certification failure |
| 历史重复失效 · Historical Repeated Failure | 同品类器件在过去评审中 ≥3 次出现相同失效模式 · Same failure mode appeared ≥3 times in past reviews |
| 安规器件核心项 · Safety-Critical Item | 电池类 RPN≥75 且涉及过充/短路/热失控 · Battery class RPN≥75 with overcharge/short-circuit/thermal runaway |
| 无测试覆盖 · No Test Coverage | 某项高风险(RPN>75)无对应测试手段可检出 · High risk(RPN>75) with no corresponding test to detect |

---

## 📖 置信度体系 / Confidence System

| 置信度 · Confidence | 定义 · Definition | 标识 · Marker |
|---|---|---|
| **确认 · Confirmed** | 知识库有明确匹配 + 工程推理支持 · Explicit KB match + engineering reasoning | 🟢 |
| **推定 · Inferred** | 基于同品类/同材料的普遍规律，但未找到直接证据 · Based on common patterns, no direct evidence | 🟡 |
| **待确认 · To Confirm** | 信息不足，需要人工补充 · Insufficient information, needs manual input | 🟠 |

---

## 🛠️ 开发 & 扩展 / Development & Extension

### 添加新品类 Checklist / Adding a New Category Checklist

在 `memory/checklist-templates/` 下创建 `<品类>-checklist.md`：

Create `<category>-checklist.md` under `memory/checklist-templates/`:

```markdown
# <品类> Checklist

## 器件测试履历
- [ ] 上代机型同位置是否有客诉/失效记录
- [ ] 相似品类历史评审是否出现相同失效
- [ ] 当前是否有EVT/DVT测试历史

## 材料与工艺
- [ ] 材料×工艺组合是否有已知失效风险
- [ ] 关键工艺控制点是否可检出
...
```

### 添加新失效模式 / Adding New Failure Modes

在 `memory/failure-database/failure-modes-database.md` 中新增条目，包含：

Add entries to `memory/failure-database/failure-modes-database.md`, including:

- 失效模式名称 · Failure mode name
- 适用品类 · Applicable categories
- 易受影响材料 · Vulnerable materials
- 易受影响工艺 · Vulnerable processes
- 拦截测试 · Intercepting tests
- 历史案例 · Historical cases (optional)

---
---

<p align="center">
  <sub>Built with OpenClaw Agent Skills · 器件可靠性评审 · 2026</sub>
</p>
