# Strategy 策略类代码模板

## 完整模板（最简版 - 无签名、POST JSON、2个action）

以 `QiBenStrategy` 为基准模板，用 `{{占位符}}` 标记需要替换的部分。

```java
package com.phone.phoneinterface.service.upstream.strategy.impl;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.phone.phoneinterface.config.ProductRouteConfig;
import com.phone.phoneinterface.manager.MediumThickManager;
import com.phone.phoneinterface.manager.OrderManager;
import com.phone.phoneinterface.mapper.ChannelMapper;
import com.phone.phoneinterface.mapper.ChannelOrderDetailMapper;
import com.phone.phoneinterface.mapper.ChannelOrderMapper;
import com.phone.phoneinterface.mapper.ChannelProductRelationMapper;
import com.phone.phoneinterface.request.ChannelOrderParam;
import com.phone.phoneinterface.result.AuthResult;
import com.phone.phoneinterface.service.upstream.dto.{{DTO_CLASS_NAME}};
import com.phone.phoneinterface.service.upstream.strategy.UpstreamServiceStrategy;
import fire.insurance.common.domain.Channel;
import fire.insurance.common.domain.ChannelOrder;
import fire.insurance.common.domain.ChannelOrderDetails;
import fire.insurance.common.enums.OrderStatus;
import fire.insurance.common.enums.QueryStatus;
import fire.insurance.common.enums.RecordType;
import fire.insurance.common.execption.BusinessException;
import fire.insurance.common.tencent.StringUtils;
import fire.insurance.common.utils.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.io.IOException;
import java.util.*;
import java.util.concurrent.Executor;

/**
 * {{公司中文名}}业务服务调用策略
 *
 * <p>对应Nacos配置中的 serviceName: "{{SERVICE_NAME}}"</p>
 */
@Service("{{SERVICE_NAME}}")
@Slf4j
public class {{STRATEGY_CLASS_NAME}} implements UpstreamServiceStrategy {
    @Resource
    private ChannelOrderMapper channelOrderMapper;

    @Resource
    private ChannelOrderDetailMapper channelOrderDetailMapper;

    @Resource
    private OrderManager orderManager;

    private final OrderIdGenerator orderIdGenerator;

    @Resource
    private ChannelProductRelationMapper relationMapper;

    @Resource
    private ChannelMapper channelMapper;

    @Resource
    private MediumThickManager mediumThickManager;

    @Autowired
    @Qualifier("asyncOrderExecutor")
    private Executor asyncOrderExecutor;

    // 通过构造函数注入
    public {{STRATEGY_CLASS_NAME}}(OrderIdGenerator orderIdGenerator) {
        this.orderIdGenerator = orderIdGenerator;
    }

    @Override
    public R<HashMap<String, Object>> execute(
            String action, ChannelOrderParam param,
            ProductRouteConfig.ProductRouteInfo upstream, AuthResult authResult) {
        switch (action) {
            case "send":
                return sendOrder(param, upstream, authResult);
            case "create":
                return verify(param, upstream, authResult);
            {{CHECK_CASE}}
            default:
                throw new BusinessException("不支持的动作方法: " + action, 500);
        }
    }

    // ==================== send 方法 ====================

    private R<HashMap<String, Object>> sendOrder(ChannelOrderParam param,
                                                  ProductRouteConfig.ProductRouteInfo upstream,
                                                  AuthResult authResult) {
        // 1. 参数校验
        R<HashMap<String, Object>> validationResult = validateSendOrderParams(param);
        if (validationResult != null) {
            return validationResult;
        }

        try {
            // 2. 构建并保存订单记录
            ChannelOrder channelOrder = buildAndSaveChannelOrder(param, authResult);

            // 记录渠道请求
            recordCompleteInteraction(channelOrder.getDzOrderCode(),
                    null, param, null, RecordType.CHANNEL_REQUEST);

            // 订单通用拦截
            R<HashMap<String, Object>> interceptionResult =
                    orderManager.checkOrderInterception(param, authResult, channelOrder);
            if (interceptionResult != null) {
                return interceptionResult;
            }

            {{CRACK_BRANCH_SEND}}

            // 3. 构建请求DTO并调用上游API
            {{DTO_CLASS_NAME}} dto = buildSendDTO(param, upstream, authResult{{EXTRA_DTO_PARAMS}});

            {{SIGN_CODE_SEND}}

            JSONObject response = callUpstreamApi(
                    upstream.getUpstream().getUrl(), dto, channelOrder.getDzOrderCode());

            // 4. 处理响应
            R<HashMap<String, Object>> result = processSmsResponse(
                    channelOrder.getUuid(), response, channelOrder.getDzOrderCode());

            // 更新渠道请求记录的响应
            updateInteractionResponse(channelOrder.getDzOrderCode(),
                    RecordType.CHANNEL_REQUEST, result);

            return result;

        } catch (BusinessException e) {
            throw e;
        } catch (IOException e) {
            throw new BusinessException("{{公司中文名}}-发送局方接口响应失败: " + e.getMessage(), 500);
        } catch (Exception e) {
            log.error("{{公司中文名}}-发送验证码时发生未知异常", e);
            throw new BusinessException("发送验证码时发生未知异常: " + e.getMessage(), 500);
        }
    }

    // ==================== verify 方法 ====================

    private R<HashMap<String, Object>> verify(ChannelOrderParam param,
                                               ProductRouteConfig.ProductRouteInfo upstream,
                                               AuthResult authResult) {
        // 1. 参数校验
        R<HashMap<String, Object>> validationResult = validateVerifyParams(param);
        if (validationResult != null) {
            return validationResult;
        }

        // 2. 查询订单
        ChannelOrder channelOrder = channelOrderMapper.selectByDzOrderCode(param.getOrderCode());
        if (channelOrder == null) {
            log.error("订单不存在，订单号: {}", param.getOrderCode());
            return logAndReturnError("订单不存在: " + param.getOrderCode());
        }

        try {
            // 3. 更新订单信息
            updateChannelOrder(param, channelOrder);

            // 记录渠道请求
            recordCompleteInteraction(channelOrder.getDzOrderCode(),
                    null, param, null, RecordType.CHANNEL_REQUEST);

            {{CRACK_BRANCH_VERIFY}}

            // 4. 构建验证请求DTO并调用上游API
            {{DTO_CLASS_NAME}} dto = buildVerifyDTO(param, channelOrder, upstream{{EXTRA_DTO_PARAMS_VERIFY}});

            {{SIGN_CODE_VERIFY}}

            JSONObject response = callUpstreamApi(
                    upstream.getUpstream().getUrl(), dto, channelOrder.getDzOrderCode());

            // 5. 处理验证响应
            R<HashMap<String, Object>> result = processVerifyResponse(
                    param.getOrderCode(), response, channelOrder.getProductName()
                    {{EXTRA_VERIFY_PARAMS}});

            // 更新渠道请求记录的响应
            updateInteractionResponse(channelOrder.getDzOrderCode(),
                    RecordType.CHANNEL_REQUEST, result);

            asyncOrderExecutor.execute(() -> {
                if (result.isSuccess()) {
                    specialChannel(channelOrder);
                }
            });

            {{CONFIRM_HANDLER_CALL}}

            return result;
        } catch (BusinessException e) {
            throw e;
        } catch (IOException e) {
            throw new BusinessException("发送局方接口响应失败: " + e.getMessage(), 500);
        } catch (Exception e) {
            log.error("验证订单时发生未知异常，订单号: {}", param.getOrderCode(), e);
            throw new BusinessException("服务器处理失败: " + e.getMessage(), 500);
        }
    }

    // ==================== check 方法（可选）====================
    {{CHECK_METHOD}}

    // ==================== DTO 构建方法 ====================

    private {{DTO_CLASS_NAME}} buildSendDTO(
            ChannelOrderParam param, ProductRouteConfig.ProductRouteInfo upstream,
            AuthResult authResult{{EXTRA_DTO_PARAM_DEF}}) {
        {{DTO_CLASS_NAME}} dto = new {{DTO_CLASS_NAME}}();
        // TODO: 根据接口文档填写字段映射
        // dto.setXxx(param.getPhone());
        // dto.setYyy(upstream.getProductCode());
        // dto.setZzz(upstream.getChannelCode());
        return dto;
    }

    private {{DTO_CLASS_NAME}} buildVerifyDTO(
            ChannelOrderParam param, ChannelOrder channelOrder,
            ProductRouteConfig.ProductRouteInfo upstream{{EXTRA_DTO_PARAM_DEF_VERIFY}}) {
        {{DTO_CLASS_NAME}} dto = new {{DTO_CLASS_NAME}}();
        // TODO: 根据接口文档填写字段映射
        // dto.setMobile(channelOrder.getPhone());
        // dto.setProductCode(upstream.getProductCode());
        // dto.setSmsCode(param.getSmsCode());
        return dto;
    }

    // ==================== 响应处理方法 ====================

    private R<HashMap<String, Object>> processSmsResponse(
            String uuid, JSONObject response, String dzOrderCode) {
        ChannelOrder updateData = new ChannelOrder();
        updateData.setUuid(uuid);
        updateData.setExtData(response.toJSONString());

        if (!{{SUCCESS_CONDITION}}) {
            String errorMsg = response.getString("{{ERROR_FIELD}}");
            log.error("{{公司中文名}}-上游返回错误，UUID: {}, 错误信息: {}", uuid, errorMsg);
            return handleErrorResponse(updateData, errorMsg, true);
        }

        {{EXTRACT_UPSTREAM_ORDER_NO_SEND}}

        HashMap<String, Object> result = new HashMap<>();
        result.put("orderCode", dzOrderCode);

        updateData.setMessage("发送验证码成功");
        updateData.setStatus(OrderStatus.SMS_SENT.getCode());
        channelOrderMapper.updateByUuid(updateData);

        log.info("验证码发送成功，UUID: {}, 我方订单号: {}", uuid, dzOrderCode);
        return R.ok(result);
    }

    private R<HashMap<String, Object>> processVerifyResponse(
            String orderCode, JSONObject response, String productName
            {{EXTRA_VERIFY_PARAM_DEF}}) {
        ChannelOrder updateData = new ChannelOrder();
        updateData.setDzOrderCode(orderCode);
        updateData.setExtData(response.toJSONString());

        if (!{{SUCCESS_CONDITION_VERIFY}}) {
            String errorMsg = response.getString("{{ERROR_FIELD_VERIFY}}");
            log.error("产品[{}]订单验证失败，订单号: {}, 错误信息: {}",
                    productName, orderCode, errorMsg);
            return handleErrorResponse(updateData, errorMsg, false);
        }

        {{EXTRACT_UPSTREAM_ORDER_NO_VERIFY}}

        HashMap<String, Object> result = new HashMap<>();
        result.put("status", OrderStatus.PROCESSING.getDesc());

        updateData.setMessage(OrderStatus.SUCCESS_PROCESSING.getDesc());
        updateData.setStatus(OrderStatus.SUCCESS_PROCESSING.getCode());
        updateData.setQueryStatus(QueryStatus.QUERIED.getCode());
        channelOrderMapper.updateByDzOrderCode(updateData);

        log.info("订单处理中，订单号: {}", orderCode);

        {{SCHEDULE_DELAYED_QUERY}}

        return R.ok(result);
    }

    // ==================== 通用方法（固定不变）====================

    private ChannelOrder buildAndSaveChannelOrder(ChannelOrderParam param, AuthResult authResult) {
        ChannelOrder channel = BeanMapperUtil.map(param, ChannelOrder.class);
        channel.setIsNoCoolingChannel(
                relationMapper.findByChannelAndProduct(
                        authResult.getChannel().getId(), authResult.getProduct().getId()
                ).getIsNoCoolingChannel()
        );
        channel.setUuid(UUID.randomUUID().toString());
        channel.setProductName(authResult.getProduct().getProductName());
        channel.setDzOrderCode(orderIdGenerator.generate());
        channel.setChannelName(authResult.getChannel().getChannelName());

        if (StringUtils.isNotEmpty(param.getExtraInfo())) {
            channel.setExtraInfo(param.getExtraInfo());
        }

        channelOrderMapper.insert(channel);
        log.info("订单创建成功，订单号: {}, UUID: {}", channel.getDzOrderCode(), channel.getUuid());
        return channel;
    }

    private void updateChannelOrder(ChannelOrderParam param, ChannelOrder channelOrder) {
        ChannelOrder updateData = new ChannelOrder();
        updateData.setSmsCode(param.getSmsCode());
        updateData.setProductCode(channelOrder.getProductCode());
        updateData.setCallback(param.getCallback());
        updateData.setUnsubscribeCallbackUrl(param.getUnsubscribeCallbackUrl());
        updateData.setDzOrderCode(channelOrder.getDzOrderCode());
        if (StringUtils.isNotEmpty(param.getExtraInfo())) {
            updateData.setExtraInfo(param.getExtraInfo());
        }
        channelOrderMapper.updateByDzOrderCode(updateData);
        log.info("订单信息已更新，订单号: {}", param.getOrderCode());
    }

    private R<HashMap<String, Object>> handleErrorResponse(
            ChannelOrder channelOrder, String errorMsg, boolean isUuidUpdate) {
        channelOrder.setMessage(errorMsg);
        channelOrder.setStatus(OrderStatus.FAILED.getCode());
        channelOrder.setQueryStatus(QueryStatus.QUERIED.getCode());
        if (isUuidUpdate) {
            channelOrderMapper.updateByUuid(channelOrder);
        } else {
            channelOrderMapper.updateByDzOrderCode(channelOrder);
        }
        return R.error(errorMsg);
    }

    private R<HashMap<String, Object>> logAndReturnError(String errorMsg) {
        log.error("{{公司中文名}}下游请求参数校验失败：{}", errorMsg);
        return R.error(errorMsg);
    }

    private void specialChannel(ChannelOrder channelOrder) {
        Channel channel = channelMapper.selectByChannelCode(channelOrder.getChannelCode());
        ChannelOrder updateChannelOrder = mediumThickManager.specialChannel(channelOrder, channel);
        if (updateChannelOrder != null) {
            channelOrderMapper.updateByDzOrderCode(updateChannelOrder);
        }
    }

    // ==================== 参数校验方法 ====================

    private R<HashMap<String, Object>> validateSendOrderParams(ChannelOrderParam param) {
        if (Objects.isNull(param)) {
            return logAndReturnError("参数不能为空");
        }
        String error = ValidateUtil.validateFields(param,
                {{SEND_REQUIRED_FIELDS}}
        );
        return error != null ? logAndReturnError(error) : null;
    }

    private R<HashMap<String, Object>> validateVerifyParams(ChannelOrderParam param) {
        if (Objects.isNull(param)) {
            return logAndReturnError("参数不能为空");
        }
        String error = ValidateUtil.validateFields(param,
                {{VERIFY_REQUIRED_FIELDS}}
        );
        return error != null ? logAndReturnError(error) : null;
    }

    // ==================== API 调用方法 ====================

    {{CALL_UPSTREAM_API_METHOD}}

    // ==================== 日志记录方法 ====================

    private void recordCompleteInteraction(String dzOrderCode, String requestUrl,
                                           Object requestParams, Object responseData,
                                           RecordType recordType) {
        try {
            ChannelOrderDetails details = new ChannelOrderDetails();
            details.setDzOrderCode(dzOrderCode);
            details.setRequestUrl(requestUrl);
            details.setRequestParams(requestParams != null ? JSON.toJSONString(requestParams) : null);
            details.setResponseData(responseData != null ? JSON.toJSONString(responseData) : null);
            details.setRecordType(recordType.getCode());
            details.setCreateTime(new Date());
            channelOrderDetailMapper.insert(details);
            log.info("记录完整交互信息，订单号: {}, 类型: {}", dzOrderCode, recordType.getDesc());
        } catch (Exception e) {
            log.error("记录交互信息失败，订单号: {}", dzOrderCode, e);
        }
    }

    private void updateInteractionResponse(String dzOrderCode,
                                           RecordType interactionType,
                                           Object responseData) {
        try {
            ChannelOrderDetails update = new ChannelOrderDetails();
            update.setDzOrderCode(dzOrderCode);
            update.setRecordType(interactionType.getCode());
            update.setResponseData(JSON.toJSONString(responseData));
            channelOrderDetailMapper.updateResponseByOrderCodeAndType(update);
        } catch (Exception e) {
            log.error("更新交互记录响应失败，订单号: {}", dzOrderCode, e);
        }
    }

    // ==================== 特殊流程方法 ====================
    {{SPECIAL_FLOW_METHODS}}
}
```

---

## 占位符说明

| 占位符 | 说明 | 示例值 |
|--------|------|--------|
| `{{STRATEGY_CLASS_NAME}}` | 策略类名 | `QiBenStrategy` |
| `{{SERVICE_NAME}}` | Spring Bean 名 | `qi-ben-service` |
| `{{DTO_CLASS_NAME}}` | DTO 类名 | `QiBenDTo` |
| `{{公司中文名}}` | 上游公司中文名 | `启奔` |
| `{{SEND_REQUIRED_FIELDS}}` | send 必填字段 | `"phone", "channelCode", "productCode"` |
| `{{VERIFY_REQUIRED_FIELDS}}` | verify 必填字段 | `"orderCode", "channelCode", "productCode", "smsCode"` |
| `{{SUCCESS_CONDITION}}` | send 成功判断条件 | `SUCCESS_CODE.equals(response.getInteger("code"))` |
| `{{ERROR_FIELD}}` | send 错误消息字段 | `message` / `msg` |
| `{{CHECK_CASE}}` | check action 的 case | `case "check": return checkOrder(...)` 或空 |
| `{{CRACK_BRANCH_SEND}}` | send 中的 crack 加密分支 | 见下方说明 |
| `{{SIGN_CODE_SEND}}` | send 中的签名代码 | 见下方说明 |
| `{{CALL_UPSTREAM_API_METHOD}}` | API 调用方法实现 | 见下方说明 |
| `{{DELAYED_QUERY_CODE}}` | 延迟查询代码块 | 见 `BeiJingKeRuiStrategy` |

---

## 变体片段

### 1. 无签名 + POST JSON（最简单）

```java
// {{CALL_UPSTREAM_API_METHOD}}
private JSONObject callUpstreamApi(String url, Object requestDto, String dzOrderCode) throws IOException {
    JSONObject requestParam = (JSONObject) JSON.toJSON(requestDto);
    log.info("调用上游接口，URL: {}, 请求参数: {}", url, requestParam);
    String responseContent = HttpUtil.postByJson(url, requestParam).get("content").toString();
    log.info("上游接口响应: {}", responseContent);
    recordCompleteInteraction(dzOrderCode, url, requestParam, responseContent, RecordType.UPSTREAM_REQUEST);
    return JSON.parseObject(responseContent);
}
```

### 2. MD5 签名放 Header（如 BeiJingKeRui）

```java
// {{SIGN_CODE_SEND}} 和 {{SIGN_CODE_VERIFY}}
String appId = StringUtils.isNotBlank(upstream.getSaleId()) ? upstream.getSaleId() : DEFAULT_APPID;
String secret = StringUtils.isNotBlank(upstream.getSecret()) ? upstream.getSecret() : DEFAULT_SECRET;
long timestamp = System.currentTimeMillis();
String signStr = "APP_ID" + appId + "TIMESTAMP" + timestamp + secret;
String ticket = DigestUtils.md5DigestAsHex(signStr.getBytes());

HashMap<String, String> headers = new HashMap<>();
headers.put("appId", appId);
headers.put("timestamp", String.valueOf(timestamp));
headers.put("ticket", ticket);
headers.put("orgCode", upstream.getChannelCode());

// {{CALL_UPSTREAM_API_METHOD}}
private JSONObject callUpstreamApiWithHeaders(String url, Object requestDto,
                                               Map<String, String> headers, String dzOrderCode) throws IOException {
    JSONObject requestParam = (JSONObject) JSON.toJSON(requestDto);
    log.info("调用上游接口，URL: {}, 请求头: {}, 请求参数: {}", url, headers, requestParam);
    String responseContent = HttpUtil.postByJson(url, requestParam, headers).get("content").toString();
    log.info("上游接口响应: {}", responseContent);
    if (dzOrderCode != null) {
        recordCompleteInteraction(dzOrderCode, url, requestParam, responseContent, RecordType.UPSTREAM_REQUEST);
    }
    return JSON.parseObject(responseContent);
}
```

### 3. MD5 签名放 Body（如 PopCorn）

`TreeMap` 升序拼接参数（extra 不参与签名） + `&apikey=` + MD5 → 设置 `dto.setSign(sign)`

### 4. GET 请求（如 ShengYang）

DTO 字段转 URL QueryString 参数，使用 `HttpUtil.get(fullUrl)`

### 5. 延迟查询

需要额外注入：
```java
@Resource
private JedisCommands jedisCluster;
```

增加方法：
- `scheduleDelayedQuery()` - 存入 Redis SortedSet
- `processDelayedQueries()` + `@Scheduled(cron = "0 * * * * ?")` - 定时扫描查询

### 6. crack 加密分支

```java
if (Boolean.TRUE.equals(upstream.getCrack())) {
    String encryptedData = encryptCaptchaInput(param.getPhone(), param.getSmsCode(), upstream);
    JSONObject resJson = callUpstreamApiByGet(CRACK_URL, encryptedData, channelOrder.getDzOrderCode());
    if (resJson.getInteger("code") != 0) {
        return handleErrorResponse(channelOrder, resJson.getString("message"), false);
    }
}
```

### 7. confirmHandler

在 verify 成功返回前，额外调用最终办理接口。

---

## 注意事项

1. 所有日志中的中文标识使用上游公司简称
2. 构造函数注入的 `OrderIdGenerator` 必须与字段 `final` 修饰
3. `validateSendOrderParams` 的必填字段中 `phone`, `channelCode`, `productCode` 是基础必填
4. `validateVerifyParams` 的必填字段中 `orderCode`, `channelCode`, `productCode`, `smsCode` 是基础必填
5. 异常处理三层：`BusinessException`（已知） → `IOException`（网络） → `Exception`（未知）
