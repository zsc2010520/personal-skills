---
name: aep-xdm-schema-design
description: 设计和评审 Adobe Experience Platform (AEP) 的 XDM schema：选择 class、划分 profile 属性与事件、设计身份字段、规划 Profile 启用、处理 schema 变更。当用户要新建 schema、把源数据映射成 XDM、评审数据模型，或询问某个字段该放 profile 还是 event、能不能修改或删除字段时使用。
---

# AEP XDM Schema 设计

XDM schema 的很多决定做了就不能撤回。设计时宁可少放字段、晚启用 Profile，也不要先上线再修改。给出方案时把不可逆的部分单独标出来。

## 不可逆的操作

动手之前先让用户知道：

- schema 一旦用于摄入数据，或启用了 Profile，就只能做增量修改。
- **不能做**：删除字段、重命名字段、改字段类型、把字段移到别的层级、收窄取值范围（例如删除枚举值）、删除 schema。
- **可以做**：新增字段、把必填改成可选、新增必填字段（只对之后的数据生效）、改显示名和描述、启用 Profile。
- schema 启用 Profile 之后不能关闭，也不能删除。已启用 Profile 且有数据的 schema 不能更换主身份字段。
- 字段被摄入过数据之后不能移除，而且在所有引用它的 schema 里都会变成不可编辑。
- 需要淘汰一个已启用 Profile 的 schema 时，做法是改名标注为废弃、新建替代的 schema、停止向旧数据集摄入。

## 先收集信息

- 业务用例：要做什么分群、什么激活、什么分析。没有用例就没有取舍的依据。
- 数据源清单：每个源有哪些实体、主键是什么、更新方式（全量、增量、流式）、数据量级。
- 每个源里可以识别到个人的标识有哪些。
- 是否需要进 Profile，是否用于实时场景。
- 用户所在组织的租户命名空间和已有的命名规范。

## 设计步骤

### 1. 给每个实体分类

| 类型 | 判断标准 | class |
| --- | --- | --- |
| Profile | 描述一个人当前的状态和属性，可变 | XDM Individual Profile |
| Event | 某个时间点发生的行为或系统事件，不可变 | XDM ExperienceEvent |
| Lookup | 与人相关但不能识别人的维度数据（商品、门店、套餐） | 自定义 class，通过 schema relationship 关联 |

判断规则：

- 姓名、生日、账号、地址、电话、邮箱这类属性属于 profile。
- 需要分析"如何随时间变化"的，建模成 event。
- 分群看当前状态（例如会员等级）的放 profile；看一段时间内的行为（例如近一年的购买）的放 event。
- 一对多关系里，"多"的一方是交易性质的建成 event；只关心当前状态的，可以做成 lookup 或 profile 里的数组。
- 平台不会自动聚合。需要累计值（累计消费、订单数）时，在摄入前或用 Query Service 算好再写入 profile。

数组与事件的取舍：profile 里放数组便于按当前状态分群，但每次变更要重传整个数组，多数据源同步困难；建成事件便于按时间分群，但按当前状态分群更复杂，事件还可能过期。把这个取舍讲给用户，由用例决定。

### 2. 设计身份

- 一个 schema 可以有多个身份字段。要启用 Profile，必须有且只有一个主身份。
- 适合做身份的字段：主键、账号 ID、邮箱、手机号、ECID。
- 使用 Adobe 应用（Web SDK、Analytics 等）的 field group 时，通常由 `identityMap` 承担主身份，不要再把别的字段标成主身份，其他标识标为次身份。
- ExperienceEvent 的 `_id` 是事件的唯一标识，不能用作身份。
- 身份值必须能唯一对应一个人。多人共用的值（公共邮箱、门店电话、默认值、测试账号）会把不同的人合并成一个 profile，这是最常见也最难修复的问题。
- 给身份字段加长度限制，格式固定的加 pattern 约束，把脏数据挡在摄入阶段。
- 每个身份字段使用哪个命名空间，要和其他数据源保持一致，否则同一个人连不到一起。

### 3. 组织字段

- 优先使用标准 field group。标准里没有的字段放在自定义 field group 里，自定义字段位于租户命名空间下。
- 按业务含义分组成对象，不要全部平铺在一层，也不要嵌套过深。
- 命名在整个组织内保持一致的风格。字段名一旦使用就不能更改。
- 类型一次选对：金额用数值，日期和时间用 date 或 date-time，标志位用 boolean。不要图省事全用字符串，类型以后不能改。
- 不确定是否需要的字段先不加，以后可以随时新增，但不能删除。
- 事件实体基本上各自单独建 schema。一对一的 profile 属性可以合并进同一个 schema。

### 4. 设置校验

- 批量摄入只校验设置了约束的字段，数组内嵌套的字段不校验。
- 可用的约束：必填、pattern、format、长度上下限、数值上下限、枚举。
- 字符串字段建议最小长度设为 1。
- 默认值只是说明性的，摄入时不会自动填充。
- 平台不接受 null 或空值，没有值的字段在 payload 里省略。因此必填字段要谨慎设置，源数据不能保证有值的不要设为必填。

### 5. 规划 Profile 启用

- 顺序是先启用 schema，再逐个启用数据集。只让确实需要进 Profile 的数据集启用。
- 启用之前先用未启用 Profile 的数据集把摄入流程和数据质量验证完，再启用。启用之前摄入的数据不会自动进入 Profile。
- 启用前确认：主身份正确且有值、身份不会造成错误合并、字段清单已经评审、数据量和事件保留策略符合 Profile 的护栏。
- 启用是不可逆的。把"是否启用"作为需要用户明确确认的决定，不要替用户默认。

## 交付要求

1. 给出实体分类表：每个实体的类型、class、主身份、次身份及命名空间。
2. 给出字段清单：路径、类型、是否必填、约束、来源字段、所属 field group。
3. 单独列出不可逆的决定和需要用户确认的事项（主身份、Profile 启用、必填字段、字段类型）。
4. 说明每个取舍的理由，尤其是 profile 与 event、数组与事件的选择。
5. 评审已有 schema 时，按"身份风险、类型错误、不必要的字段、缺少约束"分类指出问题。对不能直接修改的问题，给出增量补救方案（新增字段、新建 schema 并迁移）。

## 参考

- 数据建模最佳实践 https://experienceleague.adobe.com/en/docs/experience-platform/xdm/schema/best-practices
- Schema 组合基础与演进规则 https://experienceleague.adobe.com/en/docs/experience-platform/xdm/schema/composition
- Profile 启用规划 https://experienceleague.adobe.com/en/docs/experience-platform/xdm/schema/profile-enablement-planning
- XDM 排错指南 https://experienceleague.adobe.com/en/docs/experience-platform/xdm/troubleshooting-guide

产品行为会更新。本文件与官方文档或用户环境的实际表现不一致时，以后者为准，并提醒用户更新本技能。
