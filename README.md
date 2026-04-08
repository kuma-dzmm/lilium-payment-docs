# lilium-payment-docs

Lilium 支付与清算相关的公开文档仓库。

这个仓库只存放面向第三方接入方的公开材料，不存放内部实现文档、服务设计细节或与支付清算无关的项目文档。

## 范围

本仓库用于维护以下内容：

- Lilium 支付/清算接入文档
- 对外 API 定义，例如 OpenAPI 文档
- 批量清算 API 说明
- Hosted Checkout、Webhook、认证接入说明
- 面向第三方的示例请求、流程图和状态机说明

本仓库不用于维护以下内容：

- 内部服务实现说明
- 钱包内部数据模型说明
- 与支付清算无关的产品文档

## 当前对外入口

Lilium 对外入口分为两类：

- API 域名：`https://api.lilium.kuma.homes`
- 登录与 Checkout 域名：`https://lilium.kuma.homes`

预期对外能力包括：

- `OIDC` 登录
- `client_id + HMAC(client_secret)` 清算 API 鉴权
- Hosted Checkout
- OpenAPI 定义
- 批量清算 API
- Webhook 通知

## 目录约定

- `docs/`
  面向第三方的规范、设计稿、接入指南
- `openapi/`
  对外 API 定义文件，例如 `openapi.yaml`
- `examples/`
  请求示例、Webhook 验签示例、集成样例

## 当前文档

- [Lilium 开放清算 API 接入规范 v1](./docs/2026-04-09-lilium-open-clearing-api-design.md)

## 文档原则

- 以第三方接入视角编写
- 以公开契约为边界，不泄露内部实现细节
- 优先维护 OpenAPI 与接入规范的一致性
- 将流程图、状态机、错误模型写清楚
