# DTO 类模板参考

## 基本 DTO 模板

```java
package com.phone.phoneinterface.service.upstream.dto;

import lombok.Data;

/**
 * {{公司中文名}}上游请求 DTO
 */
@Data
public class {{DTO_CLASS_NAME}} {
    {{FIELDS}}
}
```

## DTO 设计规则

### 1. 命名规范
- 类名: `XxxDTO` (首字母大写驼峰)
- 字段名: camelCase
- 如有 JSON 字段名不一致，使用 `@JsonProperty("xxx")` 注解

### 2. 字段类型选择
| 上游字段类型 | Java 类型 |
|------------|----------|
| 字符串 | `String` |
| 整数 | `Integer` 或 `int` |
| 布尔值 | `Boolean` |
| 嵌套对象 | 静态内部类 |
| 嵌套数组 | `List<T>` 或 `Map<String, Object>` |

### 3. 何时不需要独立 DTO
如果上游请求体只有 3 个以内字段，且没有嵌套结构，可以考虑直接使用 `HashMap<String, Object>` 构建请求，无需创建独立 DTO 类（参考 `BeiJingKeRuiStrategy`）。

### 4. 嵌套结构示例
参考 `PopCornDTO`：
```java
@Data
public class PopCornDTO {
    private String apiUser;
    private String mobile;
    private Integer cid;
    private String sign;
    private String orderSn;
    private Extra extra;        // ← 嵌套对象

    @Data
    public static class Extra {  // ← 静态内部类
        private String source_url;
        private String platform_name;
        private String sms_code;
    }
}
```

## 现有 DTO 参考

| DTO | 特点 | 策略 |
|-----|------|------|
| `QiBenDTo` | 简单扁平结构，无嵌套，无 JSON 注解 | `QiBenStrategy` |
| `BjkrDTO` | 简单扁平，上报用 | `ChannelOrderServiceImpl` 回调 |
| `PopCornDTO` | 含 `Extra` 静态内部类 | `PopCornStrategy` |
| `ShengYangDTO` | 使用 `@Builder` 模式构建 | `ShengYangStrategy` |
| `LateAutumnSmsDTO` | 字段含有详细中文注释 | `LateAutumnStrategy` |

## 字段注释规范
每个字段加简要中文注释，说明含义和是否必填：
```java
/**
 * 手机号（必填）
 */
private String mobile;

/**
 * 产品ID - 由上游提供（必填）
 */
private String productCode;
```
