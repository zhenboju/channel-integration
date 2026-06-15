# Nacos 路由配置模板

## 完整配置片段

以下配置需添加到 Nacos 的 `product-routes.yaml` 文件中的 `mappings` 节点下：

```yaml
  {{ROUTE_CONFIG}}:
    productCode: "{{上游产品编码}}"
    channelCode: "{{上游渠道编码}}"
    secret: "{{签名密钥}}"
    key: "{{备用密钥}}"
    saleId: "{{销售ID}}"
    crack: {{是否启用crack加密, true/false}}
    actions:
      createOrder:
        upstream:
          serviceName: "{{SERVICE_NAME}}"    # 必须与 @Service 注解一致
          action: "send"                    # 对应策略中 case "send"
          url: "{{SEND_API_URL}}"
      sendSms:
        upstream:
          serviceName: "{{SERVICE_NAME}}"
          action: "create"                 # 对应策略中 case "create"
          url: "{{VERIFY_API_URL}}"
      {{CHECK_ACTION_CONFIG}}
```

## 字段说明

| 字段 | 说明 | 必填 | 示例 |
|------|------|:---:|------|
| `{{ROUTE_CONFIG}}` | 产品表的 `route_config` 字段值，作为 Nacos 配置的 Key | ✅ | `bjkr_route` |
| `productCode` | 上游提供的产品编码 | ✅ | `2044006165842628608` |
| `channelCode` | 上游提供的渠道编码 | ✅ | `shanx-cmcc` |
| `secret` | 签名密钥（用于计算签名） | 签名时必填 | `qdgmwy31rxm...` |
| `key` | 备用密钥（部分渠道 sign 用 key 而非 secret） | 按需 | |
| `saleId` | 销售 ID（部分渠道用作 appId） | 按需 | `z314ysxkx...` |
| `crack` | 是否启用宁夏小键盘加密流程 | 默认 false | `true` |
| `serviceName` | **Spring Bean 名称，必须与 @Service 注解的 value 完全一致** | ✅ | `qi-ben-service` |
| `action` | 传给策略 `execute()` 的 action 参数 | ✅ | `send` / `create` / `check` |
| `url` | 上游接口完整 URL | ✅ | `https://api.example.com/api/sendSms` |

## 接口 → action 映射关系（固定规则）

| Controller 接口 | Nacos actions key | `upstream.action` 值 | 策略方法 |
|-----------------|-------------------|---------------------|---------|
| `POST /create` | `createOrder` | `"send"` | `sendOrder()` / `createOrder()` |
| `POST /verify` | `sendSms` | `"create"` | `verify()` |
| `POST /checkOrder` | `check` | `"check"` | `checkOrder()` |

> **注意**: Nacos 中 `actions` 的 key 名 `createOrder`/`sendSms`/`check` 是**固定不可变**的，由 `ChannelOrderServiceImpl` 硬编码。而 `upstream.action` 的值是由策略的 `switch(action)` 决定的，两者是不同层级的概念。

## 检查清单

- [ ] `{{ROUTE_CONFIG}}` 与产品表中的 `route_config` 字段一致
- [ ] `serviceName` 与策略类 `@Service("xxx")` 的 Bean 名完全一致（包括大小写）
- [ ] `action` 值与策略 `switch(action)` 的 case 值一致
- [ ] `url` 是完整的 HTTP/HTTPS 地址
- [ ] 如果接口文档中只有一个 URL（发送和校验用同一个 URL），两处填写相同 URL 即可
- [ ] `secret`/`key`/`saleId` 的值来自上游接口文档中的密钥信息
