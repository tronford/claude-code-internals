# Claude Code Internals (v2.1.332)

An independent static analysis of the GrowthBook feature-flag wiring inside the Windows x64 distribution of **Claude Code** (`@anthropic-ai/claude-code-win32-x64@2.1.332`).

---

## Observed Metrics & Extraction Scope

Static inspection of the official `v2.1.332` executable (`403,083,424` bytes; embedded Anthropic JS chunks with `// (c) Anthropic PBC` headers span roughly `68.4 MB` from offset `0x13b327fa` to `0x17c66f98`):

- **1,061** unique `tengu_*` flag names found in direct string-literal calls to the 10 GrowthBook helpers defined in `chunk-fsawyn72.js` (`b`, `On`, `zt`, `xae`, `sA`, `XWs`, `fpe`, `Ila`, `M8t`, `Wtt`), deduplicated by first observed call site:
  - **504** pass `false` (`!1`) as the fallback argument at that first call site
  - **366** pass `true` (`!0`)
  - **186** pass numbers, strings, objects, or runtime expressions
  - **5** are 1-argument `sA(...)` calls with no 2nd argument (`(no 2nd arg)`)
- **What this count does not include:** flags stored in variables and passed indirectly rather than as string literals. For example, `var NG = "tengu_snappy_noodle"` (`0x13eef56a`) is passed via `zt(NG, we() === "firstParty")` (`0x13eef711`) and therefore falls outside literal matching.
- **Note on fallbacks:** the boolean in a helper call is a per-call fallback, not a single global constant; some flags pass different fallbacks in different places (e.g., `tengu_sdk_mcp_manifests` passes `!0` at `0x162e68c1` and `!1` at `0x162e6c22`).
- Separate from feature-flag reads, static matching of the `l("tengu_...", ...)` telemetry call shape finds **3,156** call sites across **2,103** unique `tengu_*` event names.

---

## Core Flag Architecture

1. **Resolvers (`chunk-fsawyn72.js`):**  
   Core lookups route through `On()` (`getFeatureValueWithSource`), `b()` (`On(...).value`), session-pinned `zt()` / `Ila()` / `fpe()`, wrapper `xae()`, blocking read `XWs()`, and cached gate helpers `sA()`, `M8t()`, `Wtt()`.
2. **Production Override Stub (`getEnvironmentOverrides`):**  
   In the Windows x64 build of `v2.1.332`, `getEnvironmentOverrides()` on the GrowthBook flag store class is compiled to `return null`, blocking local GrowthBook flag overrides via `b()`, `On()`, and `zt()`. Separate from the GrowthBook store, `process.env.CLAUDE_INTERNAL_FC_OVERRIDES` is still read directly during startup org-gate checks (passed to `orgConfigGateFlagOnHand`).

---

## Notable Flags Observed in v2.1.332

1. **Skills over MCP - `tengu_wiggly_crystal`**  
   - **1st-call fallback:** `false` (`0x14087865` / `336,099,429`)  
   - **Observed Behavior:** Gates the `io.modelcontextprotocol/skills` capability during MCP negotiation, subject to additional environment, entrypoint, and policy checks (`Si() && Ag()`, `CLAUDE_CODE_REMOTE`, enterprise/team policy settings).

2. **Autonomous Loop Preamble (Kairos) - `tengu_kairos_loop_persistent`**  
   - **1st-call fallback:** `false` (`0x15ff1f86` / `369,041,286`) | **Env:** `CLAUDE_CODE_LOOP_PERSISTENT`  
   - **Observed Behavior:** Switches the autonomous loop preamble asset from `loopAutonomousPreamble` to `loopAutonomousPreamblePersistent`. Related Kairos flags include `tengu_kairos_loop_keepalive`, `tengu_kairos_push_notifications`, `tengu_kairos_brief`, and cron-related flags (`tengu_kairos_cron` via `xae` at `0x14c16164`).

3. **Thinking Block Resumption - `tengu_thinking_block_resumption`**  
   - **1st-call fallback:** `false` (`0x148abbed` / `344,636,397`)  
   - **Observed Behavior:** Gates resumption of truncated extended-thinking responses (`[thinking-resumption]`), contingent on additional eligibility checks (`YRt(e)`, response resumable flag `n === !0`, and remaining token/context budget checks).

4. **Workflows Engine - `tengu_workflows_enabled`**  
   - **1st-call fallback:** `true` (`0x14045ce1` / `335,830,241`) | **Env:** `CLAUDE_CODE_WORKFLOWS`  
   - **Observed Behavior:** Controls workflow availability and default state alongside org/settings checks (`allow_workflows`, `enableWorkflows`). On `pro` plans without an explicit override, `defaultOn` evaluates `plan !== "pro"` (`false`) even while `available` can be `true`.

5. **Background Observer Agents - `tengu_observer_agents_enabled`**  
   - **1st-call fallback:** `true` (`0x14e569a9` / `350,579,113`) | **Env:** `CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS`  
   - **Observed Behavior:** Gates experimental observer agents and subagent fanout (`tengu_observer_subagent_fanout`). Activation additionally requires `!Ufe()` (background agent launches enabled) and a truthy `CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS`.

6. **Skills Health Dashboard - `tengu_skills_dashboard_enabled`**  
   - **1st-call fallback:** `false` (`0x1736e299` / `389,472,921`)  
   - **Observed Behavior:** Enables fetching `/api/claude_code/skills` to populate `skillHealthMap` (`good`, `warn`, `poor`).

7. **Remote Control Bridge - `tengu_ccr_bridge`**  
   - **1st-call fallback:** `false` (`0x143c2504` / `339,485,956`)  
   - **Observed Behavior:** Evaluates Remote Control eligibility as `HN() && _() && (b("tengu_ccr_bridge", !1) || A())`, requiring first-party API connectivity, subscription/auth state, and policy checks alongside either the flag or eligibility branch `A()`.

---

## Repository Contents

- [FLAGS.md](FLAGS.md) - Static sample of all 1,061 unique `tengu_*` flag names extracted from direct string-literal calls to the 10 GrowthBook helpers.
- [GUIDE.md](GUIDE.md) - Reference on `settings.json`, environment inheritance, `CLAUDE_CONFIG_DIR`, and observed `process.env` checks.

---

## Disclaimer

Independent static analysis of the publicly distributed npm package `@anthropic-ai/claude-code-win32-x64@2.1.332` for educational and interoperability research. Not affiliated with or endorsed by Anthropic. No proprietary binaries or binary patchers are distributed here.
