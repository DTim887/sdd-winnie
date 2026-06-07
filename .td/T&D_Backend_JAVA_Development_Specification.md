# T&D Backend JAVA Development Specification

## 1. 项目包规范

根包命名 `com.disney.shdr.[service-name]`

### 1.1 项目目录结构总览

```
com.disney.shdr.[service-name]
  │
  ├── [ServiceName]Application.java          # 启动类
  │
  ├── api/                                   # 对外接口层
  │   ├── controller/                        # REST Controller
  │   ├── request/                           # 请求 DTO（入参）
  │   └── response/                          # 响应 DTO（出参）
  │
  ├── service/                              # 业务逻辑层
  │   ├── [XxxService].java                 # 接口定义
  │   └── impl/                             # 接口实现
  │
  ├── repository/                            # 数据访问层
  │   ├── [XxxRepository].java              # JPA / MyBatis Mapper 接口
  │   └── entity/                           # 数据库实体（对应表结构）
  │
  ├── domain/                               # 领域模型（业务对象，独立于 DB 结构）
  │
  ├── infrastructure/                       # 基础设施
  │   ├── config/                           # Spring 配置类（Bean、MVC、Security 等）
  │   ├── client/                           # 外部/第三方服务调用（Feign / RestTemplate）
  │   ├── messaging/                        # MQ Producer 封装、配置、序列化
  │   └── cache/                            # 缓存操作封装
  │
  ├── common/                               # 通用组件（无业务依赖）
  │   ├── constants/                        # 常量
  │   ├── enums/                            # 枚举
  │   ├── exception/                        # 自定义异常类
  │   ├── response/                         # 统一响应体封装（Result<T>）
  │   └── util/                             # 工具类
  │
  └── filter/                               # 过滤器 / 拦截器
```

### 1.2 各层职责说明

| 层 | 职责 | 禁止事项 |
|---|---|---|
| api/controller | 接收请求、参数校验、调用 service、组装响应 | 禁止含业务逻辑 |
| api/request | 入参 DTO，含 @Valid 校验注解 | 禁止直接传入 service 内部 |
| api/response | 出参 DTO，面向调用方 | 禁止暴露内部 entity 字段 |
| service | 业务逻辑编排，跨 repository 事务 | 禁止直接操作 HTTP 上下文 |
| repository | 数据库 CRUD，只操作 entity | 禁止含业务判断 |
| repository/entity | 与数据库表一一对应 | 禁止直接作为 API 响应体返回 |
| domain | 纯业务对象，不依赖 ORM 注解 | 可选，复杂业务场景使用 |
| infrastructure/client | 封装外部调用，统一异常转换 | 禁止直接被 controller 调用 |
| common | 无业务依赖的纯工具组件 | 禁止引用上层业务包 |

### 1.3 命名约定

| 类型 | 后缀 | 示例 |
|---|---|---|
| Controller | Controller | DeviceController |
| Service 接口 | Service | DeviceService |
| Service 实现 | ServiceImpl | DeviceServiceImpl |
| Repository | Repository | DeviceRepository |
| 数据库实体 | Entity 或 DO | DeviceDO |
| 请求 DTO | Request | CreateDeviceRequest |
| 响应 DTO | Response / VO | DeviceResponse |
| 外部调用封装 | Client | CdnClient |
| 自定义异常 | Exception | DeviceNotFoundException |

---

## 2. API 设计优先

### 2.1 REST API 设计规范

所有 API 需集成 Swagger UI。

- URI 只使用名词，禁止包含动词或操作词
- 资源路径使用复数形式（如 `/devices`，`/configurations`）
- 子资源通过父资源路径嵌套（如 `/devices/{id}/configurations`）
- URI 版本号使用前缀格式：

```
/api/v1/devices
/api/v1/devices/{id}
/api/v1/devices/{id}/configurations
/api/v1/devices/{id}/configurations/{configId}
```

### 2.2 REST API Method 规范

| 操作 | Method | URI 示例 | 说明 |
|---|---|---|---|
| 查询集合 | GET | /devices | 支持分页参数 startIndex、size |
| 查询单个 | GET | /devices/{id} | 返回完整资源详情 |
| 创建资源 | POST | /devices | 成功返回 201 Created |
| 全量更新 | PUT | /devices/{id} | 幂等操作 |
| 删除资源 | DELETE | /devices/{id} | 成功返回 200/202/204 |

- 分页示例：`GET /devices?startIndex=0&size=20`
- 异步删除返回 202，需提供可追踪的 taskId；同步删除返回 200 或 204

### 2.3 统一响应体结构

```
成功（2xx）
{ "code": 200, "msg": "success", "data": { ... } }

分页数据：
{
  "code": 200, "msg": "success",
  "data": {
    "rows": [ { ... } ],
    "pageSize": 10, "pageNum": 1, "total": 100
  }
}

集合数据：
{ "code": 200, "msg": "success", "data": { "list": [ { ... } ] } }

失败（4xx）：
{ "code": 400, "msg": "bad request, http method GET is not allowed" }

服务异常（5xx）
{ "code": 500, "msg": "service error, please try again" }
```

### 2.4 HTTP 状态码规范

| 状态码 | 场景 |
|---|---|
| 200 OK | 请求成功，含响应体 |
| 201 Created | 资源创建成功 |
| 202 Accepted | 异步操作已接受，处理中 |
| 204 No Content | 成功但无响应体（如 DELETE） |
| 400 Bad Request | 请求参数错误或格式非法 |
| 401 Unauthorized | 未认证或 Token 无效 |
| 403 Forbidden | 已认证但无权限 |
| 404 Not Found | 资源不存在 |
| 405 Method Not Allowed | HTTP 方法不支持 |
| 415 Unsupported Media Type | Content-Type 不支持 |
| 500 Internal Server Error | 服务端未知异常 |

### 2.5 版本管理规范

- 采用 URI 版本号方式（推荐）：`/api/v1/...`
- 以下情况必须升级主版本号（Breaking Change）：
  - 响应数据结构变更
  - 请求/响应字段类型变更
  - 删除任何已有接口或字段
- 新增接口或新增响应字段属于非破坏性变更，无需升级主版本

### 2.6 请求头规范

- `Content-Type: application/json`
- `Accept: application/json`
- 自定义私有请求头使用 `x-` 前缀（如 `x-request-id`）
- 支持 `Accept-Encoding: gzip` 压缩，响应中包含 `Content-Encoding: gzip`

### 2.7 无状态原则

- 每个请求必须携带完整认证和上下文信息，服务端不保存任何会话状态
- 会话状态由客户端维护

### 2.8 所有 API 必须集成 Swagger UI

- 每个服务必须引入 `springdoc-openapi-starter-webmvc-ui` 作为运行时依赖
- 每个 Controller 必须使用 `@Tag` 注解标注其逻辑分组名称
- 每个对外暴露的接口方法必须标注 `@Operation(summary = "...")` 描述接口用途
- 每个请求/响应 DTO 必须使用 `@Schema` 注解描述字段含义并提供示例值
- Swagger UI 地址（`/swagger-ui.html`）在所有非生产环境中必须保持可访问；生产环境可通过配置关闭

---

## 3. 统一业务错误码规范

### 3.1 错误码结构

每个 Application 都需要预分配错误码前缀 Application ID（如 `9900`），完整错误码由以下三部分拼接而成：

| 组成部分 | 长度 | 说明 |
|---|---|---|
| Application ID | 4 位数字 | 标识归属应用，每个 Application 预分配号段 |
| Error Type | 1 位字母 | 标识错误来源类型（A / B / C 见下表） |
| Error Code | 4 位数字 | 该类型下的具体错误编号，范围 0001~9999，类别间以 100 步长划分 |

| Type | 含义 | 业务场景 |
|---|---|---|
| A | 客户端错误（Client Error） | 参数错误、用户版本过低、支付超时等由调用方引起的错误 |
| B | 系统错误（System Error） | 业务逻辑错误、程序健壮性不足等由当前系统引起的错误 |
| C | 第三方服务错误（Third-party Error） | CDN 故障、消息投递超时等由外部依赖引起的错误 |

完整格式示例：

```
9900 A 0001
^^^^ ^ ^^^^
|    | └── Error Code: 具体错误编号（4位）
|    └──── Error Type: 错误来源类型（1位字母）
└───────── Application ID: 应用标识（4位）
```

### 3.2 错误响应格式

业务错误码通过统一响应体的 `code` 字段返回，HTTP 状态码仍遵循第 2.3、2.4 章规范。

```json
{
  "code": "9900A0001",
  "msg": "请求参数 [userId] 不能为空",
  "data": null
}
```

### 3.3 使用原则

- 错误码一经发布不得修改含义，只可新增
- 新增错误码须在此文档同步更新分配表
- 错误码统一由枚举类进行维护
- 类别间保留 100 步长，为未来细分预留空间（如 A0001~A0099 为参数类，A0100~A0199 为版本类）

---

## 4. 测试与质量

### 4.1 目标与范围

本规范适用于 Java / Spring Boot 后端服务的开发自测阶段。

- 测试类型：单元测试（Unit Test），不要求集成测试和 E2E 测试
- 测试层级：Service 层（必须）+ Controller 层（必须）
- Repository 层、工具类：可选，有复杂逻辑时建议覆盖

### 4.2 测试框架

| 用途 | 工具 |
|---|---|
| 测试框架 | JUnit 5（junit-jupiter） |
| Mock 框架 | Mockito（mockito-core） |
| Controller 测试 | Spring MockMvc（@WebMvcTest） |
| 断言库 | AssertJ（推荐）/ JUnit 原生 |

### 4.3 Service 层测试规范

#### 测试类结构

```java
@ExtendWith(MockitoExtension.class)          // 纯单元测试，不启动 Spring 容器
class DeviceServiceImplTest {

    @Mock
    private DeviceRepository deviceRepository;

    @InjectMocks
    private DeviceServiceImpl deviceService;

    // ...
}
```

#### 必测场景清单

| 场景类型 | 说明 | 是否必须 |
|---|---|---|
| Happy Path | 正常入参，业务流程走通，返回预期结果 | 必须 |
| 资源不存在 | 查询/更新目标不存在时抛出正确异常 | 必须 |
| 参数边界 | 空值、null、空集合等边界入参的处理 | 必须 |
| 业务规则冲突 | 重复创建、状态不合法等业务约束 | 必须（有业务规则时） |
| 第三方/下游失败 | 外部 client 抛异常时，服务的错误处理行为 | 必须（有外部调用时） |
| 分页/集合为空 | 查询结果为空列表时的返回值 | 建议 |

#### Mock 原则

- 只 Mock 直接依赖，不 Mock 被测类自身的方法
- 外部调用（Client、MQ Producer、Cache）必须 Mock，测试不依赖真实网络/数据库
- 禁止 Mock 静态方法（如需要，说明设计有问题，应重构）

#### 示例

```java
@Test
void getDevice_whenExists_returnsResponse() {
    // Arrange
    DeviceEntity entity = buildDeviceEntity(1L, "printer");
    when(deviceRepository.findById(1L)).thenReturn(Optional.of(entity));

    // Act
    DeviceResponse result = deviceService.getDevice(1L);

    // Assert
    assertThat(result.getId()).isEqualTo(1L);
    assertThat(result.getName()).isEqualTo("printer");
}

@Test
void getDevice_whenNotFound_throwsException() {
    when(deviceRepository.findById(99L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> deviceService.getDevice(99L))
        .isInstanceOf(DeviceNotFoundException.class)
        .hasMessageContaining("99");
}
```

### 4.4 Controller 层测试规范

#### 测试类结构

```java
@WebMvcTest(DeviceController.class)          // 只加载 Web 层，不启动完整容器
class DeviceControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private DeviceService deviceService;     // Mock 掉 Service

    // ...
}
```

#### 必测场景清单

Controller 层不测业务逻辑，只测：路由是否正确、参数校验是否生效、响应体格式是否符合统一规范。

| 场景类型 | 说明 | 是否必须 |
|---|---|---|
| 正常请求 | 合法入参，返回预期 HTTP 状态码和响应体结构 | 必须 |
| 参数校验失败 | 缺少必填字段、格式非法，返回 400 | 必须 |

#### 示例

```java
@Test
void getDevice_whenExists_returns200() throws Exception {
    DeviceResponse response = new DeviceResponse(1L, "printer");
    when(deviceService.getDevice(1L)).thenReturn(response);

    mockMvc.perform(get("/api/v1/devices/1"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.code").value("200"))
        .andExpect(jsonPath("$.data.name").value("printer"));
}

@Test
void createDevice_whenMissingName_returns400() throws Exception {
    String body = """{ "name": "" }""";

    mockMvc.perform(post("/api/v1/devices")
            .contentType(MediaType.APPLICATION_JSON)
            .content(body))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.code").value("9900A0001"));
}
```

### 4.5 覆盖率要求

| 指标 | 目标 | 说明 |
|---|---|---|
| 整体行覆盖率 | ≥ 70% | 重点保障 |
| Service 层行覆盖率 | ≥ 80% | 重点保障 |
| Controller 层行覆盖率 | ≥ 70% | 重点保障 |

- 覆盖率为自检项，不卡 CI 合并，但 PR 描述中须注明当前覆盖率数值
- 若某模块因特殊原因低于目标，PR 描述中须说明原因

查看覆盖率报告：

```
mvn test jacoco:report
# 报告路径：target/site/jacoco/index.html
```

### 4.6 测试命名规范

统一格式：`方法名_场景描述_预期结果`

```java
// 好的命名
void getDevice_whenNotFound_throwsDeviceNotFoundException()
void createDevice_whenNameIsBlank_returns400()
void updateDevice_whenSuccess_returnsUpdatedData()

// 避免
void test1()
void testGetDevice()
void shouldWork()
```

### 4.7 PR 提交前自检清单

提交 PR 前，逐项确认：

- [ ] Service 层核心方法均有 Happy Path 测试
- [ ] Service 层资源不存在、参数异常等错误路径已覆盖
- [ ] Controller 层参数校验（400）已验证
- [ ] Controller 层响应体格式符合统一规范（code/msg/data）
- [ ] 所有测试本地执行通过（`mvn test`）

### 4.8 禁止事项

- 禁止测试中使用 `Thread.sleep()` 等待异步结果，改用 `CompletableFuture` 或 `Awaitility`
- 禁止测试共享可变状态（静态变量等），每个测试必须独立可重复执行
- 禁止在测试代码中写真实外部地址（DB 连接、MQ 地址、第三方 URL）
- 禁止注释掉失败的测试来让构建通过；失败测试必须修复或删除并说明原因
- 禁止只测 getter/setter，覆盖率不应靠无意义测试堆砌

---

## 5. 研发流程

### 5.1 分支策略

- 功能分支命名：`[Jira Ticket ID|Feature]/<工单号>-简短描述`
- 示例：`SHDRP-433819|project-initialization`

---

## 6. 后端 Java 技术栈

该规范仅描述 JAVA 纯后端 API 服务， 不包含前端。可以被复写

|   约束项   | 标准                                       | 覆盖策略   |
| :--------: | :----------------------------------------- | :--------- |
|   技术栈   | JAVA 17+ , Spring Boot 3.5.0+,  Spring Web | 不允许覆盖 |
|  构建工具  | Maven                                      | 不允许覆盖 |
|   数据库   | MySQL 8.0                                  | 不允许覆盖 |
|  ORM框架   | MyBatis Plus                               | 不允许覆盖 |
| 分布式缓存 | Redis 8.6                                  | 不允许覆盖 |
| 消息中间件 | RabbitMQ                                   | 不允许覆盖 |
