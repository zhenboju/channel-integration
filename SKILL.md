---
name: channel-integration
description: >
  根据上游渠道接口文档，自动生成 DTO 类、Strategy 策略实现类 和 Nacos 路由配置。
  当用户提供新的上游渠道 API 文档时使用此 skill — 分析文档提取参数，按项目统一模板生成代码并写入文件。
  触发条件：用户提到"接入新渠道"、"上游接口文档"、"新增策略"、"channel integration"、提供 API 文档内容或文件。
---

# 渠道接入自动生成器

根据上游渠道接口文档，自动分析并生成符合项目统一模板的代码文件：
- **DTO 类**: `phone-interface/src/main/java/com/phone/phoneinterface/service/upstream/dto/XxxDTO.java`
- **Strategy 类**: `phone-interface/src/main/java/com/phone/phoneinterface/service/upstream/strategy/impl/XxxStrategy.java`
- **Nacos 配置片段**: 追加到 `product-routes.yaml` 的配置内容

---

## 工作流程

### 第一步：解析接口文档

从用户提供的接口文档（可能是粘贴的文本、PDF、Word 文档等）中提取以下信息：

#### 1.1 基本信息
| 提取项 | 说明 | 用途 |
|--------|------|------|
| 上游公司名称 | 中文名 + 英文标识 | 类名 `XxxStrategy`，`@Service("xxx-service")` |
| 接口基础 URL | 完整的基础地址 | `upstream.getUpstream().getUrl()` |
| 是否需要预校验(check) | 有/无独立的预校验接口 | 决定 `execute()` 中是否有 `"check"` case |

#### 1.2 认证/签名方式
从文档中识别签名算法，常见类型：
- **无签名**: 直接 POST JSON body 或 GET 请求
- **MD5 签名放在 Header**: 如北京科睿（appId + timestamp + ticket 放 Header）
- **MD5 签名拼在 Body**: 如 PopCorn（参数升序拼接 + apikey → MD5 → 放在请求体 sign 字段）
- **其他**: RSA、自定义 token 等

记录：签名算法、密钥来源（Nacos 中的 secret/key/saleId）、签名放置位置（Header/Body/QueryString）。

#### 1.3 发送验证码接口 (send)
| 提取项 | 说明 |
|--------|------|
| URL 路径 | 完整 URL |
| 请求方式 | POST JSON / GET / form-urlencoded |
| 请求字段映射 | 文档字段 → `ChannelOrderParam` 字段 或 Nacos 配置字段 |
| 响应成功判断 | 成功码（如 `200`、`"20000"`、`"1"`）和所在字段名（如 `code`） |
| 响应错误消息字段 | 错误信息所在字段名（如 `message`、`msg`、`error`） |
| 响应中上游订单号字段 | 如有，字段路径（如 `data.orderNo`） |

#### 1.4 验证码校验下单接口 (verify)
同上，额外注意：
- 是否需要验证码字段 (`smsCode`)
- 是否需要回传 `processUrl`/`chargeLink`
- 是否需要 IP、UA、appName 等投放信息

#### 1.5 预校验接口 (check) — 可选
如果有独立的预校验接口，提取同上信息。

#### 1.6 特殊业务逻辑
| 特殊逻辑 | 识别标志 | 处理方式 |
|---------|---------|---------|
| 延迟查询 | 文档提到"异步通知"、"回调"、"需轮询查询" | 增加 `scheduleDelayedQuery()` + Redis + `@Scheduled` |
| 宁夏小键盘加密 (crack) | 文档提到需要先加密验证码 | 复用 `PopCornStrategy.encryptCaptchaInput()` |
| 二次确认 (confirmHandler) | verify 成功后还需调另一个接口 | 如 `ShengYangStrategy.confirmHandler()` |
| 无冷静期回调 | 不需要等冷静期 | 由 `ChannelProductRelation.isNoCoolingChannel` 控制，策略中不需特殊处理 |
| 扣量 | 成功时按比例不通知渠道 | 已由 `DeductionHelper.shouldDeduct()` 统一处理 |
| 回调重试 | 需要回调下游渠道 | 如 `requestPostChannelCallback()` + `MAX_RETRIES` |

#### 1.7 参数校验字段
- send 接口必填字段列表
- verify 接口必填字段列表

---

### 第二步：确认分析结果

在生成代码之前，向用户确认分析结果。展示：

```
## 接口分析结果
- 上游名称：XXX
- Service Bean 名：xxx-service
- 策略类名：XxxStrategy
- DTO 类名：XxxDTO
- 接口数量：2（send + verify）/ 3（send + verify + check）
- 签名方式：无 / MD5-Header / MD5-Body / 其他
- 请求方式：POST JSON / GET
- 特殊流程：无 / 延迟查询 / crack加密 / confirmHandler

## 字段映射
### send 接口
| 文档字段 | 映射来源 | 
|---------|---------|
| mobile | param.getPhone() |
| productId | upstream.getProductCode() |
| ... | ... |

### verify 接口
...

请确认以上分析是否正确？
```

用户确认后再进行代码生成。

---

### 第三步：生成代码文件

#### 3.1 创建 DTO 类

**文件路径**: `phone-interface/src/main/java/com/phone/phoneinterface/service/upstream/dto/XxxDTO.java`

**模板参考** `references/dto-template.md`

关键规则：
- 使用 `@Data` 注解（Lombok）
- 字段名使用 camelCase，如有 JSON 特殊映射使用 `@JsonProperty`
- 如果上游请求体简单（3个字段以内），可以使用 HashMap，不需要独立 DTO（参考 `BeiJingKeRuiStrategy`）
- 如果有嵌套结构（如 `Extra`），使用静态内部类

#### 3.2 创建 Strategy 类

**文件路径**: `phone-interface/src/main/java/com/phone/phoneinterface/service/upstream/strategy/impl/XxxStrategy.java`

**模板参考** `references/strategy-template.md`

生成规则：
1. 类名：`XxxStrategy`，实现 `UpstreamServiceStrategy`
2. `@Service("xxx-service")`，名称需与 Nacos 的 `serviceName` 一致
3. `@Slf4j` 日志
4. 注入依赖（固定 7 项 + 可选 JedisCommands）
5. 构造函数注入 `OrderIdGenerator`（+ 可选的 `ResponseGenerator`）
6. `execute()` 方法：`switch(action)` 分流，默认 2 个 case: `"send"` 和 `"create"`
7. 标准私有方法（见模板）

**根据第一步分析结果调整的部分**:

| 分析结果 | 代码调整 |
|---------|---------|
| 有 check 接口 | `execute()` 增加 `case "check"`，增加 `checkOrder()` 方法 |
| 无签名 | `callUpstreamApi()` 直接用 `HttpUtil.postByJson(url, dto)` |
| MD5-Header 签名 | 增加签名构建代码，`callUpstreamApiWithHeaders()` |
| MD5-Body 签名 | 增加 `generateSignature()` 方法 |
| GET 请求 | 参数拼接到 URL QueryString，使用 `HttpUtil.get()` |
| 有延迟查询 | 增加 `scheduleDelayedQuery()` + `processDelayedQueries()` + Redis 相关 |
| 有 crack | 在 send 和 verify 中增加 crack 加密分支 |
| 有 confirmHandler | 在 verify 成功后增加二次调用 |
| 有回调重试 | 增加 `requestPostChannelCallback()` + `executeCallbackWithRetry()` |

#### 3.3 生成 Nacos 配置片段

输出 YAML 格式的配置片段，展示给用户并说明需要添加到 `product-routes.yaml`：

```yaml
# 需添加到 product-routes.yaml 的 mappings 下
<routeConfig>:                        # ← 产品表的 route_config 字段值（需用户提供）
  productCode: "<上游产品编码>"
  channelCode: "<上游渠道编码>"
  secret: "<签名密钥>"
  key: "<备用密钥>"
  saleId: "<销售ID>"
  crack: false
  actions:
    createOrder:
      upstream:
        serviceName: "xxx-service"    # ← 与 @Service 一致
        action: "send"                # ← 调用策略的 send 方法
        url: "<send接口URL>"
    sendSms:
      upstream:
        serviceName: "xxx-service"
        action: "create"             # ← 调用策略的 verify 方法
        url: "<verify接口URL>"
    check:                            # ← 仅当有预校验接口时
      upstream:
        serviceName: "xxx-service"
        action: "check"
        url: "<check接口URL>"
```

**重要提醒**：
- `routeConfig` 需要用户提供（对应 `Product` 表的 `route_config` 字段值）
- `serviceName` 必须与 `@Service("xxx-service")` 的 Bean 名称完全一致
- `action` 字段对应策略的 `switch(action)` case

---

### 第四步：验证检查

生成所有文件后，逐项检查：

**结构检查**：
- [ ] DTO 类在正确的目录下
- [ ] Strategy 类在正确的目录下
- [ ] 类名与文件名一致
- [ ] `@Service` Bean 名与 Nacos 配置的 `serviceName` 一致

**代码检查**：
- [ ] 所有导入正确（不要导入未使用的类）
- [ ] `execute()` 的 switch case 与 Nacos 的 action 值对应
- [ ] `buildAndSaveChannelOrder()` 使用 `orderIdGenerator.generate()` 生成 dzOrderCode
- [ ] send 方法返回 `R.ok({orderCode: dzOrderCode})`
- [ ] verify 方法通过 `param.getOrderCode()` 查找已有订单
- [ ] 错误响应通过 `handleErrorResponse()` 统一处理
- [ ] 交互日志通过 `recordCompleteInteraction()` 记录

**编译检查**：
- 运行 `mvn compile -pl phone-interface -am` 确保编译通过

---

## 参考现有实现

项目中已有 70+ 个策略实现，以下是最具代表性的参考：

| 策略 | Bean 名 | 签名方式 | 请求方式 | 特殊流程 | 参考价值 |
|------|---------|---------|---------|---------|---------|
| `QiBenStrategy` | `qi-ben-service` | 无 | POST JSON | 无 | **最简模板** |
| `BeiJingKeRuiStrategy` | `beijing-kerui-service` | MD5-Header | POST+Headers | 延迟查询、扣量 | Header 签名参考 |
| `PopCornStrategy` | `Pop-Corn-service` | MD5-Body | POST(原生) | crack加密 | Body 签名 + crack 参考 |
| `ShengYangStrategy` | `shengYang-mobile-service` | 无 | GET | confirmHandler、延迟查询 | GET 方式 + 二次确认参考 |

详细代码模板见 `references/strategy-template.md` 和 `references/dto-template.md`。

---

## 常见边界情况

1. **上游字段名与项目字段名冲突**: 在 DTO 中使用 `@JsonProperty` 映射
2. **上游需要省份编码**: 参考 `ShengYangStrategy` 中的 `ProvinceCode` 处理
3. **上游返回格式非标准 JSON**: 参考 `ShengYangStrategy.callUpstreamApi()` 中的正则提取
4. **上游要求 GET 请求但参数过多**: 使用 `LinkedHashMap` 保证参数顺序
5. **签名需要特定字符编码**: 明确使用 `StandardCharsets.UTF_8`
