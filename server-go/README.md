# server-go

mir2x Go 服务端复刻工程。

当前阶段：服务端-only 方案评审。管理界面确定为 Web。尚未开始功能实现。

- 客户端不在交付范围
- 不依赖完整 mir2x_assets
- 测试使用 tools/mockclient 和 tools/fixturegen
- Lua 采用官方 Lua 5.4 + cgo 桥接，不翻译成 Go

- [完整复刻工作文档](docs/replication-work-document.md)
