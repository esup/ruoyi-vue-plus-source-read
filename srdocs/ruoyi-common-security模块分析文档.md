# ruoyi-common-security 模块分析文档

## 1. 模块概述

| 属性 | 说明 |
|------|------|
| 模块名 | ruoyi-common-security |
| 包路径 | `org.dromara.common.security` |
| 定位 | **请求安全过滤模块**，基于 Sa-Token 实现全局请求拦截、登录校验、客户端校验、Actuator 鉴权 |
| Java文件数 | 3 个 |
| 依赖关系 | 仅依赖 `ruoyi-common-satoken`（间接依赖 core、redis、Sa-Token） |

### 1.1 Maven 依赖

```
ruoyi-common-satoken  ← 唯一直接依赖（间接引入 core、redis、Sa-Token、JWT、Caffeine）
```

### 1.2 自动装配

通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册：

```
AllUrlHandler    ← URL 收集器（先于 SecurityConfig 初始化）
SecurityConfig   ← 安全配置主类
```

---

## 2. 包结构总览

```
org.dromara.common.security
├── config/                              # 配置层
│   ├── SecurityConfig.java              ← 安全配置主类（拦截器注册 + Actuator 鉴权）
│   └── properties/
│       └── SecurityProperties.java      ← 安全配置属性（排除路径）
│
└── handler/                             # 处理器层
    └── AllUrlHandler.java               ← 全量 URL 收集器
```

---

## 3. 核心组件详解

### 3.1 `SecurityProperties` — 安全配置属性

映射 `application.yml` 中 `security` 前缀的配置：

```yaml
security:
  excludes:
    - /*.html
    - /**/*.html
    - /**/*.css
    - /**/*.js
    - /favicon.ico
    - /error
    - /*/api-docs
    - /*/api-docs/**
    - /warm-flow-ui/config
```

**职责**: 定义不需要登录校验的白名单路径，支持 Ant 风格通配符。

---

### 3.2 `AllUrlHandler` — 全量 URL 收集器

**实现接口**: `InitializingBean`（Spring Bean 初始化回调）

**职责**: 在应用启动时收集所有注册的 Controller 路由地址，供拦截器使用。

**核心逻辑：**

```java
@Override
public void afterPropertiesSet() {
    Set<String> set = new HashSet<>();
    // 获取 Spring MVC 的路由映射处理器
    RequestMappingHandlerMapping mapping = SpringUtils.getBean(
        "requestMappingHandlerMapping", RequestMappingHandlerMapping.class);
    Map<RequestMappingInfo, HandlerMethod> map = mapping.getHandlerMethods();

    map.keySet().forEach(info -> {
        // 遍历所有路由模式
        Objects.requireNonNull(info.getPathPatternsCondition().getPatterns())
            .forEach(url -> {
                // 将路径变量 {id} 替换为通配符 *
                String pattern = ReUtil.replaceAll(
                    url.getPatternString(), PATTERN, "*");
                set.add(pattern);
            });
    });
    urls.addAll(set);
}
```

**路径变量转换示例：**

| 原始路径 | 转换后 |
|---------|--------|
| `/system/user/{userId}` | `/system/user/*` |
| `/system/role/{roleId}/menu` | `/system/role/*/menu` |
| `/system/user/list` | `/system/user/list` |

**设计目的：**

Sa-Token 的 `SaRouter.match()` 需要知道所有有效路由才能进行拦截。如果直接 `match("/**")` 会匹配到静态资源等无效路径，`AllUrlHandler` 精确收集所有 Controller 路由，让拦截器只校验有效的业务接口。

---

### 3.3 `SecurityConfig` — 安全配置主类

**实现接口**: `WebMvcConfigurer`（Spring MVC 配置扩展）

**职责**: 注册 Sa-Token 拦截器 + Actuator 健康检查鉴权。

#### 3.3.1 拦截器注册 — `addInterceptors()`

```java
registry.addInterceptor(new SaInterceptor(handler -> {
    AllUrlHandler allUrlHandler = SpringUtils.getBean(AllUrlHandler.class);

    SaRouter
        .match(allUrlHandler.getUrls())    // 只匹配有效路由
        .check(() -> {
            // 1. 登录校验
            StpUtil.checkLogin();

            // 2. 客户端ID校验
            String headerCid = request.getHeader("clientid");
            String paramCid = request.getParameter("clientid");
            String clientId = StpUtil.getExtra("clientid").toString();
            if (!StringUtils.equalsAny(clientId, headerCid, paramCid)) {
                throw NotLoginException.newInstance(..., "客户端ID与Token不匹配");
            }
        });
}))
.addPathPatterns("/**")                              // 拦截所有路径
.excludePathPatterns(securityProperties.getExcludes()) // 排除白名单
.excludePathPatterns(ssePath);                         // 排除 SSE 推送路径
```

**拦截逻辑流程：**

```
HTTP 请求
    │
    ▼
路径匹配
    │
    ├─ 在 security.excludes 白名单中？ → 放行
    ├─ 是 SSE 路径？ → 放行
    ├─ 不在 AllUrlHandler 收集的路由中？ → 放行（静态资源等）
    │
    ▼
Sa-Token 登录校验
    │
    ├─ StpUtil.checkLogin() → Token 无效/过期 → 抛 NotLoginException
    │
    ▼
客户端ID校验
    │
    ├─ 从 Header 或 Param 获取 clientid
    ├─ 从 Token Extra 获取 clientid
    └─ 不匹配 → 抛 NotLoginException("客户端ID与Token不匹配")
    │
    ▼
放行 → 进入 Controller
```

**双重校验的意义：**

1. **登录校验**: 确保请求携带有效 Token
2. **客户端ID校验**: 防止 Token 被盗用（不同客户端的 Token 不能混用）

#### 3.3.2 Actuator 鉴权 — `getSaServletFilter()`

```java
@Bean
public SaServletFilter getSaServletFilter() {
    String username = SpringUtils.getProperty("spring.boot.admin.client.username");
    String password = SpringUtils.getProperty("spring.boot.admin.client.password");
    return new SaServletFilter()
        .addInclude("/actuator", "/actuator/**")
        .setAuth(obj -> {
            SaHttpBasicUtil.check(username + ":" + password);
        })
        .setError(e -> {
            return SaResult.error(e.getMessage()).setCode(HttpStatus.UNAUTHORIZED);
        });
}
```

**职责**: 对 Spring Boot Actuator 健康检查接口做 HTTP Basic 鉴权，防止未授权访问。

**鉴权方式**: HTTP Basic Authentication（`Authorization: Basic base64(username:password)`）

---

## 4. 整体工作流程

### 4.1 应用启动阶段

```
Spring Boot 启动
    │
    ▼
AllUrlHandler.afterPropertiesSet()
    │
    ├─ 获取 RequestMappingHandlerMapping
    ├─ 遍历所有 @RequestMapping 方法
    ├─ 提取路由路径
    ├─ 路径变量 {xxx} → 通配符 *
    └─ 存入 urls 列表
    │
    ▼
SecurityConfig 初始化
    │
    ├─ 注册 SaInterceptor（拦截所有路径）
    ├─ 排除 security.excludes 白名单
    ├─ 排除 SSE 路径
    └─ 注册 SaServletFilter（Actuator 鉴权）
```

### 4.2 请求处理阶段

```
HTTP 请求
    │
    ▼
┌─────────────────────────────────────────────────┐
│  SaServletFilter (Servlet 过滤器层)              │
│  /actuator/** → HTTP Basic 鉴权                 │
│  其他路径 → 放行                                 │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  SaInterceptor (拦截器层)                        │
│                                                 │
│  1. 路径是否在 AllUrlHandler 收集的路由中？       │
│     └─ 否 → 放行（静态资源等）                    │
│                                                 │
│  2. 路径是否在 security.excludes 白名单中？       │
│     └─ 是 → 放行                                │
│                                                 │
│  3. StpUtil.checkLogin() → 登录校验             │
│                                                 │
│  4. clientid 校验 → 客户端一致性校验              │
│                                                 │
│  5. 全部通过 → 放行                              │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
              Controller 处理
```

---

## 5. 设计亮点

### 5.1 精确路由拦截

传统做法是 `match("/**")` 拦截所有路径再排除，但这样会匹配到大量无效路径（静态资源、favicon 等）。

本模块通过 `AllUrlHandler` 预先收集所有有效路由，只拦截真正的业务接口，**减少不必要的校验开销**。

### 5.2 路径变量自动转换

```java
// 正则: \{(.*?)\}
// /system/user/{userId} → /system/user/*
ReUtil.replaceAll(url.getPatternString(), PATTERN, "*")
```

将 Spring MVC 的路径变量 `{xxx}` 转换为 Sa-Token 可识别的通配符 `*`，确保路由匹配正确。

### 5.3 客户端一致性校验

```java
// Token 中存储的 clientid
String clientId = StpUtil.getExtra("clientid").toString();
// 请求中携带的 clientid
String headerCid = request.getHeader("clientid");
String paramCid = request.getParameter("clientid");
// 必须一致
if (!StringUtils.equalsAny(clientId, headerCid, paramCid)) {
    throw NotLoginException...
}
```

防止 Token 被盗用：即使 Token 有效，如果客户端ID不匹配也会拒绝请求。

### 5.4 Actuator 独立鉴权

Actuator 健康检查接口使用独立的 HTTP Basic 鉴权（与业务 Token 体系分离），确保监控系统访问安全。

### 5.5 分层安全设计

```
┌────────────────────────────────────────┐
│  SaServletFilter (Servlet 过滤器层)     │
│  - Actuator 接口 → HTTP Basic 鉴权     │
└──────────────────┬─────────────────────┘
                   │
┌──────────────────▼─────────────────────┐
│  SaInterceptor (Spring MVC 拦截器层)    │
│  - 业务接口 → Token 登录校验            │
│  - 业务接口 → 客户端ID一致性校验         │
└──────────────────┬─────────────────────┘
                   │
┌──────────────────▼─────────────────────┐
│  @SaCheckPermission / @SaCheckRole     │
│  (Controller 方法级注解校验)             │
│  - 权限码校验                           │
│  - 角色校验                             │
└────────────────────────────────────────┘
```

三层安全体系各司其职：
1. **过滤器层**: Actuator 独立鉴权
2. **拦截器层**: 全局登录校验 + 客户端校验
3. **注解层**: 细粒度权限/角色校验（由 Sa-Token 注解支持）

---

## 6. 模块依赖关系

```
ruoyi-common-security
│
├── 依赖
│   └── ruoyi-common-satoken
│       ├── ruoyi-common-core      ← ServletUtils、StringUtils、SpringUtils、常量
│       ├── ruoyi-common-redis     ← Redis 缓存
│       ├── sa-token-spring-boot3-starter  ← Sa-Token 框架
│       └── sa-token-jwt           ← JWT 整合
│
├── 被依赖
│   ├── ruoyi-common-web           ← Web 基础配置
│   └── ruoyi-admin                ← 主启动模块
│
└── 配置来源
    ├── security.excludes          ← application.yml（白名单路径）
    ├── sse.path                   ← application.yml（SSE 路径）
    └── spring.boot.admin.client.* ← application.yml（Actuator 鉴权账号）
```

---

## 7. 配置示例

### 7.1 白名单路径配置

```yaml
security:
  excludes:
    - /*.html
    - /**/*.html
    - /**/*.css
    - /**/*.js
    - /favicon.ico
    - /error
    - /*/api-docs
    - /*/api-docs/**
    - /warm-flow-ui/config
```

### 7.2 Actuator 鉴权配置

```yaml
# application-dev.yml
spring:
  boot:
    admin:
      client:
        username: ruoyi
        password: 123456
```

### 7.3 SSE 路径配置

```yaml
sse:
  enabled: true
  path: /resource/sse
```

---

## 8. 总结

`ruoyi-common-security` 是系统的**请求安全网关**，仅用 3 个 Java 文件实现了以下能力：

| 能力 | 实现 |
|------|------|
| 全量路由收集 | `AllUrlHandler` 启动时收集所有 Controller 路由，路径变量自动转换 |
| 全局登录拦截 | `SaInterceptor` + `StpUtil.checkLogin()` |
| 客户端一致性校验 | Token Extra 中的 clientid 与请求中的 clientid 比对 |
| 白名单放行 | `SecurityProperties.excludes` 配置化排除路径 |
| Actuator 鉴权 | `SaServletFilter` + HTTP Basic Authentication |

该模块的核心价值在于：

1. **精确拦截**: `AllUrlHandler` 只拦截有效业务路由，避免静态资源等无效路径的校验开销
2. **双重校验**: 登录校验 + 客户端ID校验，防止 Token 盗用
3. **分层安全**: Filter（Actuator）→ Interceptor（登录）→ Annotation（权限），三层各司其职
4. **配置灵活**: 白名单路径通过 YML 配置，支持 Ant 风格通配符
