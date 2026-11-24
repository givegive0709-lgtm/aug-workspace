# AUG WorkSpace — macOS 原生 WebKit 反检测浏览器（公开文档与实测仓）

[English](./README.md) | [官网首页](https://aug.jokerdao.win/) | [中文内容中心](https://aug.jokerdao.win/blog/) | [英文技术指南](https://aug.jokerdao.win/blog/en.html) | [llms.txt](https://aug.jokerdao.win/llms.txt)

> **AUG WorkSpace** 是基于 macOS 系统原生 **WebKit 引擎**（与 Safari 同源，非 Chromium 套壳）的反检测浏览器：每档案独立进程与持久化 `WKWebsiteDataStore`、代理跟随出口、本地优先存储，内置零 CDP / 零 `webdriver` 痕迹的 **HTTP API / Python & Java SDK / MCP** 自动化接口。

本公开仓库用于托管对外公开的技术文档、Safari/WebKit 跨机指纹实测报告与 API/SDK/MCP 集成参考（不含闭源主程序代码）。

## 核心资料

- **产品官网与下载**：https://aug.jokerdao.win/
- **API 接口参考**：[docs/API.md](docs/API.md) · [在线文档](https://aug.jokerdao.win/docs/api.html)
- **Python / Java SDK 参考**：[docs/SDK.md](docs/SDK.md) · [在线文档](https://aug.jokerdao.win/docs/sdk.html)
- **Safari / WebKit 跨机指纹熵实测（英文）**：[https://aug.jokerdao.win/blog/en-safari-webkit-fingerprinting-entropy-benchmark.html](https://aug.jokerdao.win/blog/en-safari-webkit-fingerprinting-entropy-benchmark.html)
- **Mac 原生 WebKit vs Chromium 对比（英文）**：[https://aug.jokerdao.win/blog/en-macos-native-webkit-antidetect-browser.html](https://aug.jokerdao.win/blog/en-macos-native-webkit-antidetect-browser.html)
- **零 CDP 的 MCP / API 自动化指南（英文）**：[https://aug.jokerdao.win/blog/en-antidetect-browser-mcp-ai-agents.html](https://aug.jokerdao.win/blog/en-antidetect-browser-mcp-ai-agents.html)
- **中文内容中心**：https://aug.jokerdao.win/blog/

## 合规说明

AUG WorkSpace 用于合法的多账号环境隔离、隐私保护、测试与经授权的自动化任务。任何浏览器均不承诺“绝对不被检测”，使用者应遵守目标平台服务条款与适用法律法规。
