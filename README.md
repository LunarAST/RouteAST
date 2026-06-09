# LunarAST Sub-Protocol: RouteAST — Network & Routing Contract Specification
## Specification for Static Extraction, Normalization and Alignment Algorithms for Multi-Language Web Routes & HTTP/gRPC Endpoints

**Version**: 0.6.0 — Official Release
**Last Updated**: 2026-06-08
**Parent Specification**: LunarAST Ecosystem Core Spec v1.5 (Final Closed Version)

---

## 1. Purpose & Physical Boundaries
This is the first domain-specific sub-protocol in the LunarAST protocol suite. It defines the standard format for static description of network interfaces (REST API, gRPC, Nginx forwarding rules), specifications for project-level physical fact cache (`actual.json`), and alignment validation algorithms.

### 1.1 Terminology & Naming Conventions
*   **General Naming Rules**: Fully follow the Google Naming & Style Guides. Use `camelCase` for property keys, and `kebab-case` for executable binaries and public configuration files.
*   **File Path**: Strictly follow the path defined in the parent spec: **`.lunar/interfaces.yml`**.
*   **HTTP Method Constraints**: The `method` field **must be uppercase** per RFC 7231. Extended HTTP methods (e.g. `PROPFIND`) are also supported.
*   **gRPC Mapping Rule**: All gRPC calls are uniformly treated as `POST`. Trim leading and trailing slashes from the full path, then split into pure literal segments. Empty literal segments are prohibited.
*   **`extractionMethod` Enumeration**:
    *   `"ast"`: Extracted automatically via Abstract Syntax Tree parsing.
    *   `"escapeHatch"`: Extracted via inline escape-hatch comments.
    *   `"openapi"`: Parsed from OpenAPI documents.
    *   `"manual"`: Manually declared by developers.
*   **No Automatic HEAD Mapping**: `HEAD` and `GET` methods must be explicitly defined separately.
*   **Universal Path Splitting Rule**: When splitting raw path strings into route segment arrays, **all adapters MUST first trim leading and trailing slashes**. Split the remaining string by `/`. This rule eliminates empty literal segments at the start or end of paths, which cause index out-of-bounds errors, parsing failures and alignment deviation. This rule applies to all HTTP paths including REST, gRPC and Nginx forwarding rules.

### 1.2 Scope & Limitations (What This Spec Does NOT Do)
*   Does not parse business logic, track middleware, or perform data flow analysis.
*   Only runs during CI build phase; no runtime traffic interception is provided.

---

## 2. Base IR Schemas (Data Structure Specifications)

### 2.1 Strongly Typed Definition for RouteSegment
*   **`literal`**: `type: "literal"`, `value: String` (non-empty)
*   **`parameter`**: `type: "parameter"`, `name: String`, `rawConstraint: Option<String>` (empty string normalized to `None`; `name` only contains letters, digits and underscores)
*   **`wildcard`**: `type: "wildcard"`. The `value` and `name` fields are ignored.

### 2.2 Full JSON Schema for Single RouteAST Contract
Metadata fields and route attributes are flattened at the same level. **Nested wrapper objects are strictly prohibited**.

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

## 3. Adapter Extraction & Stdout Communication Specification

### 3.1 Line-Delimited JSON (LDJSON) Output Stream Format
*   **Stream Output**: Flush and output one line of JSON immediately after extracting each route. In-memory accumulation is forbidden.
*   **Atomic Validation**: The last line of output **must** be `{"_lunar": {"status": "success", "count": N}}`. The controller verifies that the count value matches the total number of JSON lines. A mismatch triggers the error `ERR_LUNAR_ADAPTER_CRASH`.
*   **Error Handling**: On unrecoverable errors, output `{"_lunar":{"status":"error","message":"..."}}` and exit with a non-zero exit code.
*   **Log Isolation**: Unstructured logs must be redirected to `stderr`.
*   **Process Timeout & Circuit Break**: Default timeout is 30 seconds. The process will be forcibly terminated once timed out.

---

## 4. Normalization Layer: Semantic Normalization Rules
The normalization engine unifies framework-specific syntax variations into standard `RouteSegment` structures, and converts all HTTP `method` values to uppercase.

| Framework | Raw Route | Normalized Segments |
|:---|:---|:---|
| Express | `/api/:id(\\d+)` | `Literal("api")` `Parameter { name: "id", rawConstraint: "\\d+" }` |
| FastAPI | `/api/{id:int}` | `Literal("api")` `Parameter { name: "id", rawConstraint: "int" }` |
| Axum | `/api/:id` | `Literal("api")` `Parameter { name: "id", rawConstraint: None }` |
| Gin | `/api/*filepath` | `Literal("api")` `Wildcard` |
| gRPC | `/grpc.health.v1.Health/Check` | `[Literal("grpc.health.v1.Health"), Literal("Check")]` |

Normalized data will be persisted to `.lunar/.interfaces-autogen.json`.

---

## 5. Alignment Layer: Multi-Dimensional Comparison & Short-Circuit Alignment Algorithm

### 5.1 Position-Based & Ordinal Alignment Algorithm
1.  **Path Length Check**: Unless the last segment is a `wildcard`, the two `segments` arrays must have identical length.
2.  **Segment-by-Segment Scan (based on provider side)**:
    *   Both segments are `literal`: The `value` fields must be exactly equal.
    *   Provider: `parameter`, Consumer: `literal`: **Match succeeded**. Add warning: `"Heuristic Match"`. Parameter name comparison is skipped.
    *   Provider: `literal`, Consumer: `parameter`: **Match failed**.
    *   Both segments are `parameter`: **Alignment succeeded**. Proceed to compare parameter names.
    *   Provider: `wildcard`: **Terminate segment scan immediately**. Match succeeded; all remaining segments on the consumer side are fully matched.
    *   Non-trailing `wildcard`: The consumer segment at the same position must also be `wildcard`, otherwise the match fails.

### 5.2 Contract Status Short-Circuit Evaluation Algorithm
**Priority Chain**: `Unverified > MethodMismatch > Orphaned > (ParamNameMismatch or Aligned)`

The `Unused` status is generated by the global post-processor and excluded from this priority chain.

### 5.3 Semantic Definition of `get_aligned_parameter_names`
Compare the `segments` array of the current route against the peer route. Collect the `name` field of segments **only when both routes have a `parameter` type at the same index**. Since path structure is already validated by segment scanning, the two resulting parameter name lists have identical length and one-to-one correspondence.

### 5.4 Alignment Status Enumeration
```rust
pub enum AlignmentStatus {
    Unverified,
    MethodMismatch { client_method: String, server_method: String },
    Orphaned,
    ParamNameMismatch { client_names: Vec<String>, server_names: Vec<String> }, // Non-blocking issue
    Aligned,
    // Unused status is generated by global post-processor
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

Example 2 (Parameter Name Mismatch):
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

## 6. Single Project Physical Fact Manifest (`actual.json`) Specification

### 6.1 Full Schema for `route-ast-actual.json`
Elements inside `exposed` and `consumed` arrays **must** contain these fields: `path`, `method`, `segments`, `sourceFile`, `lineNumber`, `extractionMethod`.

The `targetProject` field in `consumed` entries is required. Fill in `"unknown"` if the target project cannot be determined. Entries marked as `unknown` will be ignored during global aggregation, but retained in `lunar-map.json` for frontend visualization.

---

## 7. Diagnostic Command Behavior Specification

### 7.1 Exit Code Rules for `lunar doctor`
Exit codes are determined **only by blocking issues**: `Unverified`, `MethodMismatch`, `Orphaned`.
`ParamNameMismatch` and `Aligned` do not affect exit codes; they are only printed as diagnostic information.

---

## 8. Future Roadmap
*   **v1.0**: Introduce normalization logic for regular expressions inside `rawConstraint`.
*   Continuously track RFC updates to maintain full compatibility with standard HTTP methods.

---

## Appendix A: Implementation Checklist (SOP)
- [ ] Adapter outputs LDJSON stream with final status marker and strict count validation for atomicity.
- [ ] Trim leading and trailing slashes for all raw paths before parsing; eliminate empty leading literal segments.
- [ ] Normalize empty `rawConstraint` string value to `None`.
- [ ] Normalize all HTTP `method` values to uppercase in normalization engine.
- [ ] Terminate segment scan immediately and match all remaining segments when a `wildcard` is detected on the provider side.
- [ ] Implement `get_aligned_parameter_names` in alignment engine for precise filtering of asymmetric parameter segments; add unit tests covering heuristic match scenarios.
- [ ] Treat `ParamNameMismatch` and `Aligned` as successful states in CI pipelines; do not block builds. Reserve them for visualization and `lunar diff` reports.
- [ ] Return `Unverified` for failed or stale pre-checks; implement complete failure isolation.
- [ ] Use `.yml` as the file extension for all project-level configuration files.
- [ ] Adapter supports outputting error status marker on failure.
- [ ] Keep entries with `targetProject: "unknown"` during aggregation; mark them on frontend while excluding from alignment logic.
- [ ] Restrict exit codes of `lunar doctor` to `Unverified`, `MethodMismatch` and `Orphaned` only.

---

> *"Contracts come first, with zero tolerance for deviation. Unify routing dialects across multiple languages, and achieve non-intrusive, deterministic network alignment."*
