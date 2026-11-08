# Condition Expression Engine

As introduced in "[Chap8 Request Routing Mechanism of BFE](../../../en_us/design/route/route.md)", both the traffic routing rules of BFE and the rules of the extension modules use "conditions" to describe request matching logic. This chapter introduces the condition expression engine (bfe_expr) introduced in v1.9.0: it coexists with the original condition engine in a dual-track manner, and greatly extends the expressiveness of conditions while keeping existing configurations fully compatible.

## Background

BFE's original condition system is based on a goyacc parser with more than 60 built-in condition primitives (such as `req_path_in()`, `req_header_in()`, etc.), combined only with `&&`/`||`/`!` boolean operators. This design is simple and reliable, but has a clear expressiveness ceiling:

- Adding a new matching dimension requires modifying the Go code and adding a new primitive;
- Arithmetic, numeric ranges and multi-value predicates cannot be expressed;
- Intermediate results cannot be passed between rules.

## Dual-Engine Architecture

The condition expression engine is built on [expr-lang/expr](https://expr-lang.org/) and runs as a second engine alongside the original engine. All condition consumers (the routing tables and the rules of each mod_* module) are dispatched through the unified `condition.Build` entry, so no changes are required on the module side.

At configuration load time:

- Dispatch by a process-level mode switch (`ConditionEngineMode` in the `[Server]` section of `bfe.conf`):
  - `LegacyFirst` (default): the original engine parses first; only when it fails (new-style expressions or mixed old/new expressions) does the expr engine take over;
  - `ExprCompatible`: the expr engine with automatic rewriting of legacy primitive names (old and new can be mixed);
  - `ExprOnly`: the pure expr engine; legacy primitive names are treated as ordinary functions.
- Legacy primitives are not translated at the string level; instead, "functional compatibility" is used: each legacy primitive is implemented as a method of the expression environment `Env`, and `legacyPatcher` rewrites the primitive names to method names before type checking.
- Expressions are compiled at configuration load time (`expr.Compile` with full type checking): type errors, unknown functions and non-bool results are exposed as configuration errors at load time, not as runtime failures.

At runtime, the expression environment is constructed once per request (`NewEnv(req)`, cached in the request context), and all conditions and action-parameter expressions of the same request share this environment, guaranteeing per-request consistency of user variables, random sources, etc.

## Expression Environment

The data-access functions available to expressions are organized by sub-environment:

| Sub-environment | Capabilities |
| --- | --- |
| BaseEnv | Basic request/response/session/routing/backend address accessors (ReqPath(), ReqHeader(), ResCode(), etc.) |
| TimeEnv | Current time and time-range checks (Now(), InTimeRange(), etc.) |
| HashEnv | Hashing and CIDR checks (Hash(), IpIn(), Rand(), etc.) |
| VarEnv | User variable read/write (UsrVarSetInt(), UsrVarGetStr(), etc.), stored in the request-level Context and passable across rules and modules |
| RedisEnv | In-expression Redis reads (see "Dynamic Data Sources" below) |
| InnerEnv | Functional forms of the existing `%variable` built-in variables |
| LegacyEnv | Same-name methods for all legacy condition primitives |

Evaluation conventions:

- The result of a condition expression must be bool; when data is missing, accessor functions return zero values (empty string/0/empty list) instead of raising errors;
- On a runtime evaluation error, the condition of that rule is treated as false and a warn log is emitted (with logid and the expression string); other rules and the request processing are unaffected;
- Condition expressions allow controlled side effects (e.g. sequence expressions like `UsrVarSetInt("k", v); false`). Rules are evaluated in configuration order and each rule's condition is evaluated at most once per request, so "a front seed rule writes a variable, later rules read it" is a reliable cross-rule passing mechanism.

## Action Parameter Expressions

Since v1.9.0, action parameters of some modules support embedded expressions with the `=` prefix, evaluated dynamically at request processing time. Parameter writings fall into three forms:

| Form | Semantics |
| --- | --- |
| `=expr` | Expression; the result must be a string; compiled and type-checked at load time, and a failure fails the configuration load |
| `==lit` | Escaped literal (`==foo` means the literal value `=foo`) |
| Others | Plain literals, used as-is with zero overhead |

Supported consumers: parameter positions of the shared actions (mod_rewrite, mod_prison) except Header names; `URL_SET`/`URL_PREFIX_ADD` of mod_redirect; Header value parameters of mod_header. A parameter whose expression fails to evaluate is treated as an empty string with a warn log, without blocking the other parameters of the same action or subsequent actions.

For example, the following rewrite rule dynamically rewrites the request Host to "original host plus a fixed suffix":

```json
{
    "Cmd": "HOST_SET",
    "Params": ["=req_host() + '.inner.example.com'"]
}
```

## Dynamic Data Sources

The expression engine supports reading external data sources inside expressions:

- **Redis** (available): `GetRedisClient(addr)` returns a process-level singleton client per address (default timeouts: conn 100ms / read 300ms / write 300ms), supporting the `Get`/`GetInt`/`HGetAll` read operations. On timeout or error, zero values are returned (no error, no retry) to avoid failure amplification.
- **HTTP services, global variables, configuration tables**: the interfaces are defined (HttpEnv/GlbEnv/ConfEnv) and the current version ships stub implementations — calls log a warning and return zero values; they will be opened up in future releases.

Note that Redis reads inside expressions are synchronous network IO and occupy the goroutine processing the request; evaluate QPS and Redis capacity before use. Also, writing Redis addresses directly in expressions means the authors of rule configurations need to be trusted.

## Mode Switch and Migration

`ConditionEngineMode` is configured in the `[Server]` section of `bfe.conf`, defaulting to `LegacyFirst`: under this mode, existing configurations behave exactly as in older versions (if the original engine parses successfully, the legacy semantics apply), so you can upgrade with zero risk and try the new expressions gradually; new-style configurations automatically fall to the expr engine. Changing the mode requires a process restart; hot reload does not change the mode.

There are a few intentional semantic differences between the two engines (e.g. string escapes are unified to Go semantics; `res_code_in` compares strings while the new-style `ResCode()` compares integers). Under the default mode these differences do not take effect; when migrating existing configurations under `ExprCompatible`/`ExprOnly`, pay attention to these differences.

## links
Previous: [Chap34 How to Develop BFE Extension Module](../how_to_write_module/how_to_write_module.md)
