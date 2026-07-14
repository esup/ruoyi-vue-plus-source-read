# ruoyi-common-json 模块分析文档

## 1. 模块概述

| 属性 | 说明 |
|------|------|
| 模块名 | ruoyi-common-json |
| 包路径 | `org.dromara.common.json` |
| 定位 | **JSON 序列化模块**，统一配置 Jackson 序列化行为，提供 JSON 工具类与 JSON 格式校验注解 |
| Java文件数 | 7 个 |
| 依赖关系 | 依赖 `ruoyi-common-core`、`jackson-databind`、`jackson-datatype-jsr310` |

### 1.1 Maven 依赖

```
ruoyi-common-core              ← 核心基础模块（SpringUtils、StringUtils、ObjectUtils）
jackson-databind               ← Jackson 核心序列化库
jackson-datatype-jsr310        ← Java 8 日期时间类型支持（LocalDateTime 等）
```

### 1.2 自动装配

通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册：

```
JacksonConfig  ← Jackson 序列化全局配置
```

---

## 2. 包结构总览

```
org.dromara.common.json
├── config/                        # 配置层
│   └── JacksonConfig.java         ← Jackson 全局配置（序列化/反序列化规则注册）
│
├── handler/                       # 序列化/反序列化处理器
│   ├── BigNumberSerializer.java   ← 大数字序列化（解决 JS 精度丢失）
│   └── CustomDateDeserializer.java ← Date 多格式反序列化
│
├── utils/                         # 工具类
│   └── JsonUtils.java             ← JSON 操作工具（序列化/反序列化/格式校验）
│
└── validate/                      # JSON 格式校验
    ├── JsonPattern.java           ← 校验注解
    ├── JsonPatternValidator.java  ← 校验器实现
    └── JsonType.java              ← JSON 类型枚举（OBJECT/ARRAY/ANY）
```

---

## 3. 核心组件详解

### 3.1 `JacksonConfig` — Jackson 全局配置

**职责**: 注册全局序列化/反序列化规则，确保前后端数据交互的一致性。

**装配顺序**: `@AutoConfiguration(before = JacksonAutoConfiguration.class)` — 在 Spring Boot 默认 Jackson 配置之前生效。

**注册的序列化规则：**

| 类型 | 处理器 | 方向 | 说明 |
|------|--------|------|------|
| `Long` / `long` | `BigNumberSerializer` | 序列化 | 超出 JS 安全范围时转为字符串 |
| `BigInteger` | `BigNumberSerializer` | 序列化 | 同上 |
| `BigDecimal` | `ToStringSerializer` | 序列化 | 始终转为字符串（避免精度丢失） |
| `LocalDateTime` | `LocalDateTimeSerializer/Deserializer` | 双向 | 格式：`yyyy-MM-dd HH:mm:ss` |
| `Date` | `CustomDateDeserializer` | 反序列化 | 支持多种日期格式 |

**时区配置：**

```java
builder.timeZone(TimeZone.getDefault());  // 使用服务器默认时区
```

---

### 3.2 `BigNumberSerializer` — 大数字序列化器

**解决的问题**: JavaScript 的 `Number` 类型遵循 IEEE 754 双精度浮点标准，安全整数范围为：

```
Number.MIN_SAFE_INTEGER = -9007199254740991  (-(2^53 - 1))
Number.MAX_SAFE_INTEGER =  9007199254740991  ( 2^53 - 1)
```

超出此范围的 Long 值（如雪花ID `1816870933832695808`）在前端会丢失精度。

**序列化策略：**

```java
// 安全范围内 → 输出数字
if (value > MIN_SAFE_INTEGER && value < MAX_SAFE_INTEGER) {
    gen.writeNumber(value);     // 输出: 12345
} else {
    gen.writeString(value);     // 输出: "1816870933832695808"
}
```

**使用单例模式：**

```java
public static final BigNumberSerializer INSTANCE = new BigNumberSerializer(Number.class);
```

---

### 3.3 `CustomDateDeserializer` — 多格式日期反序列化器

**解决的问题**: 前端传入的日期格式可能不统一（`yyyy-MM-dd`、`yyyy-MM-dd HH:mm:ss`、时间戳等），需要兼容多种格式。

**实现方式**: 委托 Hutool 的 `DateUtil.parse()` 自动识别日期格式：

```java
DateTime parse = DateUtil.parse(p.getText());  // 自动识别多种日期格式
if (ObjectUtils.isNull(parse)) {
    return null;
}
return parse.toJdkDate();
```

**支持的格式示例**:
- `2024-01-15`
- `2024-01-15 14:30:00`
- `2024/01/15`
- `2024年01月15日`
- 时间戳字符串

---

### 3.4 `JsonUtils` — JSON 工具类

**核心设计**: 通过 `SpringUtils.getBean(ObjectMapper.class)` 获取 Spring 容器中已配置的 `ObjectMapper` 实例，确保序列化行为与全局配置一致。

**方法清单：**

| 方法 | 说明 |
|------|------|
| `toJsonString(Object)` | 对象 → JSON 字符串 |
| `parseObject(String, Class<T>)` | JSON 字符串 → 指定类型对象 |
| `parseObject(byte[], Class<T>)` | 字节数组 → 指定类型对象 |
| `parseObject(String, TypeReference<T>)` | JSON 字符串 → 复杂泛型类型（如 `Map<String, Object>`） |
| `parseMap(String)` | JSON 字符串 → `Dict` 对象（Hutool 动态字典） |
| `parseArrayMap(String)` | JSON 字符串 → `List<Dict>` |
| `parseArray(String, Class<T>)` | JSON 字符串 → `List<T>` |
| `isJson(String)` | 判断是否为合法 JSON（对象或数组） |
| `isJsonObject(String)` | 判断是否为 JSON 对象（`{}`） |
| `isJsonArray(String)` | 判断是否为 JSON 数组（`[]`） |

**异常处理策略**: 所有方法内部捕获 `IOException`/`JsonProcessingException` 后包装为 `RuntimeException` 抛出，简化调用方代码。

**空值处理**: 所有方法对 `null`/空字符串进行前置判断，避免 NPE。

**`parseMap` 特殊处理**: 捕获 `MismatchedInputException` 时返回 `null` 而非抛异常，因为"不是 JSON"不算错误。

---

## 4. JSON 格式校验体系

### 4.1 `@JsonPattern` — 校验注解

```java
@Documented
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = JsonPatternValidator.class)
public @interface JsonPattern {
    JsonType type() default JsonType.ANY;        // 限制 JSON 类型
    String message() default "不是有效的 JSON 格式";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 4.2 `JsonType` — JSON 类型枚举

| 枚举值 | 说明 | 示例 |
|--------|------|------|
| `OBJECT` | JSON 对象 | `{"name":"张三"}` |
| `ARRAY` | JSON 数组 | `[1,2,3]` |
| `ANY` | 对象或数组均可 | 以上两种都合法 |

### 4.3 `JsonPatternValidator` — 校验器

```java
public boolean isValid(String value, ConstraintValidatorContext context) {
    if (StringUtils.isBlank(value)) {
        return true;  // 空值交给 @NotBlank/@NotNull 控制
    }
    return switch (jsonType) {
        case ANY    -> JsonUtils.isJson(value);
        case OBJECT -> JsonUtils.isJsonObject(value);
        case ARRAY  -> JsonUtils.isJsonArray(value);
    };
}
```

**使用示例：**

```java
// 限制必须是 JSON 对象
@JsonPattern(type = JsonType.OBJECT, message = "必须是JSON对象")
private String config;

// 限制必须是 JSON 数组
@JsonPattern(type = JsonType.ARRAY)
private String items;

// 任意 JSON 均可
@JsonPattern
private String data;
```

---

## 5. 整体工作流程

```
HTTP 请求/响应
    │
    ▼
Jackson 序列化/反序列化
    │
    ├─ 序列化（Java → JSON）
    │   ├─ Long/BigInteger → BigNumberSerializer（安全范围判断）
    │   ├─ BigDecimal → ToStringSerializer（始终字符串）
    │   └─ LocalDateTime → "yyyy-MM-dd HH:mm:ss"
    │
    └─ 反序列化（JSON → Java）
        ├─ LocalDateTime ← "yyyy-MM-dd HH:mm:ss"
        └─ Date ← CustomDateDeserializer（多格式兼容）
    │
    ▼
业务代码使用 JsonUtils
    ├─ 对象 ↔ JSON 字符串转换
    ├─ JSON 格式校验
    └─ Redis/消息队列 数据序列化
    │
    ▼
Bean Validation 校验
    └─ @JsonPattern → JsonPatternValidator → JsonUtils.isJson/isJsonObject/isJsonArray
```

---

## 6. 设计亮点

### 6.1 前后端精度对齐

`BigNumberSerializer` 精准解决前后端交互中的经典问题：

```
后端返回: 1816870933832695808  (Long)
前端接收: 1816870933832695810  (精度丢失!)

解决后:
后端返回: "1816870933832695808"  (String)
前端接收: "1816870933832695808"  (精度保持)
```

### 6.2 日期格式兼容

`CustomDateDeserializer` 委托 Hutool `DateUtil.parse()` 自动识别格式，前端无需严格对齐日期格式。

### 6.3 全局配置复用

`JsonUtils` 通过 `SpringUtils.getBean(ObjectMapper.class)` 复用容器中已配置好的 `ObjectMapper`，确保：
- 工具类序列化行为与 HTTP 响应完全一致
- 无需重复创建 `ObjectMapper` 实例
- 配置变更只需修改 `JacksonConfig`

### 6.4 声明式 JSON 校验

`@JsonPattern` 注解将 JSON 格式校验从业务代码中抽离，符合 Bean Validation 规范，可与 `@NotNull`/`@NotBlank` 组合使用。

### 6.5 单例序列化器

`BigNumberSerializer.INSTANCE` 采用单例模式，避免重复创建实例，减少内存开销。

---

## 7. 模块依赖关系

```
ruoyi-common-json
│
├── 依赖
│   ├── ruoyi-common-core      ← SpringUtils、StringUtils、ObjectUtils
│   ├── jackson-databind       ← ObjectMapper、序列化/反序列化核心
│   └── jackson-datatype-jsr310 ← Java 8 日期时间支持
│
└── 被依赖
    └── ruoyi-admin            ← 主启动模块引入后全局生效
    └── 其他需要 JSON 处理的模块
```

---

## 8. 总结

`ruoyi-common-json` 是一个**职责聚焦**的 JSON 序列化模块，仅用 7 个 Java 文件实现了以下能力：

| 能力 | 实现 |
|------|------|
| 大数字精度保护 | `BigNumberSerializer` 超出 JS 安全范围自动转字符串 |
| 日期格式统一 | `LocalDateTime` 固定 `yyyy-MM-dd HH:mm:ss`，`Date` 多格式兼容 |
| JSON 工具方法 | `JsonUtils` 提供序列化/反序列化/格式校验一站式方法 |
| 声明式 JSON 校验 | `@JsonPattern` 注解 + `JsonPatternValidator` 校验器 |
| 全局配置统一 | `JacksonConfig` 确保所有序列化行为一致 |

该模块的核心价值在于：**解决前后端数据交互中的精度、格式、校验三大问题**，为整个系统提供可靠的 JSON 处理基础设施。
