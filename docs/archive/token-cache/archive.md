# token-cache

## Purpose

定义用量快照的磁盘及实例内缓存、失效、并发持久化与隐私边界，不提供历史备份。

## Requirements

### Requirement: 数据最小化与缓存位置

磁盘缓存 SHALL 位于 Pi agent directory 的 pi-token-usage/cache-v1/，跟随 PI_CODING_AGENT_DIR；MUST 只保存统计记录、时间、provider/model、哈希和最小来源元数据，不保存对话正文。

#### Scenario: 自定义 agent directory

- **WHEN** 用户使用非默认 agent directory 查询
- **THEN** SHALL 在对应目录内读写缓存，不固定使用 ~/.pi/agent
- **AND** 缓存 SHALL 使用文件快照协议 version=4，旧协议或无效快照从源会话重建，目录名不因此改变

### Requirement: 每次验证源文件

缓存命中 MUST 仍检查源文件身份、大小及修改元数据；每次查询 SHALL 重新发现来源并重新计算去重、协调、时间窗及模型过滤，不复用最终报告。

#### Scenario: 源文件变化或删除

- **WHEN** 源文件追加、替换、截断、同大小改写或删除
- **THEN** SHALL 按对应变化增量读取或使快照失效，已删除来源的缓存不继续作为历史账本计入
- **AND** 模型名单变化 SHALL 可复用完整源快照，不丢弃先前被排除模型的历史

### Requirement: 增量读取与保证范围

追加路径 SHALL 校验文件身份、已有前缀及追加边界，只解析新增完整行；MUST 不声称能抵御恶意伪造文件元数据或刻意保留采样区的历史改写。

#### Scenario: 末行补全或源文件被重写

- **WHEN** 上次末行未写完后被补全
- **THEN** SHALL 从未完成边界继续读取，不重复统计已完整读取的记录
- **AND** 扫描期间检测到截断或同大小重写 SHALL 记录无法完整扫描诊断，不悄悄使用不一致结果

### Requirement: 原子持久化与失败回退

快照 SHALL 逐文件使用唯一临时文件和原子替换保存，支持多个 Pi 会话并发查询；创建缓存目录及文件 SHALL 分别使用 0700 和 0600 模式。

#### Scenario: 缓存损坏或无法写入

- **WHEN** 缓存损坏、缺失或写入失败
- **THEN** SHALL 尽可能回退源扫描并完成有效统计，同时保留诊断
- **AND** 新快照保存失败 MUST 不进入实例内缓存，以便后续查询重试落盘
- **AND** 缓存权限 MUST 不被描述为同用户安全沙箱或对已有目录权限的自动修复

### Requirement: 实例内有界快照

每个扩展实例 SHALL 独立保留最多 4096 个文件快照和按序列化统计元数据计重的 64 MiB 额度，按最近访问淘汰；超大项 SHALL 不保留。额度 MUST 不被描述为精确 JavaScript 堆上限。

#### Scenario: 重载或容量用尽

- **WHEN** Pi 重启或 /reload 重建扩展实例
- **THEN** SHALL 清空该实例内存缓存，磁盘快照仍可按验证规则复用
- **AND** 容量耗尽 SHALL 淘汰较旧访问项，不无限增长，不共享其他扩展实例的内存快照

### Requirement: 有界扫描并发

目录发现 SHALL 最多 8 路并发，来源读取 SHALL 最多 4 路并发；MUST 不新增后台监视器或周期性历史扫描。

#### Scenario: 重复查询大型历史

- **WHEN** 源文件没有变化且快照可用
- **THEN** SHALL 复用有效快照，仍重新发现文件和验证来源，不承诺固定耗时或加速倍数

### Requirement: 缓存清理不等于源数据删除

用户 SHALL 可在 Pi 空闲时删除扩展缓存；删除磁盘缓存 MUST 不被描述为已经清除当前实例内存，源 sessions/ MUST 不作为扩展缓存清理目标。

#### Scenario: 强制重新扫描

- **WHEN** 用户删除扩展缓存目录并执行 /reload
- **THEN** 下次查询 SHALL 从源会话重建快照
- **AND** 只删除磁盘目录而不重载时 SHALL 仍可复用经源文件检查的内存快照
