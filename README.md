# LunarAST Sub-Protocol: RouteAST Synchronous Network and Routing Contract Specification

**——Static Extraction, Normalization, and Alignment Algorithm Specification for Multi-Language Web Routing and HTTP/gRPC Endpoints**

**Version**: 0.6.0 — Official Release
**Last Updated**: 2026-06-08
**Mother Specification**: LunarAST Ecosystem Mother Specification v1.5 (Final Closed-Loop Edition)

---

## 1. Purpose and Physical Boundaries

This specification is the first sub-domain contract specification of the `LunarAST` protocol family, defining the static description format for synchronous network interfaces (REST API, gRPC, Nginx forwarding rules), the project-level physical fact cache (`actual.json`) output specification, and the alignment verification algorithm.

### 1.1 Terminology and Naming Conventions
*   **Unified Naming**: Full compliance with Google Naming & Style Guides. Property key names use `camelCase`; executable binaries and public configuration file names use `kebab-case`.
*   **File Path**: Strictly adheres to **`.lunar/interfaces.yml`** as defined by the mother specification.
*   **Method Constraints**: The `method` property value is forcibly uppercase, following RFC 7231, with support for common extension methods (e.g., `PROPFIND`).
*   **gRPC Mapping Standard**: gRPC interfaces are uniformly treated as the `POST` method. Their full path (e.g., `/grpc.health.v1.Health/Check`) must first have leading and trailing slashes stripped, then be split by `/` into all-literal segments (e.g., `["grpc.health.v1.Health", "Check"]`), physically preventing empty literal segments at the head that could cause index out-of-bounds and parsing errors.
*   **`extractionMethod` Enumeration**:
    *   `"ast"`: Automatically extracted via syntax tree analysis.
    *   `"escapeHatch"`: Extracted via single-line directive comments like `// lunar:consume`.
    *   `"openapi"`: Parsed from project OpenAPI contract declaration files.
    *   `"manual"`: Manually declared in `interfaces.yml`.
*   **No Automatic HEAD Mapping**: The `HEAD` method and `GET` must be explicitly declared separately.

### 1.2 Responsibility Boundaries (What It Does Not Do)
*   Does not parse business logic; does not track middleware logic; does not perform cross-function data-flow analysis.
*   Operates only during the CI build phase; does not provide runtime traffic gateway interception or dynamic matching.

---

## 2. Data Structure Specification (Base IR Schemas)

### 2.1 RouteSegment Strongly Typed Definition

*   **`literal`**: `type: "literal"`, `value: String` (non-empty)
*   **`parameter`**: `type: "parameter"`, `name: String`, `rawConstraint: Option<String>` (empty string normalized to `None`; `name` should only contain alphanumeric characters and underscores)
*   **`wildcard`**: `type: "wildcard"`; `value` and `name` are forcibly ignored during normalization

### 2.2 Complete JSON Representation of a Single Route Contract (RouteAST)

Metadata fields are flattened at the same level as routing properties; **nesting wrapper objects is strictly forbidden**:

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

### 2.3 Path Serialization Standard
*   `literal` segment → `/{value}`
*   `parameter` segment → `/{name}`
*   `wildcard` segment → `/*`

---

## 3. Adapter Extraction and Stdout Communication Specification

### 3.1 Line-Delimited JSON (LDJSON) Output Stream Format

*   **Streaming Output**: Upon extracting and structuring each route, it must be immediately written to `stdout` and `flush`ed; accumulating the entire dataset in memory is strictly forbidden.
*   **Atomicity Verification**: The final line must be the marker `{"_lunar": {"status": "success", "count": N}}`. The control layer verifies that the actual number of parsed lines equals `count`; if they mismatch, it triggers `ERR_LUNAR_ADAPTER_CRASH` and discards all output from this round.
*   **Error Handling**: If the adapter encounters an unrecoverable error, it should output `{"_lunar":{"status":"error","message":"..."}}` and exit with a non-zero exit code.
*   **Log Isolation**: Unstructured logs are redirected to `stderr`.
*   **Process Timeout Circuit Breaker**: A default 30-second execution limit is set for each adapter process; upon timeout, it is forcibly killed.

---

## 4. Confirmation Layer: Semantic Normalization Rules

The confirmation engine forcibly converts the dialect features of multi-language frameworks into the standard `RouteSegment` representation, and uniformly converts the `method` field to uppercase.

| Framework | Original Route | Normalized segments |
|:---|:---|:---|
| Express | `/api/:id(\\d+)` | `Literal("api")` `Parameter { name: "id", rawConstraint: "\\d+" }` |
| FastAPI | `/api/{id:int}` | `Literal("api")` `Parameter { name: "id", rawConstraint: "int" }` |
| Axum | `/api/:id` | `Literal("api")` `Parameter { name: "id", rawConstraint: None }` |
| Gin | `/api/*filepath` | `Literal("api")` `Wildcard` |
| gRPC | `/grpc.health.v1.Health/Check` | `[Literal("grpc.health.v1.Health"), Literal("Check")]` |

The confirmation engine persists this normalized data to `.lunar/.interfaces-autogen.json`.

---

## 5. Alignment Layer: Multi-Dimensional Comparison and Short-Circuit Alignment Algorithm

### 5.1 Position-based & Ordinal Alignment Algorithm

1.  **Path Length Alignment**: Unless the provider's end is a `wildcard`, the `segments` lengths of both sides must be equal.
2.  **Position-by-Position Scanning** (using the provider as the baseline):
    *   Both `literal`: `value` must be exactly equal.
    *   Provider `parameter`, consumer `literal`: **Match succeeded**; attach `warning: "Heuristic Match"` and bypass parameter name comparison.
    *   Provider `literal`, consumer `parameter`: **No match**.
    *   Both `parameter`: **Aligned successfully**; subsequent parameter name comparison applies.
    *   Provider `wildcard`: **Immediately terminate position-by-position scanning**, match succeeds, consuming all remaining segments from the consumer.
    *   Non-final `wildcard`: Requires the consumer to also have a `wildcard` at the same position; otherwise, no match.

### 5.2 Contract Status Short-Circuit Evaluation Algorithm

**Priority Chain**: `Unverified > MethodMismatch > Orphaned > (ParamNameMismatch or Aligned)`

`Unused` is generated by the global post-processor and is not in this chain.

### 5.3 `get_aligned_parameter_names` Semantic Definition

Iterates segment-by-segment comparing the `segments` arrays of `self` and `other`. **Only when both routes are of `parameter` type at the same index position** is the `name` attribute of the segment at that position collected into the result list. Because the path structure has been guaranteed consistent through position-by-position comparison, the collected parameter name lists from both sides are necessarily equal in length and correspond one-to-one.

### 5.4 Contract Status Enumeration

```rust
pub enum AlignmentStatus {
    Unverified,
    MethodMismatch { client_method: String, server_method: String },
    Orphaned,
    ParamNameMismatch { client_names: Vec<String>, server_names: Vec<String> }, // Non-blocking
    Aligned,
    // Unused is generated by the global post-processor
}
```

### 5.5 Final Alignment Entry Construction Specification

Example 1 (Heuristic Match):
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

Example 2 (Parameter Name Difference):
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

## 6. Single-Project Physical Fact Manifest (`actual.json`) Output Specification

### 6.1 `route-ast-actual.json` Complete Schema

Elements in the `exposed` and `consumed` arrays must include `path`, `method`, `segments`, `sourceFile`, `lineNumber`, `extractionMethod`. The `targetProject` in `consumed` is required; if it cannot be determined, fill with `"unknown"`. During central confluence, such entries are ignored for alignment but retained in `lunar-map.json` for frontend indication.

---

## 7. Diagnostic Command Behavior Specification

### 7.1 `lunar doctor` Exit Code Rules

The exit code is determined solely by blocking anomalies (`Unverified`, `MethodMismatch`, `Orphaned`). `ParamNameMismatch` and `Aligned` do not affect the exit code and are output only as diagnostic information.

---

## 8. Future Version Planning

*   **v1.0**: Introduce semantic normalization of `rawConstraint` regexes.
*   Continuously track RFC extensions to maintain full coverage of standard HTTP methods.

---

## Appendix A: Implementation Checklist (SOP)

- [ ] Adapter outputs LDJSON stream, includes end marker, and performs atomic count strict equality verification with actual streamed lines.
- [ ] Adapter strips leading and trailing slashes first when extracting gRPC paths to prevent empty literal segments at the head.
- [ ] Empty string `rawConstraint` normalized to `None` (`null`).
- [ ] Confirmation engine uniformly converts `method` to uppercase.
- [ ] On provider `wildcard` match, immediately stop scanning and consume remaining segments.
- [ ] Alignment engine uses `get_aligned_parameter_names` to precisely filter asymmetric parameter slots during `ParamNameMismatch` evaluation; unit tests cover heuristic match scenarios.
- [ ] `ParamNameMismatch` and `Aligned` are both treated as success states in CI, never blocking builds, only used for visualization and `lunar diff` display.
- [ ] Failure/stale guard returns `Unverified` first, implementing complete failure isolation.
- [ ] Project-level configuration file extension strictly uses `.yml`.
- [ ] Adapter supports outputting `error` status end marker.
- [ ] Entries with `targetProject` as `"unknown"` are ignored during confluence but retained and marked in `lunar-map.json`.
- [ ] `lunar doctor` exit code determined only by `Unverified`, `MethodMismatch`, `Orphaned`.

---

*"Contract supremacy, not a fraction off. Let multi-language routing dialects converge here, achieving zero-intrusion, deterministic network alignment."*
