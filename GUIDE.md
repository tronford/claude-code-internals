# Claude Code Configuration & Environment Reference

Reference on how Claude Code (`@anthropic-ai/claude-code-win32-x64@2.1.332`) resolves user settings, environment variables, and local configuration directories.

---

## 1. User Configuration (`settings.json` & `CLAUDE_CONFIG_DIR`)

By default, Claude Code stores user-level configuration at:
* **Windows:** `%USERPROFILE%\.claude\settings.json`
* **macOS / Linux:** `~/.claude/settings.json`

The base directory can also be overridden by setting `CLAUDE_CONFIG_DIR` (`process.env.CLAUDE_CONFIG_DIR`).

### Environment Variable Injection (`"env"`)

Variables placed inside the `"env"` block of `settings.json` are loaded into the Claude Code process without modifying system-wide shell profiles or unrelated OS applications:

```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50",
    "CLAUDE_CODE_WORKFLOWS": "true",
    "CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS": "1"
  }
}
```

**Inheritance caveat:** while `settings.json`'s `"env"` entries do not modify other desktop applications, they remain part of the Claude Code process environment and **can be inherited by child subprocesses** launched by the session (such as bash, PowerShell, or local tool/MCP processes), subject to Claude Code's subprocess scrubbing/allowlist rules.

---

## 2. How `CLAUDE_INTERNAL_FC_OVERRIDES` Behaves

In `v2.1.332`, `getEnvironmentOverrides()` on the GrowthBook feature-flag store is compiled as `return null`.

As a result:
- Setting `CLAUDE_INTERNAL_FC_OVERRIDES` in `settings.json` or your shell **does not** override GrowthBook feature-flag lookups (`b()`, `On()`, `zt()`, `xae()`, etc.) in an unmodified official binary.
- Outside the GrowthBook flag store, `process.env.CLAUDE_INTERNAL_FC_OVERRIDES` is still read directly during startup org-gate handling (`orgConfigGateFlagOnHand`) and referenced in subprocess/background-job allowlists.

---

## 3. Native Environment Variables Observed in `v2.1.332`

| Environment Variable | Observed Conditions & Effect in Binary |
|---|---|
| `CLAUDE_CONFIG_DIR` | Overrides the default `.claude` configuration directory path (`process.env.CLAUDE_CONFIG_DIR`). |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Parsed as a float in the range `(0, 100]` to lower the automatic compaction threshold percentage, capped at the model's default threshold. |
| `CLAUDE_CODE_WORKFLOWS` | `false` disables workflows; `true` sets both `available` and `defaultOn` to the result of `b("tengu_workflows_enabled", !0)` (still subject to org/settings gates). |
| `CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS` | Required (truthy) alongside `tengu_observer_agents_enabled` and `!Ufe()` (background agent launches enabled) to activate observer agents. |
| `CLAUDE_CODE_LOOP_PERSISTENT` | When truthy, selects the persistent loop preamble; when false/unset, falls back to `b("tengu_kairos_loop_persistent", !1)`. |
| `CLAUDE_CODE_LOOP_KEEPALIVE` | Checked with `!== void 0`: any defined value (including `false`) overrides `tengu_kairos_loop_keepalive`. |
| `CLAUDE_CODE_SUBAGENT_CACHE_EVICT` | Checked only after `yw() && yDe()` (first-party/provider preconditions pass); when truthy returns `true`, when false/unset falls back to `tengu_subagent_cache_evict`. |
| `CLAUDE_CODE_PLUGIN_BINARY_ASSETS` | When truthy enables plugin binary assets; when false/unset falls back to `tengu_plugin_binary_assets`. |
| `CLAUDE_CODE_DISABLE_ULTRATHINK` | When truthy disables `ultrathink` keyword handling even if `tengu_turtle_carbon` is enabled. |
