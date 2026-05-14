# FlowTrace 自动化流程留痕系统：从脚本按钮到周期化流程自动化

> 一种面向网页自动化脚本面板的动作采集、周期识别与动作包沉淀方案  
> 项目代号：**FlowTrace**  
> 事件规范：**ARS-v0.1｜Automation Recording Standard v0.1**

---

## 0. 摘要

网页自动化脚本通常从“按钮”开始：一个按钮封装一组动作，例如点击菜单、等待页面、选择字段、提交导入、等待结果。随着脚本数量增加，按钮会越来越多，流程也会被拆散在多个面板、多个版本和多段人工经验里。

**FlowTrace 的目标不是再增加一组按钮，而是记录人工、脚本、网页共同完成任务的过程，识别重复周期，分析稳定性，并把稳定片段沉淀成可复用动作包。**

它面向的不是某一个网站或某一个脚本，而是大部分带有控制面板的网页自动化脚本：

- Snov.io 导入 / 导出脚本
- CRM 跟进 / 联系人收集脚本
- Outlook 自动发送脚本
- 电商、后台、数据平台中的表格处理脚本
- 任何“人工操作 + 脚本按钮 + 页面状态判断”共同构成的网页自动化流程

FlowTrace 的核心架构是：

```text
网页脚本端：轻量记录器
        ↓
统一事件文件：ARS-v0.1
        ↓
Python 分析端：周期识别、重复片段提取、稳定性分析
        ↓
输出报告：人类可读 + AI/程序可继续加工
```

最终产物不是一份流水账，而是一组可判断、可复查、可沉淀的结果：

```text
report.html              给人看的可视化报告
events.jsonl             原始事件流
cycles.csv               周期摘要
flow_candidate.json      动作包候选
summary.json             总结与风险判断
raw_log.txt              兜底原始日志
```

---

## 1. 项目命名

## 1.1 推荐名称

**FlowTrace｜自动化流程留痕系统**

中文简称：

```text
流程留痕
```

英文仓库名：

```text
flowtrace-automation-recorder
```

一句话定义：

```text
FlowTrace 记录人工、脚本、网页共同完成的自动化流程，识别重复周期，分析稳定性，并生成可复用动作包。
```

## 1.2 为什么不用“流程录制器”

“录制器”容易让人联想到鼠标宏：记录坐标、按时间回放、模拟点击。

FlowTrace 不是这种工具。它记录的是：

```text
谁做了动作
做了什么动作
动作属于哪个周期
动作前后页面处于什么状态
动作是否成功
不同周期之间是否重复
哪些片段可以固化成动作包
```

因此，“留痕”比“录制”更准确。它强调的是**可分析的过程证据**，而不是简单回放。

---

## 2. 项目背景：按钮脚本的自然瓶颈

网页自动化开发通常从小功能开始。

一开始，一个按钮解决一个痛点：

```text
点击搜索
点击导入
点击导出
点击写记录
点击下一步
点击确认
```

随着功能增加，按钮会变成动作包：

```text
点击一个按钮
  → 等待页面稳定
  → 找到目标按钮
  → 点击
  → 判断是否成功
  → 失败重试
```

再继续发展，多个动作包会组成流程：

```text
搜索列表
  → 进入列表
  → 添加潜在客户
  → 从文件导入
  → 等待人工选择文件
  → 上传文件
  → 下一步
  → 导入
  → 返回后验证邮箱
```

这时问题开始出现：

1. **按钮数量越来越多**  
   面板变复杂，用户要记住每个按钮的使用时机。

2. **动作关系藏在人工经验里**  
   人知道“点完这个应该等那个”，但脚本不知道。

3. **周期性流程没有被识别**  
   同一套流程重复执行很多轮，但脚本只看到一次次孤立点击。

4. **异常很难复盘**  
   出错时只知道“卡住了”，不知道是哪个周期、哪个动作、哪个页面状态不稳定。

5. **脚本开发经验难以迁移**  
   一个网站调过的等待、重试、按钮识别经验，不能自然迁移到另一个脚本面板。

FlowTrace 解决的正是这个瓶颈：**从按钮级自动化，升级到周期化流程自动化。**

---

## 3. 项目目标

FlowTrace 的目标分为三层。

## 3.1 记录：不断链地记录人、脚本、网页动作

FlowTrace 要记录的不只是人工在网页上点了什么，也要记录人工在脚本面板上点了什么，以及脚本自动执行了什么。

动作来源至少分为四类：

```text
human_panel     人工点击脚本面板
human_web       人工点击网页
script_auto     脚本自动操作网页
system          页面或系统状态变化
```

示例：

```json
{"source":"human_panel","type":"click","target_text":"搜索"}
{"source":"script_auto","type":"click","target_text":"添加潜在客户"}
{"source":"human_web","type":"click","target_text":"选择文件"}
{"source":"system","type":"url_change","url":"https://app.snov.io/import/prospects"}
```

关键不是记录得越细越好，而是记录得**足够判断流程**：

```text
点了什么
谁点的
属于哪一轮
当时大概是什么页面状态
动作是否成功
耗时多久
是否发生异常
```

## 3.2 分析：识别周期、重复片段和不稳定步骤

大部分自动化流程都有“周期”。

例如：

```text
第 1 轮：列表 A → 导入 → 下一步 → 导入 → 验证
第 2 轮：列表 B → 导入 → 下一步 → 导入 → 验证
第 3 轮：列表 C → 导入 → 下一步 → 导入 → 验证
```

如果多个片段有 80% 以上重复，就可能是一个可抽象周期。

FlowTrace 不要求浏览器脚本实时完成复杂判断。更合理的是：

```text
浏览器脚本只采集统一事件
Python 程序负责周期识别和相似度分析
```

Python 可以计算：

```text
每轮步骤是否一致
每个步骤是否稳定
哪些动作经常重试
哪些动作依赖人工
哪些步骤适合合并成动作包
哪些步骤不适合自动化
```

## 3.3 沉淀：把稳定片段转成动作包候选

当某段流程在多个周期里稳定出现，就可以沉淀成动作包。

例如：

```text
动作包：打开文件导入
包含：
  1. 点击添加潜在客户
  2. 点击从文件导入
  3. 等待文件导入弹窗出现

稳定性：高
建议：可固化
```

再比如：

```text
动作包：上传文件并进入字段映射
包含：
  1. 等待人工选择文件
  2. 点击上传文件
  3. 等待字段映射页出现

稳定性：中
原因：上传后偶尔不跳转
建议：固化，但必须保留 3 次重试
```

---

## 4. 参考的成熟方法体系

FlowTrace 不是凭空设计出来的，它可以借鉴几套成熟体系，但不完全照搬。

## 4.1 Task Mining：解释为什么要记录人工动作

Task Mining 关注用户在桌面或应用中实际执行任务的步骤。Microsoft 对 task mining 的说明是：它可以捕获用户执行任务的详细步骤，并通过分析录制的用户动作，理解任务如何执行、发现常见错误和自动化机会。  
参考：Microsoft Power Automate Task Mining Overview  
https://learn.microsoft.com/en-us/power-automate/task-mining-overview

FlowTrace 借鉴的是这部分思想：

```text
记录人工点击
记录人工输入
记录脚本面板操作
记录人工介入点
```

但 FlowTrace 更窄、更工程化：它不是观察办公室员工怎么工作，而是观察**网页自动化脚本开发过程中，人、脚本、网页如何协同完成任务**。

## 4.2 Process Mining：解释为什么要分析周期和流程

Process Mining 通过事件日志理解真实流程如何运行，识别瓶颈、低效和自动化机会。Microsoft 文档也强调 process mining 通过业务流程数据帮助理解真实流程、发现改进和自动化机会。  
参考：Microsoft Power Automate Process Mining Overview  
https://learn.microsoft.com/en-us/power-automate/process-mining-overview

FlowTrace 借鉴的是：

```text
事件流
周期
流程路径
异常分支
重复片段
瓶颈步骤
```

区别是：传统 Process Mining 多分析企业系统日志；FlowTrace 分析的是**网页脚本面板与浏览器页面共同产生的事件流**。

## 4.3 XES：借鉴事件日志标准

XES 是 process mining 领域常用的事件日志标准。IEEE 1849-2023 是 eXtensible Event Stream 标准，用于事件日志与事件流的互操作。  
参考：IEEE 1849-2023  
https://standards.ieee.org/ieee/1849/10907/

FlowTrace 不建议完整照搬 XES，因为 XES 对本项目来说偏重。但它应该吸收核心思想：

```text
case = 一个周期
event = 一个动作
activity = 动作名称
timestamp = 时间
attributes = 附加信息
```

因此 FlowTrace 定义自己的轻量规范：

```text
ARS-v0.1｜Automation Recording Standard v0.1
```

## 4.4 BPMN：用于人类可读流程表达

BPMN 是业务流程建模标准。OMG 对 BPMN 的定位是：提供一种业务用户可理解、同时又能表达复杂流程语义的图形化记法。  
参考：OMG BPMN  
https://www.omg.org/bpmn/

FlowTrace 可以借鉴 BPMN 做报告里的流程图，例如：

```text
主页稳定
  ↓
点击添加潜在客户
  ↓
点击从文件导入
  ↓
等待人工选文件
  ↓
上传文件
  ↓
字段映射：下一步
  ↓
数据设置：导入
```

但 BPMN 不适合做底层数据格式。底层仍应使用 JSONL / JSON / CSV。

## 4.5 Trace / Span：用于层级结构设计

OpenTelemetry 的 Trace / Span 模型很适合 FlowTrace 借鉴。OpenTelemetry 文档中，trace 可以理解为由 span 构成的有向无环图；span 表示一个具体操作，并通过父子关系组织起来。  
参考：OpenTelemetry Traces  
https://opentelemetry.io/docs/concepts/signals/traces/  
https://opentelemetry.io/docs/specs/otel/overview/

对应到 FlowTrace：

```text
Trace = 一次录制会话
Span = 一个周期
Child Span = 周期里的动作步骤
Event = 点击、输入、等待、URL 变化、异常
```

这种结构比普通平铺日志更适合分析。

---

## 5. 系统总体架构

FlowTrace 分为三端。

## 5.1 网页脚本端：轻量记录器

职责：

```text
记录面板点击
记录网页点击
记录输入事件
记录 URL 变化
记录脚本自动步骤
记录周期开始 / 周期结束
导出统一事件文件
```

它不负责复杂分析，不做流程推断，不生成最终动作包。

原则：

```text
只观察，不主动额外操作网页
只记录关键事件，不保存完整 DOM
只在用户开启录制后工作
默认不记录敏感输入值
```

## 5.2 Python 分析端：周期分析器

职责：

```text
读取 events.jsonl
按人工标记切分周期
无标记时辅助识别重复片段
计算周期相似度
识别稳定步骤和不稳定步骤
提取动作包候选
生成 HTML 报告、CSV 摘要和 JSON 动作包建议
```

Python 更适合做：

```text
序列相似度
周期聚类
异常排行
重复片段提取
统计报表
可视化报告生成
```

## 5.3 AI 辅助端：动作包归纳与脚本生成

AI 不直接控制网页，它负责读结构化结果：

```text
summary.json
events.jsonl
cycles.csv
flow_candidate.json
```

然后辅助判断：

```text
哪些步骤适合自动化
哪些步骤必须保留人工
哪些等待条件不够稳定
哪些动作包可以固化
正式脚本应该怎么写
```

---

## 6. 统一事件规范：ARS-v0.1

ARS 是 FlowTrace 的核心。不同项目的脚本面板都必须按同一规范导出事件，否则 Python 无法统一处理。

## 6.1 会话级字段

```json
{
  "schema": "ARS-v0.1",
  "project": "SN",
  "script": "SN导入导出助手",
  "script_version": "4.0.3",
  "session_id": "2026-05-14_170022",
  "started_at": "2026-05-14 17:00:22",
  "timezone": "Asia/Shanghai"
}
```

## 6.2 事件级字段

```json
{
  "event_id": 12,
  "time": "2026-05-14 17:03:21",
  "cycle_id": "cycle_001",
  "source": "human_web",
  "type": "click",
  "target_text": "上传文件",
  "target_role": "button",
  "page_state": "file_import_modal",
  "url": "https://app.snov.io/prospects",
  "result": "recorded",
  "note": ""
}
```

## 6.3 source 枚举

```text
human_panel     人工点击脚本面板
human_web       人工点击网页
script_auto     脚本自动动作
system          系统观察事件
```

## 6.4 type 枚举

```text
click
input
change
keydown
url_change
cycle_start
cycle_end
script_step_start
script_step_end
wait_start
wait_end
error
pause
resume
```

## 6.5 result 枚举

```text
recorded
success
failed
timeout
skipped
retry
manual_required
```

---

## 7. 什么是快照

在 FlowTrace 里，**快照不是截图**。

快照是某个时间点的页面状态摘要，用来帮助后续判断“动作发生时页面处于什么状态”。

例如点击“上传文件”前：

```json
{
  "snapshot_type": "before_action",
  "page_state": "file_import_modal",
  "visible_buttons": ["取消", "上传文件"],
  "has_loading": false,
  "has_modal": true,
  "url": "https://app.snov.io/prospects",
  "keywords": ["从文件导入", "上传文件"]
}
```

点击后：

```json
{
  "snapshot_type": "after_action",
  "page_state": "import_wizard_mapping",
  "visible_buttons": ["取消导入", "下一步"],
  "has_loading": false,
  "has_modal": false,
  "url": "https://app.snov.io/import/prospects",
  "keywords": ["字段映射", "跳过第一行", "下一步"]
}
```

快照的价值是：

```text
判断动作前是否在正确页面
判断动作后是否进入预期状态
帮助 Python 识别异常
帮助 AI 理解失败原因
```

快照不保存完整网页，不保存完整 DOM，只保存摘要。

---

## 8. 周期识别设计

## 8.1 人工标记优先

第一版应支持人工标记：

```text
周期开始
周期结束
```

事件示例：

```json
{"type":"cycle_start","cycle_id":"cycle_001","label":"珠宝包装岗位"}
{"type":"click","source":"human_panel","target_text":"搜索"}
{"type":"click","source":"script_auto","target_text":"添加潜在客户"}
{"type":"click","source":"human_web","target_text":"选择文件"}
{"type":"cycle_end","cycle_id":"cycle_001","result":"success"}
```

人工标记最稳，也最适合开发阶段。

## 8.2 自动识别作为辅助

没有人工标记时，Python 可以通过动作相似度识别周期。

把动作转成 token：

```text
human_panel:搜索
script_auto:添加潜在客户
script_auto:从文件导入
human_web:选择文件
script_auto:上传文件
script_auto:下一步
script_auto:导入
```

然后比较多个片段。

如果两个片段相似度超过 80%，可判断为同类周期候选。

## 8.3 周期稳定性指标

每个周期可以计算：

```text
步骤完整率
步骤顺序一致率
耗时波动
重试次数
异常数量
人工介入点
失败步骤
```

输出判断：

```text
稳定：可以固化为动作包
基本稳定：可以固化，但需要保留重试和异常处理
不稳定：暂不适合固化，需要继续侦察
危险：不建议自动执行，容易误点或漏判断
```

---

## 9. 动作包沉淀

动作包不是随便把几步合在一起，而是必须满足三个条件：

```text
出现频率高
顺序稳定
前置条件和后置验证明确
```

动作包示例：

```json
{
  "name": "打开文件导入",
  "stability": "high",
  "steps": [
    {
      "action": "click_by_text",
      "text": "添加潜在客户",
      "postcondition": "menu_contains_从文件导入"
    },
    {
      "action": "click_by_text",
      "text": "从文件导入",
      "postcondition": "file_import_modal_open"
    }
  ]
}
```

动作包必须保留：

```text
前置条件
动作
后置验证
失败重试
失败退出
是否需要人工介入
```

否则动作包只是更大的按钮，不是真正可靠的流程自动化。

---

## 10. 输出结果设计

FlowTrace 的输出必须同时服务三类对象：

```text
人
Python 程序
AI
```

## 10.1 给人的 report.html

人看的报告重点是判断效率。

结构：

```text
一、流程总览
二、周期列表
三、稳定性分析
四、异常步骤排行
五、每轮详情
六、动作包建议
七、原始事件附录
```

示例总览：

```text
流程名称：SN 导入流程
录制时间：2026-05-14 16:30:22
周期数量：12
成功周期：10
失败周期：2
平均耗时：48.2 秒
最长耗时：91.4 秒
人工介入点：选择文件
最高风险步骤：点击上传文件
建议：可以固化为动作包，但上传后跳转需要保留重试逻辑
```

## 10.2 给表格查看的 cycles.csv

```text
轮次,目标,结果,耗时,重试,异常,判断
1,珠宝包装岗位,成功,43s,0,无,稳定
2,美妆采购岗位,成功,47s,0,无,稳定
3,鞋服包装岗位,成功,66s,1,上传后未跳转,基本稳定
4,礼品高管岗位,失败,90s,3,上传按钮点击无效,不稳定
```

## 10.3 给程序和 AI 的 JSON / JSONL

建议导出：

```text
summary.json
events.jsonl
cycles.jsonl
flow_candidate.json
```

flow_candidate.json 示例：

```json
{
  "flow_name": "SN导入一轮",
  "confidence": 0.86,
  "steps": [
    {
      "name": "点击添加潜在客户",
      "action": "click_by_text",
      "text": "添加潜在客户",
      "precondition": "main_page_ready",
      "postcondition": "menu_contains_从文件导入"
    },
    {
      "name": "点击从文件导入",
      "action": "click_by_text",
      "text": "从文件导入",
      "postcondition": "file_import_modal_open"
    },
    {
      "name": "等待人工选择文件",
      "action": "wait_button_enabled",
      "text": "上传文件"
    }
  ],
  "warnings": [
    "文件选择无法由浏览器脚本自动完成",
    "上传步骤需要保留重试"
  ]
}
```

---

## 11. 网页安全与账号风险

FlowTrace 的浏览器脚本必须遵守一个原则：

```text
录制器只观察，不主动额外操作网页。
```

允许：

```text
监听点击
监听输入
监听 URL 变化
低频观察 loading / 弹窗 / 按钮状态
记录脚本自身动作
停止后导出本地文件
```

不允许：

```text
录制器额外点击网页
录制器自动提交数据
录制器向外部服务器上传数据
录制器高频扫描全 DOM
录制器绕过网页限制
```

账号风险主要来自额外业务行为，而不是本地监听本身。FlowTrace 不应制造额外点击和额外请求，因此对网页账号的风险应控制在较低水平。

---

## 12. 最小可行版本

第一版不要做成完整平台。

## 12.1 浏览器脚本 v0.1

功能：

```text
开始录制
周期开始
周期结束
停止录制
导出
```

记录：

```text
面板点击
网页点击
网页输入
URL 变化
脚本自动步骤日志
错误日志
```

导出：

```text
events.jsonl
summary.json
raw_log.txt
```

## 12.2 Python 分析器 v0.1

功能：

```text
读取 events.jsonl
按人工 cycle_id 分组
输出每轮动作列表
统计每轮耗时
统计动作出现次数
计算周期相似度
生成 report.html
生成 cycles.csv
```

## 12.3 第二阶段

加入：

```text
重复片段提取
动作包候选生成
异常排行
稳定性评分
flow_candidate.json
```

---

## 13. 泛化到大部分脚本面板的方式

FlowTrace 不绑定某个网站，而是绑定一类自动化形态：

```text
有脚本面板
有人工点击
有脚本自动步骤
有网页状态变化
有重复周期
有失败重试
有动作沉淀需求
```

因此，它可以泛化到：

```text
Snov.io 导入 / 导出
CRM 联系人收集
CRM 跟进记录
Outlook 计划发送
抖音收藏采集
后台表格下载
网页批量上传
跨系统 RPA 操作
```

不同项目只要按 ARS-v0.1 输出事件，就能交给同一个 Python 分析器处理。

---

## 14. 项目价值

FlowTrace 的价值不是“省一个按钮”，而是改变自动化开发方式。

原来的开发方式：

```text
遇到问题
  → 写按钮
  → 报错
  → 加等待
  → 再报错
  → 加判断
  → 按经验维护
```

FlowTrace 之后：

```text
记录流程
  → 分析周期
  → 找出稳定片段
  → 找出不稳定步骤
  → 生成动作包候选
  → 固化为正式脚本
```

它把自动化开发从“凭感觉调脚本”推进到“基于事件证据沉淀流程”。

---

## 15. 最终定位

FlowTrace 是一个面向网页自动化脚本面板的流程留痕与周期分析系统。

它不替代 RPA，也不替代具体业务脚本。它的角色是：

```text
在人工经验和正式自动化之间，建立一层可记录、可分析、可复用的流程证据层。
```

它适合用于：

```text
开发新脚本前的侦察
脚本调试过程中的复盘
多个周期流程的稳定性分析
动作包抽象
正式自动化脚本生成前的证据准备
AI 辅助脚本开发的数据输入
```

最终目标：

```text
让自动化脚本不只是“能跑”，而是能被记录、比较、解释、复用和沉淀。
```

---

## 参考资料

1. Microsoft Power Automate Task Mining Overview  
   https://learn.microsoft.com/en-us/power-automate/task-mining-overview

2. Microsoft Power Automate Process Mining Overview  
   https://learn.microsoft.com/en-us/power-automate/process-mining-overview

3. Microsoft Power Automate Process Mining and Task Mining Overview  
   https://learn.microsoft.com/en-us/power-automate/process-advisor-overview

4. IEEE 1849-2023: eXtensible Event Stream (XES)  
   https://standards.ieee.org/ieee/1849/10907/

5. OMG BPMN  
   https://www.omg.org/bpmn/

6. OpenTelemetry Traces  
   https://opentelemetry.io/docs/concepts/signals/traces/

7. OpenTelemetry Overview  
   https://opentelemetry.io/docs/specs/otel/overview/
