# LunarAST 子协议：RouteAST 同步网络与路由契约规范

**——面向多语言 Web 路由与 HTTP/gRPC 端点的静态提取、归一化与对齐算法规范**

**版本**：0.6.0 — 正式发布版
**最后更新**：2026-06-08
**母规范**：LunarAST 生态母规范 v1.5（最终闭环版）

---

## 1. 目的与物理边界

本规范是 `LunarAST` 协议家族的第一个子领域契约规范，用于定义同步网络接口（REST API、gRPC、Nginx 转发规则）的静态描述格式、项目级物理事实缓存（`actual.json`）产出规范以及对齐校验算法。

### 1.1 术语与命名约定
*   **统一命名**：全量遵循 Google Naming & Style Guides。属性键名使用 `camelCase`，可执行二进制与公共配置文件使用 `kebab-case`。
*   **文件路径**：严格沿用母规范定义的 **`.lunar/interfaces.yml`**。
*   **方法约束**：`method` 强制大写，遵循 RFC 7231，支持扩展方法（如 `PROPFIND`）。
*   **gRPC 映射**：统一视为 `POST`，路径剔除首尾斜杠后分割为全字面量段，杜绝空字面量段。
*   **`extractionMethod` 枚举**：
    *   `"ast"`：语法树自动提取。
    *   `"escapeHatch"`：逃生舱注释提取。
    *   `"openapi"`：OpenAPI 解析。
    *   `"manual"`：人工声明。
*   **不自动映射 HEAD**：`HEAD` 与 `GET` 必须显式声明。

### 1.2 职责边界（不做什么）
*   不解析业务逻辑，不追踪中间件，不进行数据流分析。
*   仅在 CI 构建期工作，不提供运行时流量拦截。

---

## 2. 数据结构规范 (Base IR Schemas)

### 2.1 路由段 (RouteSegment) 强类型定义

*   **`literal`**：`type: "literal"`, `value: String`（非空）
*   **`parameter`**：`type: "parameter"`, `name: String`, `rawConstraint: Option<String>`（空字符串归一化为 `None`；`name` 仅含字母数字下划线）
*   **`wildcard`**：`type: "wildcard"`，`value` 与 `name` 强制忽略

### 2.2 单条路由契约 (RouteAST) 完整 JSON 表达

元数据字段与路由属性同级扁平化，**严禁**嵌套包装对象：

```json
{
  "method": "POST",
  "segments": [
    { "type": "literal", "value": "api" },
    { "type": "literal", "value": "v1" },
    { "type": "parameter", "name": "userId", "rawConstraint": "\\d+" }
  ],
  "sourceFile": "src/controllers/user_controller.rs",
  "lineNumber": 42,
  "extractionMethod": "ast"
}
```

### 2.3 路径序列化标准
*   `literal` 段 → `/{value}`
*   `parameter` 段 → `/{name}`
*   `wildcard` 段 → `/*`

---

## 3. 适配器提取与 Stdout 通信规范

### 3.1 换行分隔 JSON (LDJSON) 输出流格式

*   **流式输出**：每提取一条路由立即 `flush` 输出一行 JSON，严禁内存堆积。
*   **原子性校验**：最后一行必须为 `{"_lunar": {"status": "success", "count": N}}`，控制层验证 `count` 与实际行数一致，否则触发 `ERR_LUNAR_ADAPTER_CRASH`。
*   **错误处理**：适配器发生不可恢复错误时，输出 `{"_lunar":{"status":"error","message":"..."}}` 并以非零退出码退出。
*   **日志隔离**：非结构化日志重定向至 `stderr`。
*   **进程超时熔断**：默认 30 秒超时，超时强制 Kill。

---

## 4. 确认层：语义归一化规则 (Semantic Normalization)

确认引擎将多语言框架的方言特征强制转换为标准的 `RouteSegment` 表达，同时将 `method` 统一转为大写。

| 框架 | 原始路由 | 归一化后的 segments |
|:---|:---|:---|
| Express | `/api/:id(\\d+)` | `Literal("api")` `Parameter { name: "id", rawConstraint: "\\d+" }` |
| FastAPI | `/api/{id:int}` | `Literal("api")` `Parameter { name: "id", rawConstraint: "int" }` |
| Axum | `/api/:id` | `Literal("api")` `Parameter { name: "id", rawConstraint: None }` |
| Gin | `/api/*filepath` | `Literal("api")` `Wildcard` |
| gRPC | `/grpc.health.v1.Health/Check` | `[Literal("grpc.health.v1.Health"), Literal("Check")]` |

确认引擎将归一化数据持久化到 `.lunar/.interfaces-autogen.json`。

---

## 5. 对齐层：多维比对与短路对齐算法

### 5.1 序数对齐算法（Position-based & Ordinal Alignment）

1.  **路径长度对齐**：除非提供端末尾为 `wildcard`，否则双方 `segments` 长度必须相等。
2.  **逐段位置扫描**（以提供端为基准）：
    *   均为 `literal`：`value` 必须完全相等。
    *   提供端 `parameter`，消费端 `literal`：**匹配成功**，附加 `warning: "Heuristic Match"`，并绕过参数名比对。
    *   提供端 `literal`，消费端 `parameter`：**不匹配**。
    *   均为 `parameter`：**对齐成功**，后续比对参数名。
    *   提供端 `wildcard`：**立即终止逐段扫描**，匹配成功，吞并消费端剩余所有段。
    *   非末尾 `wildcard`：要求消费端同位置也为 `wildcard`，否则不匹配。

### 5.2 契约状态短路评估算法

**优先级链**：`Unverified > MethodMismatch > Orphaned > (ParamNameMismatch 或 Aligned)`

`Unused` 由全局后处理器生成，不在此列。

### 5.3 `get_aligned_parameter_names` 语义定义

逐段比对自身与 `other` 的 `segments` 数组。**只有当两个路由在相同索引位置同为 `parameter` 类型时**，才将该位置自身段的 `name` 属性收集入结果列表。由于路径结构已通过逐段比对保证一致，双方收集到的参数名列表长度必然相等且一一对应。

### 5.4 契约状态枚举

```rust
pub enum AlignmentStatus {
    Unverified,
    MethodMismatch { client_method: String, server_method: String },
    Orphaned,
    ParamNameMismatch { client_names: Vec<String>, server_names: Vec<String> }, // 非阻断
    Aligned,
    // Unused 由全局后处理器生成
}
```

### 5.5 最终对齐条目构造规范

示例一（启发式匹配）：
```json
{
  "clientProject": "myPaymentService",
  "serverProject": "authService",
  "path": "/api/v1/users/{userId}",
  "method": "POST",
  "status": "Aligned",
  "warning": "Heuristic Match"
}
```

示例二（参数名差异）：
```json
{
  "clientProject": "myPaymentService",
  "serverProject": "authService",
  "path": "/api/v1/orders/{orderId}",
  "method": "GET",
  "status": "ParamNameMismatch"
}
```

---

## 6. 单项目物理事实清单 (`actual.json`) 产出规范

### 6.1 `route-ast-actual.json` 完整 Schema

`exposed` 与 `consumed` 数组元素必须包含 `path`、`method`、`segments`、`sourceFile`、`lineNumber`、`extractionMethod`。

`consumed` 中的 `targetProject` 为必填；无法确定时填充 `"unknown"`，中心合流时忽略此类条目但保留在 `lunar-map.json` 中以供前端标示。

---

## 7. 诊断命令行为规范

### 7.1 `lunar doctor` 退出码规则

退出码仅由阻断性异常（`Unverified`、`MethodMismatch`、`Orphaned`）决定。`ParamNameMismatch` 和 `Aligned` 不影响退出码，仅作为诊断信息输出。

---

## 8. 未来版本规划

*   **v1.0**：引入 `rawConstraint` 正则语义归一化。
*   持续跟踪 RFC 扩展，保持对标准 HTTP 方法的完整覆盖。

---

## 附录 A：实现检查清单（SOP）

- [ ] 适配器输出 LDJSON 流，包含结束标记并执行原子性计数强等值校验。
- [ ] gRPC 路径提取时首要执行 Slash 去除过滤，杜绝首位空字面量段。
- [ ] `rawConstraint` 空字符串归一化为 `None`。
- [ ] 确认引擎将 `method` 统一转为大写。
- [ ] 提供端 `wildcard` 匹配时立即终止扫描并吞并剩余段。
- [ ] 对齐引擎采用 `get_aligned_parameter_names` 对非对称参数段执行精确过滤，单元测试覆盖启发式匹配场景。
- [ ] `ParamNameMismatch` 与 `Aligned` 在 CI 中均视为成功状态，不阻断构建，仅用于可视化及 `lunar diff` 差异展示。
- [ ] 失败/陈旧守卫优先返回 `Unverified`，实施完全的失败隔离。
- [ ] 项目级配置文件后缀严格使用 `.yml`。
- [ ] 适配器支持输出 `error` 状态结束标记。
- [ ] `targetProject` 为 `"unknown"` 的条目在合流时忽略但保留并标示。
- [ ] `lunar doctor` 退出码仅由 `Unverified`、`MethodMismatch`、`Orphaned` 决定。

---

*“契约至上，不差分毫。让多语言的路由方言在此归流，实现零侵入、确定性的网络对齐。”*
