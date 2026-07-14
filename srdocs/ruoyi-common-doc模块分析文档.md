# ruoyi-common-doc 模块分析文档

## 1. 模块概述

| 属性 | 说明 |
|------|------|
| 模块名 | ruoyi-common-doc |
| 包路径 | `org.dromara.common.doc` |
| 定位 | **接口文档模块**，基于 SpringDoc OpenAPI 自动生成 API 文档，并增强 Sa-Token 权限注解的可视化展示 |
| Java文件数 | 7 个 |
| 依赖关系 | 依赖 `ruoyi-common-core`、`springdoc-openapi-starter-webmvc-api`、`therapi-runtime-javadoc` |

### 1.1 Maven 依赖

```
ruoyi-common-core                         ← 核心基础模块
springdoc-openapi-starter-webmvc-api      ← SpringDoc OpenAPI（Swagger UI 生成）
therapi-runtime-javadoc                   ← 运行时 JavaDoc 读取（编译期提取注释）
jackson-module-kotlin                     ← Jackson Kotlin 模块（兼容性支持）
```

### 1.2 自动装配

通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册：

```
SpringDocConfig  ← 唯一自动配置入口
```

---

## 2. 包结构总览

```
org.dromara.common.doc
├── config/                          # 配置层
│   ├── SpringDocConfig.java         ← 自动配置主类（OpenAPI Bean、路径处理、解析器注册）
│   └── properties/
│       └── SpringDocProperties.java ← YML配置属性映射
│
├── core/                            # 核心逻辑层
│   ├── model/
│   │   └── SaTokenSecurityMetadata.java  ← Sa-Token权限元数据模型
│   └── resolver/                    # 解析器体系
│       ├── JavadocResolver.java                ← 解析器接口（策略接口）
│       ├── AbstractMetadataJavadocResolver.java ← 抽象元数据解析器（模板方法）
│       └── SaTokenAnnotationMetadataJavadocResolver.java ← Sa-Token权限注解解析器
│
└── handler/                         # 处理器层
    └── OpenApiHandler.java          ← 自定义OpenAPI处理器（继承OpenAPIService）
```

---

## 3. 核心组件详解

### 3.1 `SpringDocConfig` — 自动配置主类

**职责**: 初始化 OpenAPI 文档对象、注册自定义处理器、处理上下文路径拼接、注册权限解析器。

**关键 Bean：**

| Bean | 说明 |
|------|------|
| `openApi()` | 创建 `OpenAPI` 对象，填充文档基本信息（标题/描述/版本/联系人）、安全方案、标签、路径 |
| `openApiBuilder()` | 替换默认的 `OpenAPIService`，注入自定义的 `OpenApiHandler` 与 `JavadocResolver` 列表 |
| `openApiCustomizer()` | 对所有 API 路径前置拼接 `context-path`，解决路径重复拼接问题 |
| `saTokenAnnotationJavadocResolver()` | 注册 Sa-Token 权限注解解析器 |

**条件装配：**

```java
@ConditionalOnProperty(
    name = "springdoc.api-docs.enabled",
    havingValue = "true",
    matchIfMissing = true  // 默认开启，显式设为false才关闭
)
```

**`PlusPaths` 内部类：**

```java
static class PlusPaths extends Paths { }
```

用于标记路径是否已经被处理过，防止 `OpenApiCustomizer` 重复拼接 `context-path`。

---

### 3.2 `SpringDocProperties` — 配置属性

映射 `application.yml` 中 `springdoc` 前缀的配置：

```yaml
springdoc:
  info:
    title: '接口文档标题'
    description: '接口文档描述'
    version: '版本号'
    contact:
      name: '作者'
      email: '邮箱'
      url: '项目地址'
  externalDocs: ...   # 扩展文档地址
  tags: [...]         # 标签列表
  paths: {...}        # 路径配置
  components: ...     # 组件配置（含安全方案 securitySchemes）
```

**内部类 `InfoProperties`** 复制自 Swagger `Info` 类，目的是让 Spring Boot 自动生成配置提示（IDE 补全支持）。

---

### 3.3 `OpenApiHandler` — 自定义 OpenAPI 处理器

**继承关系：**

```
OpenAPIService (SpringDoc 核心服务)
    └── OpenApiHandler (本模块增强)
```

**核心重写方法：`buildTags()`**

该方法在 SpringDoc 构建 API 文档的 Tag 时被调用，本模块对其进行了增强：

**增强点 1：JavaDoc 注释作为 Tag 名称**

```java
// 默认行为：使用类名（如 SysUserController）作为 Tag
// 增强后：使用类的 JavaDoc 第一行注释作为 Tag 名称
// 例如：/** 用户管理 */ → Tag 名称为 "用户管理"
String description = javadocProvider.get().getClassJavadoc(handlerMethod.getBeanType());
tag.setName(list.get(0));  // 取 JavaDoc 第一行
```

**增强点 2：JavaDoc 注释作为接口摘要**

```java
// 方法的 JavaDoc 第一句话自动成为接口的 summary
String description = javadocProvider.get().getMethodJavadocDescription(handlerMethod.getMethod());
String summary = javadocProvider.get().getFirstSentence(description);
operation.setSummary(summary);
```

**增强点 3：权限注解解析注入描述**

```java
// 遍历所有注册的 JavadocResolver，将权限信息追加到接口描述中
for (JavadocResolver resolver : javadocResolvers) {
    String desc = resolver.resolve(handlerMethod, operation);
    description = description + desc;
}
operation.setDescription(description);
```

最终效果：Swagger UI 中每个接口自动展示权限要求（无需手写 `@Operation(description=...)`）。

---

## 4. 解析器体系（策略 + 模板方法）

### 4.1 `JavadocResolver` — 解析器接口

```java
public interface JavadocResolver extends Comparable<JavadocResolver>, Ordered {

    // 是否支持解析该 HandlerMethod
    boolean supports(HandlerMethod handlerMethod);

    // 执行解析，返回追加到接口描述的文本
    String resolve(HandlerMethod handlerMethod, Operation operation);

    // 优先级（默认最低）
    default int getOrder() { return Ordered.LOWEST_PRECEDENCE; }

    // 解析器名称（默认类名）
    default String getName() { return this.getClass().getSimpleName(); }
}
```

**设计要点：**
- 继承 `Ordered` + `Comparable`，支持多解析器按优先级排序
- `supports()` 判断是否适用，`resolve()` 执行解析

---

### 4.2 `AbstractMetadataJavadocResolver<M>` — 抽象元数据解析器

**模板方法模式：**

```java
public abstract class AbstractMetadataJavadocResolver<M> implements JavadocResolver {

    private final Supplier<M> metadataProvider;  // 元数据工厂

    // 模板方法：调用子类提供的 resolve(handlerMethod, operation, metadata)
    @Override
    public String resolve(HandlerMethod handlerMethod, Operation operation) {
        return resolve(handlerMethod, operation, metadataProvider.get());
    }

    // 子类实现：带元数据的解析
    public abstract String resolve(HandlerMethod handlerMethod, Operation operation, M metadata);

    // 注解检测工具方法（类级别/方法级别）
    public boolean hasClassAnnotation(HandlerMethod, Class/String);
    public boolean hasMethodAnnotation(HandlerMethod, Class/String);
    public boolean hasAnnotation(HandlerMethod, Class/String);  // 类 OR 方法

    // 注解值提取工具方法
    public Map<String, Object> getClassAnnotationValueMap(HandlerMethod, Class/String);
    public Map<String, Object> getMethodAnnotationValueMap(HandlerMethod, Class/String);
}
```

**设计要点：**
- 泛型 `<M>` 支持任意元数据类型
- 支持通过**类名字符串**检测注解（避免编译期硬依赖，Sa-Token 不在 doc 模块的 Maven 依赖中）
- 提供注解检测与值提取的便捷方法，子类只需关注业务逻辑

---

### 4.3 `SaTokenAnnotationMetadataJavadocResolver` — Sa-Token 权限解析器

**核心职责：** 扫描 Controller 方法/类上的 Sa-Token 注解，将权限要求转换为 Markdown 文本，追加到接口文档描述中。

**支持的 Sa-Token 注解：**

| 注解 | 说明 | 处理方式 |
|------|------|---------|
| `@SaCheckPermission` | 权限校验 | 提取 `value`/`mode`/`type`/`orRole` 属性 |
| `@SaCheckRole` | 角色校验 | 提取 `value`/`mode`/`type` 属性 |
| `@SaIgnore` | 忽略权限 | 标记为"忽略权限检查" |
| `@SaCheckLogin` | 登录校验 | （常量定义，暂未特殊处理） |

**解析流程：**

```
HandlerMethod
    │
    ├─ 有 @SaIgnore？
    │   └─ YES → 输出 "忽略权限检查"，结束
    │
    ├─ 有 @SaCheckPermission？（方法级 + 类级）
    │   └─ 提取 value/mode/type/orRole → 添加到 metadata.permissions
    │
    └─ 有 @SaCheckRole？（方法级 + 类级）
        └─ 提取 value/mode/type → 添加到 metadata.roles
```

**类加载策略：**

```java
// 通过类名字符串加载，避免编译期依赖 Sa-Token
private static final String BASE_CLASS_NAME = "cn.dev33.satoken.annotation";
private static final String SA_CHECK_ROLE_CLASS_NAME = BASE_CLASS_NAME + ".SaCheckRole";
// ...
static {
    SA_CHECK_ROLE_CLASS = ClassLoaderUtil.loadClass(SA_CHECK_ROLE_CLASS_NAME, false);
    // ...
}
```

**输出效果（Markdown 渲染到 Swagger UI）：**

```
场景1: 需要登录（无注解）
> 权限策略：需要登录

场景2: @SaIgnore
> 权限策略：忽略权限检查

场景3: @SaCheckPermission(value = "system:user:list", mode = "AND")
**权限校验：**
- `system:user:list`

场景4: @SaCheckPermission(value = {"system:user:add", "system:user:edit"}, mode = "OR")
**权限校验：**
- `system:user:add` | `system:user:edit`

场景5: @SaCheckPermission(value = "system:user:list", orRole = "admin")
**权限校验：**
- `system:user:list`
  - 或角色：`admin`
```

---

### 4.4 `SaTokenSecurityMetadata` — 权限元数据模型

**数据结构：**

```
SaTokenSecurityMetadata
├── permissions: List<AuthInfo>   ← @SaCheckPermission 解析结果
├── roles: List<AuthInfo>         ← @SaCheckRole 解析结果
├── ignore: boolean               ← @SaIgnore 标记
│
└── AuthInfo (内部类)
    ├── values: String[]     ← 权限值/角色值数组
    ├── mode: String         ← 校验模式（AND/OR）
    ├── type: String         ← 类型说明
    ├── orValues: String[]   ← 或权限/角色值（用于 orRole）
    ├── orType: String       ← 或值类型（role/permission）
    └── getModeSymbol()      ← AND→` & `，OR→` | `
```

**`toMarkdownString()` 方法：**

将权限信息渲染为 HTML+Markdown 混合格式，直接嵌入 Swagger UI 的接口描述区域。

---

## 5. 整体工作流程

```
Spring Boot 启动
    │
    ▼
SpringDocConfig 自动装配
    ├─ 创建 OpenAPI Bean（文档基本信息）
    ├─ 创建 OpenApiHandler Bean（替换默认 OpenAPIService）
    ├─ 创建 OpenApiCustomizer Bean（路径前缀处理）
    └─ 创建 SaTokenAnnotationMetadataJavadocResolver Bean
    │
    ▼
请求 /v3/api-docs（Swagger UI 加载文档）
    │
    ▼
OpenApiHandler.buildTags() 被调用（每个 Controller 方法）
    │
    ├─ 1. 构建 Tag（方法级 + 类级 @Tag 注解）
    │
    ├─ 2. JavaDoc 注释 → Tag 名称（类注释第一行）
    │
    ├─ 3. JavaDoc 注释 → 接口 Summary（方法注释第一句话）
    │
    └─ 4. 遍历 javadocResolvers
        │
        └─ SaTokenAnnotationMetadataJavadocResolver
            ├─ 检测 @SaIgnore / @SaCheckPermission / @SaCheckRole
            ├─ 提取注解属性 → SaTokenSecurityMetadata
            └─ toMarkdownString() → 追加到 operation.description
    │
    ▼
最终 Swagger UI 展示
    ├─ Tag: 来自类的 JavaDoc 注释（如"用户管理"）
    ├─ Summary: 来自方法的 JavaDoc 第一句话
    └─ Description: JavaDoc 描述 + 权限信息 Markdown
```

---

## 6. 设计模式与亮点

### 6.1 设计模式

| 模式 | 体现 |
|------|------|
| **策略模式** | `JavadocResolver` 接口 + 多个实现，`OpenApiHandler` 遍历调用 |
| **模板方法** | `AbstractMetadataJavadocResolver.resolve()` 定义骨架，子类实现具体解析 |
| **工厂方法** | `Supplier<M> metadataProvider` 延迟创建元数据对象 |
| **装饰器模式** | `OpenApiHandler` 继承 `OpenAPIService`，重写 `buildTags()` 增强功能 |

### 6.2 设计亮点

1. **零侵入权限展示**: 无需在 Controller 上额外写 `@Operation(description=...)` 描述权限，自动从 `@SaCheckPermission` 等注解提取并渲染到文档
2. **JavaDoc 即文档**: 配合 `therapi-runtime-javadoc`，编译期提取 Java 注释，运行时生成文档，代码注释即 API 文档
3. **软依赖 Sa-Token**: 通过 `ClassLoaderUtil.loadClass()` 按字符串加载注解类，doc 模块无需 Maven 依赖 Sa-Token，解耦彻底
4. **可扩展解析器**: `JavadocResolver` 接口化设计，未来可轻松扩展其他权限框架（如 Spring Security）的解析器
5. **优先级排序**: 解析器支持 `Ordered` 排序，多个解析器按顺序执行
6. **路径防重复**: `PlusPaths` 标记类防止 `context-path` 被重复拼接
7. **条件装配**: 通过 `@ConditionalOnProperty` 支持一键关闭文档功能
8. **配置提示友好**: `SpringDocProperties.InfoProperties` 复制 Swagger 类结构，让 IDE 自动生成 YML 配置提示

---

## 7. 模块依赖关系

```
ruoyi-common-doc
│
├── 依赖
│   ├── ruoyi-common-core         ← StringUtils、StreamUtils
│   ├── springdoc-openapi         ← OpenAPI 模型、OpenAPIService
│   └── therapi-runtime-javadoc   ← JavaDoc 运行时读取
│
├── 软依赖（运行时类加载，非 Maven 依赖）
│   └── cn.dev33.satoken.annotation.*  ← Sa-Token 权限注解
│
└── 被依赖
    └── ruoyi-admin               ← 主启动模块引入后自动生效
```

---

## 8. 配置示例

```yaml
springdoc:
  api-docs:
    enabled: true                    # 是否开启接口文档
  info:
    title: 'RuoYi-Vue-Plus_接口文档'
    description: '多租户管理系统'
    version: '5.6.2'
    contact:
      name: 'Lion Li'
      email: 'crazylionli@163.com'
  group-configs:                     # 分组配置
    - group: 1.演示模块
      packages-to-scan: org.dromara.demo
    - group: 3.系统模块
      packages-to-scan: org.dromara.system
    - group: 5.工作流模块
      packages-to-scan: org.dromara.workflow
```

---

## 9. 总结

`ruoyi-common-doc` 是一个**轻量但设计精巧**的文档增强模块，仅用 7 个 Java 文件实现了以下能力：

| 能力 | 实现方式 |
|------|---------|
| API 文档自动生成 | SpringDoc OpenAPI + therapi-runtime-javadoc |
| JavaDoc 注释 → Tag 名称 | 重写 `OpenAPIService.buildTags()` |
| JavaDoc 注释 → 接口摘要 | 自动提取方法注释第一句话 |
| 权限注解 → 文档描述 | `JavadocResolver` 解析器体系 |
| Sa-Token 权限可视化 | `SaTokenAnnotationMetadataJavadocResolver` + Markdown 渲染 |
| 上下文路径处理 | `OpenApiCustomizer` + `PlusPaths` 防重复 |
| 可扩展架构 | 策略模式 + 模板方法，新增解析器只需实现接口 |

该模块的核心价值在于：**让开发者只写 Java 注释和权限注解，即可自动生成包含权限说明的完整 API 文档**，实现了代码与文档的真正统一。
