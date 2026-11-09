# token-accounting

## Purpose

定义本机已保存 Pi 用量的来源覆盖、归属、去重和子代理协调，不将本地记录视为服务商账单。

## Requirements

### Requirement: 有界来源发现

查询 SHALL 递归发现 Pi agent directory 下的 sessions/ 及当前会话 getSessionDir() 中的会话 JSONL，支持 PI_CODING_AGENT_DIR；MUST 不按当前项目限制范围，也不扫描整个磁盘或读取账号凭据。

#### Scenario: 根目录重叠、别名或缺失

- **WHEN** 根目录重复、互相包含或通过符号链接选中
- **THEN** SHALL 解析已选根的真实路径并去重，不因别名重复统计
- **AND** 递归 MUST 不跟随内部符号链接，空根路径不得展开为当前工作目录
- **AND** 不存在的根 SHALL 可产生空扫描，其他读取失败 SHALL 保留诊断并继续可访问来源

### Requirement: 真实用量提取

只有具备有效会话头的来源 SHALL 参与提取；assistant、携带 usage 的 toolResult、独立 usage、压缩摘要及分支摘要 SHALL 保留有效用量，MUST 不估算缺失的文本 token。

#### Scenario: 数值归一化

- **WHEN** 源记录包含有效的非负安全整数用量
- **THEN** 总量 SHALL 优先使用明确的 totalTokens，缺失时使用 input + output + cacheRead + cacheWrite
- **AND** 思考 token 与 1h 缓存写入 SHALL 不被额外重复相加
- **AND** 无效时间或用量 SHALL 跳过并记录诊断，汇总超出安全整数范围 SHALL 使查询失败

### Requirement: 明确的模型归属

provider 和 model MUST 只取用量记录自身明确保存且有效的字段，去除首尾空白；MUST 不从当前模型、相邻记录或父代理选择推断。

#### Scenario: 归属缺失

- **WHEN** 记录缺少有效 provider 或 model
- **THEN** SHALL 在原历史及完整缓存中保留其有效用量，但不计入受[当前模型名单](../token-query/archive.md)约束的新报告
- **AND** 同名模型跨 provider SHALL 独立归属

### Requirement: 分支与复制去重

放弃分支的有效用量 SHALL 保留；fork/clone 复制的同一原始事件 SHALL 只计一次，事件身份 MUST 不因树链接重接而变化。

#### Scenario: 压缩或标签操作重接记录

- **WHEN** 同一事件仅 parentId、firstKeptEntryId 或 fromId 等树链接变化
- **THEN** SHALL 仍视为同一用量，不重复累计
- **AND** 时间或事件内容不同的真实消费 SHALL 不仅因相同树位置而合并

### Requirement: 子代理覆盖协调

识别出的 subagent/bg_wait 汇总 SHALL 优先由引用子会话自身的 assistant 用量覆盖，MUST 不把父级汇总与已覆盖子用量直接相加。

#### Scenario: 子会话包含继承上下文或后代用量

- **WHEN** 协调某次子代理汇总
- **THEN** 继承父上下文、独立缓存预热、压缩及后代工具用量 SHALL 不充当该次 assistant 用量的覆盖依据
- **AND** 这些真实记录 SHALL 仍按自身身份分别参与统计，不因不作为覆盖依据而删除

#### Scenario: 子记录不完整或调用边界不明

- **WHEN** 汇总存在未覆盖差额、重复身份汇总或无法证明独立调用边界
- **THEN** SHALL 只保留未覆盖差额并按汇总时间回退；重复身份汇总 SHALL 以最大累计值回退
- **AND** 汇总和差额 MUST 不继承父 provider/model；无明确归属时不进入新查询
- **AND** SHALL 保留时间不精确或结果可能为下界的诊断，不承诺完整账务

### Requirement: 先协调后过滤

查询 MUST 在完整的截止点前历史上先进行复制去重和子代理协调，再执行模型名单及查询时间窗筛选。

#### Scenario: 某些子模型被排除

- **WHEN** 子会话模型不在当前名单但其用量已经覆盖父汇总
- **THEN** SHALL 不因排除该子模型而重新计回父级汇总
- **AND** 截止点后的更大累计汇总 MUST 不覆盖本次可用的较早汇总

### Requirement: 损坏和不完整来源

损坏 JSON 行 SHALL 跳过并保留诊断；尚未写完的末行 SHALL 暂不计入，后续查询可在补全后读取。

#### Scenario: 历史缺失或不支持的来源

- **WHEN** 会话已删除、未保存、在其他机器或不在选定根内，或 provider 未记录 usage
- **THEN** SHALL 不声称这些消费已被完整覆盖，不以文本估算补齐
- **AND** 诊断数据 SHALL 按[报告规则](../token-report/archive.md)保留，不以界面没有警告证明数据完整
