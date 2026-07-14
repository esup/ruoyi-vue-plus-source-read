# ruoyi-common-core 模块分析文档

## 1. 模块概述

| 属性 | 说明 |
|------|------|
| 模块名 | ruoyi-common-core |
| 包路径 | `org.dromara.common.core` |
| 定位 | **整个项目的基石模块**，提供所有其他模块共用的基础能力 |
| Java文件数 | 95 个 |
| 依赖关系 | 被几乎所有其他模块依赖，自身仅依赖 Spring 基础、Hutool、MapStruct-Plus、ip2region |

### 1.1 Maven 依赖

```
spring-context-support    ← Spring 核心工具
spring-web                ← Web 基础（RequestContextHolder 等）
spring-boot-starter-validation  ← Bean Validation（JSR-380）
spring-boot-starter-aop         ← AOP 支持
commons-lang3             ← Apache 字符串工具
jakarta.servlet-api       ← Servlet API
hutool-core / hutool-http / hutool-extra  ← Hutool 工具套件
lombok                    ← 代码简化
mapstruct-plus            ← 对象映射框架
ip2region                 ← 离线 IP 地址定位库
```

### 1.2 自动装配

通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册以下自动配置类：

```
ApplicationConfig     ← 应用基础配置
ThreadPoolConfig      ← 线程池配置
ValidatorConfig       ← 校验框架配置
SpringUtils           ← Spring 上下文工具（注册为 Bean）
```

---

## 2. 包结构总览

```
org.dromara.common.core
├── config/              # 配置类（3个）
│   ├── ApplicationConfig
│   ├── ThreadPoolConfig
│   └── ValidatorConfig
│
├── constant/            # 常量定义（8个）
│   ├── CacheConstants
│   ├── CacheNames
│   ├── Constants
│   ├── GlobalConstants
│   ├── HttpStatus
│   ├── RegexConstants
│   ├── SystemConstants
│   └── TenantConstants
│
├── domain/              # 领域模型
│   ├── R.java           # 统一响应体
│   ├── dto/             # 数据传输对象（14个）
│   ├── event/           # 事件模型（3个）
│   └── model/           # 登录模型（9个）
│
├── enums/               # 枚举定义（6个）
│   ├── BusinessStatusEnum
│   ├── DeviceType
│   ├── FormatsType
│   ├── LoginType
│   ├── UserStatus
│   └── UserType
│
├── exception/           # 异常体系（9个）
│   ├── ServiceException
│   ├── SseException
│   ├── base/BaseException
│   ├── file/（3个）
│   └── user/（3个）
│
├── utils/               # 工具类（20个）
│   ├── DateUtils
│   ├── DesensitizedUtils
│   ├── MapstructUtils
│   ├── MessageUtils
│   ├── NetUtils
│   ├── ObjectUtils
│   ├── ServletUtils
│   ├── SpringUtils
│   ├── StreamUtils
│   ├── StringUtils
│   ├── TreeBuildUtils
│   ├── ValidatorUtils
│   ├── file/（2个）
│   ├── ip/（2个）
│   ├── reflect/（1个）
│   ├── regex/（2个）
│   └── sql/（1个）
│
├── validate/            # 自定义校验（7个）
│   ├── AddGroup / EditGroup / QueryGroup
│   ├── dicts/（DictPattern + Validator）
│   └── enumd/（EnumPattern + Validator）
│
└── xss/                 # XSS防护（2个）
    ├── Xss              # 注解
    └── XssValidator     # 校验器
```

---

## 3. 核心组件详解

### 3.1 统一响应体 — `R<T>`

全局统一的 API 响应封装，所有 Controller 返回值的标准格式：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": { ... }
}
```

**静态工厂方法：**

| 方法 | 说明 |
|------|------|
| `R.ok()` / `R.ok(data)` / `R.ok(msg)` | 成功响应（code=200） |
| `R.fail()` / `R.fail(msg)` / `R.fail(code, msg)` | 失败响应（code=500） |
| `R.warn(msg)` / `R.warn(msg, data)` | 警告响应（code=WARN） |
| `R.isSuccess(ret)` / `R.isError(ret)` | 判断响应状态 |

设计特点：
- 泛型 `<T>` 支持任意数据类型
- 实现 `Serializable`，支持序列化传输
- 使用 Lombok `@Data` 简化代码

---

### 3.2 常量体系（8个常量类）

```
constant/
├── CacheConstants     ← 缓存相关常量（缓存Key前缀等）
├── CacheNames         ← 缓存组名称常量（定义缓存Key格式与过期策略）
├── Constants          ← 通用常量（分隔符、编码、通用标识）
├── GlobalConstants    ← 全局常量（全局Redis Key前缀）
├── HttpStatus         ← HTTP状态码常量（200/401/403/500/WARN等）
├── RegexConstants     ← 正则表达式常量（手机号/邮箱/身份证/URL等）
├── SystemConstants    ← 系统常量（正常/异常状态、菜单类型、超管ID等）
└── TenantConstants    ← 租户常量（租户相关标识）
```

**`CacheNames` 缓存键命名规范：**

```
格式: cacheNames#ttl#maxIdleTime#maxSize#local

示例:
  demo:cache#60s#10m#20     ← 60秒过期，10分钟空闲，最大20条
  sys_config                ← 不过期
  sys_tenant#30d            ← 30天过期（全局Key前缀）
  online_tokens             ← 在线用户Token缓存
```

缓存组支持：过期时间（ttl）、最大空闲时间（maxIdleTime，LRU清理）、组最大长度（maxSize）、本地缓存开关（local）。

---

### 3.3 异常体系

```
RuntimeException
└── BaseException              ← 基础异常（支持国际化消息）
    ├── ServiceException       ← 业务异常（支持占位符 {}）
    ├── SseException           ← SSE推送异常
    ├── FileException          ← 文件异常
    │   ├── FileNameLengthLimitExceededException
    │   └── FileSizeLimitExceededException
    └── UserException          ← 用户异常
        ├── CaptchaException       ← 验证码错误
        └── CaptchaExpireException ← 验证码过期
```

**`ServiceException` 使用方式：**

```java
// 直接传消息
throw new ServiceException("用户不存在");

// 支持占位符格式化（Hutool StrFormatter）
throw new ServiceException("用户{}不存在", username);

// 指定错误码
throw new ServiceException("操作失败", 500);
```

---

### 3.4 登录模型

**`LoginUser` — 登录用户身份模型：**

```
LoginUser
├── tenantId        ← 租户ID（多租户隔离）
├── userId          ← 用户ID
├── deptId          ← 部门ID
├── deptCategory    ← 部门类别编码
├── deptName        ← 部门名称
├── token           ← 用户唯一标识（Sa-Token）
├── userType        ← 用户类型（sys_user/sys_social）
├── loginTime       ← 登录时间戳
├── expireTime      ← Token过期时间
├── ipaddr          ← 登录IP
├── loginLocation   ← 登录地点（ip2region解析）
├── browser / os    ← 浏览器/操作系统信息
├── menuPermission  ← 菜单权限集合（Set<String>）
├── rolePermission  ← 角色权限集合（Set<String>）
├── username / nickname  ← 用户名/昵称
├── roles           ← 角色对象列表（List<RoleDTO>）
├── posts           ← 岗位对象列表（List<PostDTO>）
├── roleId          ← 数据权限当前角色ID
├── clientKey       ← 客户端标识
├── deviceType      ← 设备类型
└── getLoginId()    ← 返回 "userType:userId" 格式登录ID
```

**登录请求体（策略模式）：**

```
LoginBody (抽象基类，标记接口)
├── PasswordLoginBody   ← 账号密码登录
├── SmsLoginBody        ← 短信验证码登录
├── EmailLoginBody      ← 邮箱验证码登录
├── SocialLoginBody     ← 第三方社交登录
├── XcxLoginBody        ← 小程序登录
└── RegisterBody        ← 用户注册
```

---

### 3.5 DTO 数据传输对象

用于跨模块传递数据，避免模块间直接依赖实体类：

| DTO | 用途 |
|-----|------|
| `UserDTO` | 用户信息传递 |
| `RoleDTO` | 角色信息传递 |
| `DeptDTO` | 部门信息传递 |
| `PostDTO` | 岗位信息传递 |
| `DictDataDTO` / `DictTypeDTO` | 字典数据传递 |
| `OssDTO` | OSS文件信息传递 |
| `UserOnlineDTO` | 在线用户信息 |
| `StartProcessDTO` | 工作流发起参数 |
| `StartProcessReturnDTO` | 工作流发起返回 |
| `CompleteTaskDTO` | 工作流完成任务 |
| `TaskAssigneeDTO` | 工作流任务指派人 |
| `FlowCopyDTO` | 工作流抄送 |
| `FlowInstanceBizExtDTO` | 工作流实例业务扩展 |

---

### 3.6 事件模型

基于 Spring Event 的发布-订阅机制，用于模块间解耦通信：

| 事件类 | 说明 |
|--------|------|
| `ProcessEvent` | 流程总体事件（流程定义编码、实例ID、节点信息、状态等） |
| `ProcessTaskEvent` | 流程任务事件（任务级别，办理人、操作类型等） |
| `ProcessDeleteEvent` | 流程删除事件（删除流程实例时触发） |

---

### 3.7 枚举定义

| 枚举 | 说明 |
|------|------|
| `BusinessStatusEnum` | 业务状态（0正常 / 1失败），包含大量状态判断静态方法 |
| `DeviceType` | 设备类型（PC / Android / iOS / 小程序等） |
| `FormatsType` | 格式化工具类型（支持多种数据格式的转换） |
| `LoginType` | 登录类型（密码/短信/邮箱/社交/小程序） |
| `UserStatus` | 用户状态（正常/停用） |
| `UserType` | 用户类型（系统用户/社交用户） |

---

## 4. 工具类详解（20个）

### 4.1 核心工具类

#### `StringUtils` — 字符串工具（385行）

继承 `org.apache.commons.lang3.StringUtils`，底层委托 Hutool `StrUtil`：

| 方法 | 说明 |
|------|------|
| `isEmpty()` / `isNotEmpty()` | 空串判断 |
| `format(template, params)` | `{}` 占位符格式化 |
| `str2Set()` / `str2List()` | 字符串转集合 |
| `splitList()` / `splitTo()` | 切分字符串，支持自定义转换 |
| `toUnderScoreCase()` | 驼峰转下划线 |
| `toCamelCase()` / `convertToCamelCase()` | 下划线转驼峰 |
| `isMatch(pattern, url)` | Ant风格路径匹配（`?`/`*`/`**`） |
| `padl(num, size)` | 数字左补零 |
| `joinComma()` | 逗号拼接 |
| `containsAnyIgnoreCase()` | 忽略大小写包含判断 |
| `startWithAnyIgnoreCase()` | 忽略大小写前缀匹配 |
| `convert(input, fromCharset, toCharset)` | 字符集转换 |

#### `StreamUtils` — Stream流工具（329行）

对 Java Stream API 的封装，简化集合操作：

| 方法 | 说明 |
|------|------|
| `filter(collection, predicate)` | 过滤集合 |
| `findFirst()` / `findFirstValue()` | 查找第一个匹配元素 |
| `findAny()` / `findAnyValue()` | 查找任意匹配元素 |
| `join(collection, function)` | 拼接集合元素为字符串 |
| `sorted(collection, comparator)` | 排序 |
| `toIdentityMap()` | 集合转 Map（值类型不变） |
| `toMap(collection, keyFn, valueFn)` | 集合转 Map（自定义Key/Value） |
| `groupByKey()` | 按单Key分组 |
| `groupBy2Key()` | 按双Key双层分组 |
| `group2Map()` | 按双Key分组并取单值 |
| `toList(collection, function)` | 集合泛型转换 |
| `toSet(collection, function)` | 集合转Set |
| `merge(map1, map2, mergeFn)` | 合并两个Map |

> **注意**: 所有返回 List 的方法使用 `Collectors.toList()` 而非 `.toList()`，因为后者返回不可变 List，会导致序列化问题。

#### `ServletUtils` — HTTP请求工具（290行）

继承 Hutool `JakartaServletUtil`，封装 `RequestContextHolder`：

| 方法 | 说明 |
|------|------|
| `getRequest()` / `getResponse()` | 获取当前请求/响应对象 |
| `getParameter()` / `getParameterToInt()` / `getParameterToBool()` | 获取请求参数（支持类型转换） |
| `getParams()` / `getParamMap()` | 获取所有参数 |
| `getHeader()` / `getHeaders()` | 获取请求头 |
| `getClientIP()` | 获取客户端IP（支持代理穿透） |
| `renderString()` | JSON响应渲染 |
| `isAjaxRequest()` | Ajax请求判断 |
| `urlEncode()` / `urlDecode()` | URL编解码 |
| `getSession()` | 获取Session |

#### `DateUtils` — 日期工具（378行）

封装日期处理操作，基于 Hutool 的 `DateUtil`。

#### `SpringUtils` — Spring上下文工具（67行）

实现 `ApplicationContextAware`，提供静态方式获取 Spring Bean：

```java
// 获取Bean
MyService service = SpringUtils.getBean(MyService.class);

// 判断是否虚拟线程
boolean isVirtual = SpringUtils.isVirtual();
```

#### `MapstructUtils` — 对象映射工具（93行）

封装 MapStruct-Plus，提供便捷的对象转换方法：

```java
// 单个对象转换
UserVo vo = MapstructUtils.convert(user, UserVo.class);

// 集合转换
List<UserVo> voList = MapstructUtils.convert(userList, UserVo.class);
```

---

### 4.2 专项工具类

| 工具类 | 说明 |
|--------|------|
| `DesensitizedUtils` | 数据脱敏工具（身份证/手机号/地址/邮箱/银行卡等） |
| `MessageUtils` | 国际化消息获取（基于 `MessageSource`） |
| `NetUtils` | 网络工具（获取本机IP、主机名等） |
| `ObjectUtils` | 对象工具（空判断、类型转换等） |
| `TreeBuildUtils` | 树结构构建工具（将平铺列表构建为树形结构） |
| `ValidatorUtils` | 手动触发 Bean Validation 校验 |
| `file/FileUtils` | 文件操作工具（获取扩展名、文件类型等） |
| `file/MimeTypeUtils` | MIME类型常量（常见文件类型映射） |
| `ip/AddressUtils` | IP地址解析（调用 ip2region） |
| `ip/RegionUtils` | ip2region 封装（离线IP定位，支持多级缓存） |
| `reflect/ReflectUtils` | 反射工具（获取字段值、方法调用等） |
| `regex/RegexUtils` | 正则校验工具 |
| `regex/RegexValidator` | 正则校验器（手机号/邮箱/身份证/URL等格式验证） |
| `sql/SqlUtil` | SQL工具（防止SQL注入，转义特殊字符） |

---

## 5. 配置类详解

### 5.1 `ThreadPoolConfig` — 线程池配置

```java
核心线程数 = CPU核心数 + 1

ScheduledExecutorService:
  - daemon线程（JVM退出时自动销毁）
  - 支持虚拟线程（JDK21 开启 virtual.enabled=true 时自动切换）
  - 拒绝策略: CallerRunsPolicy（调用者线程执行）
  - 异常处理: afterExecute 自动捕获并打印异常

销毁策略（@PreDestroy）:
  1. shutdown() → 停止接收新任务，尝试完成已存在任务
  2. awaitTermination(120s) → 等待120秒
  3. shutdownNow() → 强制取消未完成任务
  4. awaitTermination(120s) → 再等待120秒
  5. 日志输出 "Pool did not terminate"
```

### 5.2 `ValidatorConfig` — 校验框架配置

配置 `Validator` 工厂，启用参数名发现（`-parameters` 编译选项配合），使校验失败时能显示真实参数名。

### 5.3 `ApplicationConfig` — 应用基础配置

配置 Spring 应用的基础设置，如包扫描等。

---

## 6. 自定义校验体系

### 6.1 分组校验

```java
AddGroup      ← 新增操作校验分组
EditGroup     ← 编辑操作校验分组
QueryGroup    ← 查询操作校验分组
```

使用方式：
```java
@NotNull(groups = AddGroup.class)
@Null(groups = EditGroup.class)
private Long id;
```

### 6.2 字典值校验 — `@DictPattern`

校验字段值是否为有效的字典项：

```java
@DictPattern(dictType = "sys_user_sex", separator = ",")
private String sex;
```

校验器 `DictPatternValidator` 会查询 Redis 缓存中的字典数据，验证字段值是否在合法字典值范围内。

### 6.3 枚举值校验 — `@EnumPattern`

校验字段值是否为有效的枚举值：

```java
@EnumPattern(enumClass = UserStatus.class, method = "getCode")
private String status;
```

### 6.4 XSS校验 — `@Xss`

自定义校验注解，配合 `XssValidator` 检测并过滤 XSS 攻击脚本。

---

## 7. 设计模式与亮点

### 7.1 设计模式应用

| 模式 | 体现 |
|------|------|
| **策略模式** | `LoginBody` 抽象基类 + 5种登录策略子类 |
| **模板方法** | `BaseException` 定义异常基础结构，子类扩展 |
| **工厂方法** | `R<T>` 静态工厂方法创建响应 |
| **观察者模式** | `ProcessEvent` 系列事件类，基于 Spring Event |
| **建造者模式** | `ServiceException` 链式 setter 返回自身 |

### 7.2 设计亮点

1. **缓存键命名规范**: `cacheNames#ttl#maxIdleTime#maxSize#local` 格式，一个注解控制缓存策略
2. **Stream API 封装**: `StreamUtils` 统一处理空集合，避免 NPE，注释提醒避免 `.toList()` 序列化陷阱
3. **异常支持占位符**: `ServiceException` 支持 `{}` 占位符，基于 Hutool StrFormatter
4. **虚拟线程适配**: `ThreadPoolConfig` 自动适配 JDK21 虚拟线程
5. **线程池优雅关闭**: 三级停止策略（shutdown → shutdownNow → 强制退出）
6. **DTO 解耦设计**: 跨模块传递使用 DTO，避免模块间实体直接依赖
7. **事件驱动**: 工作流事件通过 Spring Event 解耦，模块间无需直接引用
8. **字典值校验注解**: `@DictPattern` 将字典校验从业务代码中抽离，声明式校验
9. **Ant路径匹配**: `StringUtils.isMatch()` 支持 `?`/`*`/`**` 通配符，用于权限路径匹配
10. **ip2region 离线定位**: 无需外部服务即可解析IP地址归属地

---

## 8. 模块依赖关系

```
ruoyi-common-core
│
├── 被依赖方（核心基础）
│   ├── ruoyi-common-mybatis     ← 使用 DTO、常量、异常
│   ├── ruoyi-common-redis       ← 使用 CacheNames、常量
│   ├── ruoyi-common-satoken     ← 使用 LoginUser、异常
│   ├── ruoyi-common-security    ← 使用 LoginUser、常量
│   ├── ruoyi-common-web         ← 使用 R、异常、工具类
│   ├── ruoyi-common-log         ← 使用常量、工具类
│   ├── ruoyi-common-tenant      ← 使用 TenantConstants
│   ├── ruoyi-common-oss         ← 使用 OssDTO
│   ├── ruoyi-common-excel       ← 使用工具类
│   └── ruoyi-modules/*          ← 全面使用
│
└── 依赖方（外部依赖）
    ├── spring-context-support
    ├── spring-web
    ├── spring-boot-starter-validation
    ├── spring-boot-starter-aop
    ├── commons-lang3
    ├── hutool-core / hutool-http / hutool-extra
    ├── mapstruct-plus
    ├── ip2region
    └── lombok
```

---

## 9. 总结

`ruoyi-common-core` 是整个 RuoYi-Vue-Plus 框架的**基础设施层**，承担以下核心职责：

| 职责 | 实现 |
|------|------|
| **统一规范** | `R<T>` 响应体、常量体系、异常体系 |
| **身份模型** | `LoginUser` + 5种登录策略 |
| **跨模块通信** | 14个 DTO + 3个事件类 |
| **工具能力** | 20个工具类覆盖字符串/集合/日期/网络/文件/反射/正则/SQL |
| **校验框架** | 分组校验 + 字典值校验 + 枚举校验 + XSS校验 |
| **基础配置** | 线程池（支持虚拟线程）+ 校验器 + 应用配置 |

该模块设计遵循 **"高内聚、低耦合"** 原则，不依赖任何业务模块，仅提供基础能力。所有上层模块通过依赖 core 模块获得统一的基础设施支持，是整个框架稳定运行的根基。
