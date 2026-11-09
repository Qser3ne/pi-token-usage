# token-query

## Purpose

定义本机用量查询的命令、时间范围、当前模型名单和查询生命周期，不调用模型或服务商接口。

## Requirements

### Requirement: 命令入口

扩展 SHALL 只注册 /token 与 /token-usage，不提供 /tokens 别名；/token-usage SHALL 固定查询最近 24 小时且拒绝额外参数，/token SHALL 要求正整数加 h 或 d 单位。

#### Scenario: 参数无效

- **WHEN** /token 未带范围，或范围为零、负数、小数、缺少单位或超出可表示范围
- **THEN** SHALL 仅提示用法或范围错误，不读取模型名单、不扫描会话
- **AND** /token-usage 带有非空白参数 SHALL 同样拒绝，不执行查询

### Requirement: 固定滚动时间窗

查询 SHALL 以命令开始时刻为固定截止点，向前滚动指定时长；1 天 MUST 等于 24 小时，上下端点 SHALL 均计入，不按日历日划分。

#### Scenario: 查询进行期间有新记录

- **WHEN** 开始查询后出现时间晚于本次截止点的用量
- **THEN** SHALL 不将该用量计入本次结果，后续查询再按新范围统计
- **AND** 相同时钟下 /token-usage 和 /token 24h SHALL 得到相同范围及统计

### Requirement: 每次冻结当前模型名单

命令 SHALL 在执行时读取 Pi 当前已加载的模型：非空 scopedModels 优先，否则使用 modelRegistry.getAvailable()；MUST 精确匹配 provider/model ID 二元组，不跟随选择器临时搜索词或临时视图。

#### Scenario: 名单变化或为空

- **WHEN** 两次查询间已加载的 scoped/available 模型范围改变
- **THEN** SHALL 对每次查询重新读取名单，使用相同口径过滤整体、记录数、模型明细和缓存率
- **AND** 名单为空 SHALL 返回零，不回退为全部历史或当前单一模型
- **AND** 名单外或缺少任一归属的记录 SHALL 不计入新查询；协调顺序见[统计口径](../token-accounting/archive.md)

#### Scenario: 名单读取失败

- **WHEN** scoped 或 available 读取真正抛错
- **THEN** SHALL 提示查询失败，不回退为无过滤统计
- **AND** 本扩展 MUST 不主动刷新模型目录、解析认证或请求服务商

### Requirement: 本地执行与生命周期

查询 SHALL 读取本地已保存记录并生成[报告](../token-report/archive.md)，MUST 不触发 agent turn 或把结果加入模型上下文。

#### Scenario: 首次查询或会话替换

- **WHEN** 当前扩展实例首次执行有效查询
- **THEN** 有 UI 时 SHALL 提示首次读取可能较慢
- **AND** 查询完成前发生会话替换或关闭时 MUST 不向新会话追加旧报告或发送迟到错误
- **AND** 真正统计失败 SHALL 在当前有效会话有 UI 时以错误通知，不伪造零结果

### Requirement: 运行时依赖边界

扩展 SHALL 通过 src/index.ts 作为唯一包入口运行，使用宿主提供的 Pi 库，不捆绑额外运行时第三方依赖；Node.js MUST >=22.19.0，Pi 兼容性基线为 1.5.0。

#### Scenario: 安装加载

- **WHEN** 宿主通过包入口加载扩展
- **THEN** SHALL 提供两个查询命令及报告渲染器，不自行修改全局安装设置、账号凭据或模型配置
