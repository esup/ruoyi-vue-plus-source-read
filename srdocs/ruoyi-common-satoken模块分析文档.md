# ruoyi-common-satoken 模块分析文档

## 1. 模块概述

| 属性 | 说明 |
|------|------|
| 模块名 | ruoyi-common-satoken |
| 包路径 | `org.dromara.common.satoken` |
| 定位 | **权限认证模块**，基于 Sa-Token + JWT 实现登录认证、权限校验、会话管理 |
| Java文件数 | 5 个 |
| 依赖关系 | 依赖 `ruoyi-common-core`、`ruoyi-common-redis`、`sa-token-spring-boot3-starter`、`sa-token-jwt`、`caffeine` |

### 1.1 Maven 依赖

```
ruoyi-common-core              ← 核心基础（LoginUser、常量、枚举、异常）
ruoyi-common-redis             ← Redis 缓存（PlusSaTokenDao 持久层）
sa-token-spring-boot3-starter  ← Sa-Token 权限认证框架
sa-token-jwt                   ← Sa-Token 整合 JWT
caffeine                       ← 本地缓存（多级缓存优化）
spring-webmvc                  ← Web MVC 支持
```

### 1.2 自动装配

通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册：

```
SaTokenConfig  ← 唯一自动配置入口
```

### 1.3 内置配置（common-satoken.yml）

```yaml
# 内置配置 不允许修改 如需修改请在 nacos 上写相同配置覆盖
sa-token:
  dynamic-active-timeout: true   # 允许动态设置 token 有效期
  is-read-body: true             # 允许从请求参数读取 token
  is-read-header: true           # 允许从 header 读取 token
  is-read-cookie: false          # 关闭 cookie 鉴权（从根源杜绝 CSRF 漏洞）
  token-prefix: "Bearer"         # token 前缀
```

---

## 2. 包结构总览

```
org.dromara.common.satoken
├── config/                          # 配置层
│   └── SaTokenConfig.java           ← 自动配置主类（注册4个核心Bean）
│
├── core/                            # 核心实现层
│   ├── dao/
│   │   └── PlusSaTokenDao.java      ← Sa-Token持久层（Caffeine + Redis 多级缓存）
│   └── service/
│       └── SaPermissionImpl.java    ← 权限接口实现（StpInterface）
│
├── handler/                         # 异常处理层
│   └── SaTokenExceptionHandler.java ← Sa-Token 异常统一处理
│
└── utils/                           # 工具类层
    └── LoginHelper.java             ← 登录鉴权助手（登录/获取用户/权限判断）
```

---

## 3. 核心组件详解

### 3.1 `SaTokenConfig` — 自动配置主类

**职责**: 注册 Sa-Token 框架的 4 个核心 Bean。

**注册的 Bean：**

| Bean | 类型 | 说明 |
|------|------|------|
| `getStpLogicJwt()` | `StpLogicJwtForSimple` | Sa-Token 整合 JWT（简单模式） |
| `stpInterface()` | `SaPermissionImpl` | 权限接口实现（获取权限列表/角色列表） |
| `saTokenDao()` | `PlusSaTokenDao` | 自定义持久层（Caffeine + Redis 多级缓存） |
| `saTokenExceptionHandler()` | `SaTokenExceptionHandler` | 异常处理器 |

**配置加载方式：**

```java
@PropertySource(
    value = "classpath:common-satoken.yml",
    factory = YmlPropertySourceFactory.class  // 自定义工厂支持 YML 格式
)
```

**JWT 模式选择：**

框架使用 `StpLogicJwtForSimple`（简单模式），特点：
- Token 由 Sa-Token 生成并管理
- JWT 仅作为 Token 载体，不存储会话数据
- 会话数据仍存储在 Redis 中（通过 `PlusSaTokenDao`）
- 兼顾 JWT 的无状态特性与会话管理的灵活性

---

### 3.2 `PlusSaTokenDao` — 持久层（多级缓存）

**继承关系：**

```
SaTokenDao (Sa-Token 接口)
    └── SaTokenDaoBySessionFollowObject (简化Session处理)
        └── PlusSaTokenDao (本模块实现)
```

**核心设计：Caffeine + Redis 二级缓存**

```
请求
 │
 ▼
┌─────────────────────────────────┐
│  L1: Caffeine 本地缓存           │
│  - 过期时间: 5秒                 │
│  - 初始容量: 100                 │
│  - 最大条数: 1000                │
│  - 策略: expireAfterWrite        │
└────────────┬────────────────────┘
             │ 未命中
             ▼
┌─────────────────────────────────┐
│  L2: Redis 分布式缓存            │
│  - 通过 RedisUtils 操作          │
│  - 支持过期时间/永久存储           │
└─────────────────────────────────┘
```

**实现的方法：**

| 方法 | 说明 | 缓存策略 |
|------|------|---------|
| `get(key)` | 获取字符串值 | Caffeine → Redis |
| `set(key, value, timeout)` | 写入字符串值 | Redis写入 → Caffeine失效 |
| `update(key, value)` | 更新值（过期时间不变） | Redis更新 → Caffeine失效 |
| `delete(key)` | 删除值 | Redis删除 → Caffeine失效 |
| `getTimeout(key)` | 获取剩余存活时间 | 直接查 Redis（+1秒精度补偿） |
| `updateTimeout(key, timeout)` | 修改剩余存活时间 | 直接更新 Redis |
| `getObject(key)` | 获取对象 | Caffeine → Redis |
| `getObject(key, classType)` | 获取对象（指定类型） | Caffeine → Redis |
| `setObject(key, object, timeout)` | 写入对象 | Redis写入 → Caffeine失效 |
| `updateObject(key, object)` | 更新对象 | Redis更新 → Caffeine失效 |
| `deleteObject(key)` | 删除对象 | Redis删除 → Caffeine失效 |
| `getObjectTimeout(key)` | 获取对象剩余存活时间 | 直接查 Redis（+1秒精度补偿） |
| `updateObjectTimeout(key, timeout)` | 修改对象剩余存活时间 | 直接更新 Redis |
| `searchData(prefix, keyword, ...)` | 搜索数据 | Caffeine → Redis.keys() |

**精度补偿：**

```java
// Sa-Token 使用秒，Redis 使用毫秒，存在 1 秒精度差
// 通过 +1 手动补偿
return timeout < 0 ? timeout : timeout / 1000 + 1;
```

**缓存一致性策略：**

- **读操作**: 先查 Caffeine，未命中则查 Redis 并回填 Caffeine
- **写操作**: 写入 Redis 后，立即失效 Caffeine 对应 key
- **删除操作**: 删除 Redis 后，立即失效 Caffeine 对应 key

---

### 3.3 `SaPermissionImpl` — 权限接口实现

**实现接口：** `StpInterface`（Sa-Token 权限查询接口）

**核心方法：**

| 方法 | 说明 |
|------|------|
| `getPermissionList(loginId, loginType)` | 获取用户的菜单权限列表 |
| `getRoleList(loginId, loginType)` | 获取用户的角色权限列表 |

**查询逻辑流程：**

```
getPermissionList / getRoleList
    │
    ├─ 1. 尝试从当前会话获取 LoginUser
    │
    ├─ 2. LoginUser 存在且 loginId 匹配？
    │   │
    │   ├─ YES → 直接返回 loginUser.getMenuPermission() / getRolePermission()
    │   │
    │   └─ NO → 通过 PermissionService 查询（跨会话场景）
    │       │
    │       ├─ 解析 loginId 格式: "userType:userId"
    │       ├─ 提取 userId
    │       └─ 调用 permissionService.getMenuPermission(userId) / getRolePermission(userId)
    │
    └─ 3. PermissionService 不存在？
        └─ 抛出 ServiceException("PermissionService 实现类不存在")
```

**设计要点：**

1. **优先使用会话数据**: 当前登录用户直接从 Session 读取，避免重复查询数据库
2. **支持跨会话查询**: 当 `loginId` 与当前登录用户不匹配时（如管理员查看其他用户权限），通过 `PermissionService` 查询
3. **SPI 扩展**: `PermissionService` 是定义在 `ruoyi-common-core` 中的接口，由业务模块实现，支持灵活替换

---

### 3.4 `SaTokenExceptionHandler` — 异常处理器

**职责**: 统一处理 Sa-Token 抛出的认证/授权异常，返回标准 `R<T>` 响应。

**处理的异常类型：**

| 异常 | HTTP状态码 | 响应消息 | 说明 |
|------|-----------|---------|------|
| `NotPermissionException` | 403 | "没有访问权限，请联系管理员授权" | 权限码校验失败 |
| `NotRoleException` | 403 | "没有访问权限，请联系管理员授权" | 角色权限校验失败 |
| `NotLoginException` | 401 | "认证失败，无法访问系统资源" | 未登录/Token失效 |

**日志记录：**

```java
log.error("请求地址'{}',权限码校验失败'{}'", requestURI, e.getMessage());
```

每个异常处理都会记录请求 URI 和异常信息，便于排查问题。

---

### 3.5 `LoginHelper` — 登录鉴权助手

**职责**: 封装 Sa-Token 的登录/登出/获取用户信息等操作，提供统一的业务层调用接口。

**Session 存储结构：**

```
Token Session
├── loginUser (LoginUser 对象)
│
└── Extra 扩展信息
    ├── tenantId      ← 租户ID
    ├── userId        ← 用户ID
    ├── userName      ← 用户名
    ├── deptId        ← 部门ID
    ├── deptName      ← 部门名称
    └── deptCategory  ← 部门类别编码
```

**核心方法：**

| 方法 | 说明 |
|------|------|
| `login(loginUser, model)` | 执行登录，将用户信息存入 Session |
| `getLoginUser()` | 获取当前登录用户（从 Token Session） |
| `getLoginUser(token)` | 根据指定 Token 获取用户 |
| `getUserId()` / `getUserIdStr()` | 获取当前用户ID |
| `getUsername()` | 获取当前用户名 |
| `getTenantId()` | 获取当前租户ID |
| `getDeptId()` / `getDeptName()` / `getDeptCategory()` | 获取部门信息 |
| `getUserType()` | 获取用户类型（sys_user/sys_social） |
| `isSuperAdmin()` / `isSuperAdmin(userId)` | 判断是否为超级管理员 |
| `isTenantAdmin()` / `isTenantAdmin(rolePermission)` | 判断是否为租户管理员 |
| `isLogin()` | 检查当前用户是否已登录 |

**登录流程：**

```java
public static void login(LoginUser loginUser, SaLoginParameter model) {
    // 1. 调用 Sa-Token 登录
    StpUtil.login(loginUser.getLoginId(),
        model.setExtra(TENANT_KEY, loginUser.getTenantId())
             .setExtra(USER_KEY, loginUser.getUserId())
             // ... 其他扩展信息
    );
    // 2. 将完整 LoginUser 对象存入 Token Session
    StpUtil.getTokenSession().set(LOGIN_USER_KEY, loginUser);
}
```

**设计要点：**

1. **双重存储**: 扩展信息存入 JWT Extra（轻量查询），完整对象存入 Session（详细使用）
2. **泛型支持**: `getLoginUser()` 返回泛型 `<T extends LoginUser>`，支持业务扩展
3. **空安全**: `getExtra()` 方法捕获异常返回 null，避免 Token 失效时抛异常
4. **多用户体系**: 支持 `userType`（用户类型）+ `deviceType`（设备类型）多对多权限控制

---

## 4. 整体工作流程

### 4.1 登录认证流程

```
用户请求登录
    │
    ▼
AuthController.login()
    │
    ▼
SysLoginService.login()
    │
    ├─ 1. 校验用户名/密码/验证码
    ├─ 2. 构建 LoginUser 对象（填充权限/角色/部门信息）
    └─ 3. 调用 LoginHelper.login(loginUser, model)
        │
        ├─ StpUtil.login(loginId, extras)  ← Sa-Token 生成 Token
        └─ StpUtil.getTokenSession().set("loginUser", loginUser)  ← 存储用户信息
    │
    ▼
返回 Token 给前端
    │
    ▼
前端后续请求携带: Authorization: Bearer {token}
```

### 4.2 请求鉴权流程

```
HTTP 请求（携带 Token）
    │
    ▼
Sa-Token 拦截器
    │
    ├─ 从 Header/参数 读取 Token（is-read-body/is-read-header）
    ├─ 校验 Token 有效性
    └─ 解析 JWT 获取 loginId
    │
    ▼
权限注解校验（@SaCheckPermission / @SaCheckRole）
    │
    ▼
SaPermissionImpl.getPermissionList() / getRoleList()
    │
    ├─ 优先从 Session 获取 LoginUser
    └─ 不匹配则通过 PermissionService 查询
    │
    ▼
比对权限码/角色标识
    │
    ├─ 匹配 → 放行
    └─ 不匹配 → 抛异常 → SaTokenExceptionHandler 统一处理
```

### 4.3 多级缓存读取流程

```
获取 Token Session 数据
    │
    ▼
┌─────────────────────────┐
│ L1: Caffeine 本地缓存    │
│ - 命中 → 直接返回        │
│ - 未命中 → 继续          │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ L2: Redis 分布式缓存     │
│ - 查询并回填 L1          │
│ - 返回结果               │
└─────────────────────────┘
```

---

## 5. 设计模式与亮点

### 5.1 设计模式

| 模式 | 体现 |
|------|------|
| **策略模式** | `StpLogic` 可替换不同 JWT 实现（Simple/Stateless/Mixin） |
| **模板方法** | `SaTokenDaoBySessionFollowObject` 定义骨架，`PlusSaTokenDao` 实现具体存储 |
| **代理模式** | `SaPermissionImpl` 代理 `PermissionService`，支持会话内直接读取 |
| **装饰器模式** | `PlusSaTokenDao` 在 Redis 基础上装饰 Caffeine 缓存层 |

### 5.2 设计亮点

1. **多级缓存架构**: Caffeine（5秒）+ Redis，优化高并发场景下 Session 查询性能
2. **缓存一致性保证**: 写操作后立即失效本地缓存，避免脏数据
3. **精度补偿机制**: `getTimeout()` 返回值 +1 秒，解决 Sa-Token（秒）与 Redis（毫秒）的精度差
4. **双重存储策略**: 轻量信息存 JWT Extra（避免频繁查 Session），完整对象存 Session
5. **跨会话支持**: `SaPermissionImpl` 支持查询非当前登录用户的权限（通过 `PermissionService`）
6. **SPI 扩展设计**: `PermissionService` 接口化，业务模块可自由实现权限查询逻辑
7. **CSRF 防护**: 默认关闭 Cookie 鉴权（`is-read-cookie: false`），从根源杜绝 CSRF 漏洞
8. **异常统一处理**: 三种 Sa-Token 异常统一返回 `R<T>` 格式，保持 API 响应一致性
9. **内置配置保护**: `common-satoken.yml` 标注"不允许修改"，通过 Nacos 覆盖实现灵活配置
10. **多用户体系支持**: `userType` + `deviceType` 多对多组合，支持 PC/APP/小程序等多端独立权限控制

---

## 6. 模块依赖关系

```
ruoyi-common-satoken
│
├── 依赖
│   ├── ruoyi-common-core      ← LoginUser、常量、枚举、异常、PermissionService接口
│   ├── ruoyi-common-redis     ← RedisUtils（缓存操作）
│   ├── sa-token-spring-boot3-starter  ← Sa-Token 核心框架
│   ├── sa-token-jwt           ← JWT 整合
│   └── caffeine               ← 本地缓存
│
├── 被依赖
│   ├── ruoyi-common-security  ← 安全模块（请求过滤+鉴权）
│   └── ruoyi-admin            ← 主启动模块
│
└── 关联模块
    └── ruoyi-common-core      ← PermissionService 接口定义在此
```

---

## 7. 与其他模块的协作

```
┌─────────────────────────────────────────────────────────┐
│                      ruoyi-admin                        │
│  ┌─────────────────────────────────────────────────┐    │
│  │  AuthController → SysLoginService               │    │
│  │       ↓                                         │    │
│  │  LoginHelper.login()                            │    │
│  │       ↓                                         │    │
│  │  ┌─────────────────────────────────────────┐    │    │
│  │  │      ruoyi-common-satoken               │    │    │
│  │  │  ┌─────────────────────────────────┐    │    │    │
│  │  │  │ StpUtil.login() → 生成Token     │    │    │    │
│  │  │  │ PlusSaTokenDao → Redis存储      │    │    │    │
│  │  │  │ SaPermissionImpl → 权限查询     │    │    │    │
│  │  │  └─────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 8. 配置示例

### 8.1 application.yml 中的 Sa-Token 配置

```yaml
sa-token:
  token-name: Authorization        # token 名称（同时也是 cookie 名称）
  is-concurrent: true              # 是否允许同一账号并发登录
  is-share: false                  # 多人登录是否共用 token
  jwt-secret-key: abcdefghijklmnopqrstuvwxyz  # JWT 秘钥
```

### 8.2 权限注解使用示例

```java
// 权限码校验
@SaCheckPermission("system:user:list")
@GetMapping("/list")
public R<List<SysUser>> list() { ... }

// 角色校验
@SaCheckRole("admin")
@DeleteMapping("/{userId}")
public R<Void> remove(@PathVariable Long userId) { ... }

// 登录校验
@SaCheckLogin
@PostMapping("/update")
public R<Void> update(@RequestBody SysUser user) { ... }

// 忽略校验
@SaIgnore
@GetMapping("/public/info")
public R<Void> publicInfo() { ... }
```

---

## 9. 总结

`ruoyi-common-satoken` 是系统的**权限认证核心模块**，仅用 5 个 Java 文件实现了以下能力：

| 能力 | 实现 |
|------|------|
| 登录认证 | `LoginHelper.login()` + Sa-Token + JWT |
| 会话管理 | `PlusSaTokenDao`（Caffeine + Redis 二级缓存） |
| 权限查询 | `SaPermissionImpl`（会话优先 + SPI 扩展） |
| 异常处理 | `SaTokenExceptionHandler`（401/403 统一响应） |
| 业务工具 | `LoginHelper`（登录/获取用户/权限判断一站式方法） |

该模块的核心价值在于：

1. **高性能**: Caffeine + Redis 多级缓存，5秒本地缓存大幅降低 Redis 访问压力
2. **高扩展**: `PermissionService` SPI 接口，业务模块可自由实现权限逻辑
3. **高安全**: JWT 简单模式 + 关闭 Cookie 鉴权，兼顾无状态与安全
4. **高可用**: 完善的异常处理与空安全设计，保证系统稳定运行
