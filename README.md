# pi-token-usage

通过两个 Pi 命令，查询本机跨项目、跨会话的已保存 token 用量；仅统计当前 `/model` 默认列表中的模型。无需连接服务商接口，不产生模型调用。

## 行为档案

`docs/archive/` 是当前行为的唯一档案位置；以下链接指向源码仓库内文档。

| capability | 范围 |
|---|---|
| [token-query](docs/archive/token-query/archive.md) | 命令、滚动时间窗、当前模型名单和生命周期 |
| [token-accounting](docs/archive/token-accounting/archive.md) | 来源覆盖、用量归属、去重和子代理协调 |
| [token-report](docs/archive/token-report/archive.md) | 模型明细、缓存率、TUI/RPC 和旧报告兼容 |
| [token-cache](docs/archive/token-cache/archive.md) | 两级缓存、失效、并发持久化和隐私边界 |

## 使用示例

```text
/token-usage
/token 24h
/token 7d
/token 12h
```

## 试用与安装

环境：Node.js >=22.19.0、Pi（已针对 1.5.0 验证）。

安装独立 Git 仓库：

```bash
pi install git:github.com/Qser3ne/pi-token-usage
```

需要可复现安装时，在 Git 来源后追加 `@<已推送的完整提交哈希>`，安装后执行 `/reload`。

在本项目根目录临时试用，不修改设置：

```bash
pi -e ./src/index.ts
```

也可自行选择安装本地包：`pi install /absolute/path/to/pi-token-usage`。本地安装引用目录本身，请保留该目录。

## 运维与覆盖限制

统计依赖本机已保存历史，不是服务商账单。其他机器、未保存或已删除记录等覆盖边界见 [token-accounting](docs/archive/token-accounting/archive.md)。

缓存清理位置与注意事项见 [token-cache](docs/archive/token-cache/archive.md)。不要删除源 `sessions/` 来清理扩展缓存；强制清空两级缓存时，删除扩展缓存后再执行 `/reload`。

## 仓库

独立公开仓库：[Qser3ne/pi-token-usage](https://github.com/Qser3ne/pi-token-usage)，同时作为 pi-extensions 的 Git 子模块维护。

发布时先推送子仓库提交，再更新父仓库的子模块指针。历史验收记录见 [docs/verification.md](docs/verification.md)，其中旧版口径不代替当前档案。

## 开发

```bash
npm install
npm test
npm run typecheck
npm pack --dry-run
```

运行时无额外第三方依赖；Pi 宿主库声明为 peerDependencies。测试使用合成会话，不需要真实 API key。验证细节见 [docs/verification.md](docs/verification.md)。
