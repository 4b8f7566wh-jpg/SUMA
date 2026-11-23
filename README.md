# GATE/01 — 供应商准入评测

门禁优先（gate-first）的供应商准入评测 Web 应用。供应商提交财务、CSR、绿电、BOM 报价与比较条件；系统先执行强制门禁，门禁不通过直接输出 `BLOCKED`，**不进入 BOM 缺口与敏感性分析**；通过后才计算成本结构、证据缺口与敏感性，并由 DeepSeek Flash 增强叙述层。

![首页](docs/screenshots/home.png)

![评测报告](docs/screenshots/report-approved.png)

## 技术栈

- Next.js 14 App Router + TypeScript
- Tailwind CSS + shadcn/ui 风格组件（Radix Slot / Label / Progress / Tabs / Separator）
- react-hook-form + Zod，每个步骤独立 schema
- SheetJS `xlsx`：浏览器端 Open-Book BOM 解析
- Supabase：表单提交、评测结果、Storage 文件上传
- DeepSeek API：`https://api.deepseek.com`，默认模型 `deepseek-flash`

## 快速开始

```bash
pnpm install
cp .env.example .env.local
pnpm dev
```

打开 <http://localhost:3000>。

### 手机 / 局域网预览

`dev` 脚本已绑定 `0.0.0.0`：

```bash
pnpm dev
```

手机与电脑连接同一局域网后，访问：

```text
http://<电脑局域网 IP>:3000
```

Windows 可用 `ipconfig` 查看无线网卡的 IPv4 地址（例如 `192.168.71.2`）。手机端表单使用底部固定操作栏、数字键盘输入与竖向卡片布局。

![手机端 BOM 卡片](docs/screenshots/mobile-bom-card.png)

不配置任何环境变量也能运行：

- 没有 `DEEPSEEK_API_KEY` 时，自动使用确定性规则引擎生成完整报告，并在报告中明确标注“规则引擎叙述”；
- 没有 Supabase 时，评测与文件登记仍在当前会话中完成，但不会持久化；
- 报告在 `sessionStorage` 中保留，可在同一标签页直接刷新 `/report/[id]`。

## 环境变量

```dotenv
DEEPSEEK_API_KEY=
SUPABASE_URL=
SUPABASE_ANON_KEY=

# 可选
DEEPSEEK_MODEL=deepseek-flash
SUPABASE_BUCKET=supplier-evidence
```

密钥只在服务端 Route Handler 中读取，不会进入浏览器 bundle。

## Supabase 配置

1. 新建 Supabase 项目；
2. 在 SQL Editor 执行 [supabase/schema.sql](supabase/schema.sql)；
3. 把 Project URL 与 anon key 写入 `.env.local`；
4. 重新启动 `pnpm dev`。

程序会写入两张表：

- `supplier_submissions`：完整提交快照（JSONB）；
- `evaluation_reports`：结构化报告（JSONB）、决策、引擎版本、AI 模式。

文件通过 `POST /api/upload` 上传到 Storage bucket `supplier-evidence`。SQL 中的 RLS 策略为**演示用匿名策略**，正式环境请改成登录态 + 行级隔离，或由服务端使用 `service_role` key。

## 门禁与评测逻辑

Zod 是唯一数据真相，前后端共用 [lib/schemas.ts](lib/schemas.ts)。每个步骤有独立 schema：

| 步骤 | Schema | 关键约束 |
| --- | --- | --- |
| 1 财务与主体 | `financialStepSchema` | 连续 3 年、金额与录入方式 |
| 2 CSR + 绿电 | `csrGreenStepSchema` | CSR 四项 0–100、绿电 ≤ 总用电 |
| 3 BOM | `bomStepSchema` | 行级 evidence、材料毛净重、利润基准 |
| 4 比较条件 | `comparisonStepSchema` | 枚举条件 + `_form_meta` |
| 服务端 | `fullSubmissionSchema` | 最终入口校验 |

阈值集中在 [lib/engine-shared.ts](lib/engine-shared.ts)：

- 财务：连续 3 年；最新年度净利润 > 0 为**阻断**；累计亏损、OCF/DSO/负债增速为警示；
- CSR：四项单项 ≥ 60 为**阻断**；均值 ≥ 70 为目标；
- 绿电：占比 < 10% 为**阻断**；≥ 20% 为目标；未披露降为条件准入；
- BOM：可核验金额覆盖 ≥ 70%、材料未验证占比 ≤ 30%、利润基准已声明、材料毛净重/废料率齐全，均为**阻断**；
- 商务条件：产能、PPM、CPK/FMEA、变更响应、材料占比一致性，只做条件警示，不阻断。

决策规则：

```
任一 blocking gate FAIL  → BLOCKED（跳过 BOM_GAP / SENSITIVITY）
否则有 WARN / 非阻断 FAIL / 比较矛盾 → CONDITIONAL
否则 → APPROVED
```

比较条件**不做加权评分**，只并列呈现申报值与可推导值，并标注“一致 / 分歧 / 无法核验”。

## Excel BOM 自动解析

BOM 步骤内置 `ExcelParser`，使用 SheetJS 在浏览器端解析供应商 Open-Book 报价 Excel：

- 支持 `.xlsx` / `.xls`、多 Sheet、多级表头、合并单元格；
- 表头扫描容忍前 5–10 行 Logo / 说明 / 空白；
- 自动识别原材料、制造费用、工装、物流、包装与利润率；
- 每个原材料行 → 一条 `material`；每个工序 → 一条 `machine_burden`（保留节拍、人工费率、设备费率，金额优先取 FGK Total）；
- 工装总价 ÷ 工装寿命 → `tooling_amortization`；物流 / 包装 → 对应行；利润率或利润金额 → `profit`；
- 解析结果先预览，不直接提交；点击“应用到 BOM 表单”后才写入 RHF State；
- 所有导入行默认 `evidence_type = "erp_export"`，可在 BOM 编辑器中修改；
- 找不到列或字段时，显示“以下数据需手动补充 / 复核”，并保留空值让供应商填写；
- 可替换当前 BOM 或追加，追加时利润行以解析结果为准，避免重复利润行。

![Excel 自动解析](docs/screenshots/excel-parser.png)

核心文件：

- [ExcelParser.tsx](components/evaluation-form/ExcelParser.tsx) — 拖拽上传、Sheet 切换、预览与写入交互；
- [parse-bom-excel.ts](lib/excel/parse-bom-excel.ts) — `parseBomExcel()` 表头识别、行遍历与字段映射；
- [bom-step.tsx](components/evaluation-form/bom-step.tsx) — 解析结果写入 `useFieldArray` 的 BOM State。

## Open-Book 深度成本穿透审查

只有 `source = "excel"` 的 BOM 行会触发以下专项信号；手工行与内置样例不会误触发。所有信号仅在门禁通过、进入 BOM 分析后生成。

| 规则 | 触发条件 | 信号 |
| --- | --- | --- |
| 材料利用率 | 毛净重推导废料率或申报废料率 ≥ 35%；≥ 50% 为高优先级 | 要求提供排样图或工艺改善 |
| 加工节拍 | `Cycle Time > 1800 秒` | 核对现场排班、设备 OEE 与工时测量记录 |
| 人工费率 | 长三角基准 120 元/小时，上浮 30% 即 > 156 元/小时 | 同上 |
| 设备费率 | 长三角基准 180 元/小时，上浮 30% 即 > 234 元/小时 | 核对费率构成（折旧、能源、维护） |
| 工装摊销 | 寿命 ≤ 5 万件，且单件摊销 ≥ 1 元或 ≥ BOM 单价 2% | 确认寿命是否匹配车型生命周期 |
| 利润穿透 | Excel Mark-ups 占售价 > 15%；> 20% 为高优先级 | 拆分 SG&A 与净利润，排查重复计费 |

阈值集中在 [engine-shared.ts](lib/engine-shared.ts) 的 `excel*` 字段，可按地区/客户标准调整。

![Excel 深度审查信号](docs/screenshots/excel-deep-review.png)

## 报告结构

`EvaluationReport` 包含六个固定区块：

- `STATUS`：决策、置信度、核心数字、下一步动作；
- `GATE`：逐项 PASS / WARN / FAIL，是否阻断、实际值、阈值、动作；
- `COMPARE`：自述透明度、BOM 推导透明度、材料占比一致性等；
- `BOM_GAP`：成本构成、证据缺口、材料利用率；
- `SENSITIVITY`：材料/人工/设备/物流/废料/需求下滑对毛利率的冲击；
- `SIGNALS`：填报时长、字段反复修改、跳填、证据矛盾等行为信号。

门禁阻断时，`bom_gap.skipped = true` 且 `sensitivity.skipped = true`，报告中不会出现报价结构推断。

## DeepSeek 的边界

`lib/deepseek.ts` 只接收规则引擎已经计算好的结构化结果，要求返回固定 JSON 叙述层。合并时：

- 只替换 `status_narrative / executive_summary / recommended_actions / gate_notes / compare_notes / bom_narrative / sensitivity_narrative / signal_notes` 的文本；
- 不修改任何 `decision`、gate 状态、金额、比例或阈值；
- 请求超时（45s）、HTTP 错误、JSON 结构不合法时自动回退规则引擎；
- 阻断报告中禁止 AI 分析 BOM 结构。

## 主要路由

| 路由 | 说明 |
| --- | --- |
| `/` | 首页 + 五步填报 |
| `/report/[id]` | 结构化报告（Supabase 或 sessionStorage） |
| `POST /api/evaluate` | Zod 校验 → 门禁 → 可选 AI → Supabase |
| `POST /api/upload` | Supabase Storage 上传（≤ 15MB） |
| `GET /api/report/[id]` | 读取持久化报告 |

## 目录

```text
app/
  page.tsx                  首页与规则说明
  report/[id]/page.tsx      报告页
  api/evaluate/route.ts     评测入口
  api/upload/route.ts       文件上传
  api/report/[id]/route.ts  历史报告
components/
  evaluation-form/          五步表单、上传器、字段组件
  report/report-view.tsx    六区块报告
  ui/                       shadcn 风格基础组件
lib/
  schemas.ts                Zod 唯一真相
  engine.ts                 门禁、BOM、敏感性、信号
  engine-shared.ts          阈值与工具
  deepseek.ts               AI 叙述层
  supabase-server.ts        持久化
  sample-data.ts            通过 / 阻断样例
supabase/schema.sql
```

## 验证

```bash
pnpm typecheck
pnpm build
```

首页提供两个样例：

- **载入通过样例**：全部门禁通过，展示完整六区块与敏感性；
- **载入阻断样例**：财务亏损、CSR 低于底线、绿电不足、BOM 无证据，验证 `BLOCKED` 短路。

## 已知边界

- 报告与 Supabase 均为演示级配置，正式使用前需补充认证、RLS、服务端限流与审计；
- 文件上传只存储对象路径，未实现签名下载；
- 财务金额单位为“元”，未做币种换算；
- 敏感性基于售价不变的近似，目标毛利率固定为 8%，可在阈值文件调整；
- 本工具是采购决策辅助，不替代现场审核、财务尽调与法务意见。
