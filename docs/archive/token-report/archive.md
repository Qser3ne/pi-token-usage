# token-report

## Purpose

定义本地用量报告的持久化、模型明细、缓存命中率及历史兼容展示。

## Requirements

### Requirement: 上下文外的持久报告

查询成功 SHALL 将 UsageReport 保存为 pi-token-usage.report 自定义条目，供 TUI 历史渲染，不进入模型上下文；RPC 模式 SHALL 另发送 info 级文本通知。

#### Scenario: 查看已保存报告

- **WHEN** 重新渲染先前报告
- **THEN** SHALL 展示该次查询保存的数值，不按现在的模型名单重新过滤或改写
- **AND** 缺少可渲染数据时 SHALL 显示报告不可用，不编造统计

### Requirement: 整体与模型明细

报告 SHALL 显示时间范围、整数总量、输入、输出、缓存读取、缓存写入和缓存命中率；MUST 不使用 K/M 缩略值。新报告 SHALL 标记「当前模型列表」，按 provider/model 显示有用量的组合。

#### Scenario: 多个模型或全零数据

- **WHEN** 查询窗口内存在多个允许的模型组合
- **THEN** 明细 SHALL 按总 token 降序排列，同量时按名称稳定排序
- **AND** 任一 token 分项或总量大于零 SHALL 视为有用量；全零、窗口外或从未使用的组合不补零占位
- **AND** 整体各项与模型组汇总 SHALL 使用相同名单和时间口径，分组之和与整体一致

### Requirement: 缓存命中率口径

缓存命中率 MUST 等于 cacheRead ÷（input + cacheRead + cacheWrite）×100%，不含输出；整体与每个模型 SHALL 先汇总 token 再计算比例，不平均记录百分比。

#### Scenario: 缓存写入或无输入

- **WHEN** 报告包含缓存写入
- **THEN** 缓存写入 SHALL 计入分母但不算命中，比例保留两位小数
- **AND** 输入侧分母为零 SHALL 显示「—」，不显示虚构的 0% 或 100%

### Requirement: 旧报告兼容

缺少 scope 标记的历史报告 SHALL 保留「所有渠道」标签；缺少 models 明细时 MUST 只显示整体，不把旧 providers 明细当成模型。

#### Scenario: 历史模型标签不完整

- **WHEN** 旧报告已有模型明细但缺少 provider 或 model 标签
- **THEN** SHALL 分别显示「未知渠道」或「未知模型」
- **AND** 不主动重算旧报告以套用新的过滤范围

### Requirement: 诊断保存与显示边界

来源缺失、损坏行和统计不确定性等 warnings SHALL 保存在报告数据中；界面 MUST 不展示整块精度警告标题、警告列表或来源脚注。

#### Scenario: 报告带有内部诊断

- **WHEN** 成功查询的报告包含 warnings
- **THEN** TUI 与 RPC SHALL 仍按普通报告显示，不附精度警告区或来源脚注
- **AND** 隐藏诊断 MUST 不被描述为数据完整保证；参数错误及真正查询失败仍按[查询规则](../token-query/archive.md)通知
