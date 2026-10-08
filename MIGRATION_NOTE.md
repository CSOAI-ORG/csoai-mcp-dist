# MIGRATION_NOTE - MCP 2026-07-28 wire - class `full`

**Date:** 2026-10-07 - **Lane:** M4 MCP-migration (header-add wave) - **Branch:** `mcp-2026-wire-header-add`  
**Runbook:** `MCP_2026_WIRE_MIGRATION_PLAN_2026-10-07.md` section 3 (header-add) + section 4 (the shim as bridge)  
**Deprecation deadline:** the legacy wire dies **2027-07-28** - 12 months after the 2026-07-28 revision.

## 1. Transport reality

Server runs on stdio (raw newline-delimited JSON-RPC in `server.js` - **no MCP SDK at all**).

## 2. What changed in this branch

1. **No SDK pin exists to change**: `package.json` declares no `@modelcontextprotocol/sdk` / `mcp` dependency (`server.js` implements the handshake itself), and the runbook forbids inventing a manifest line that is not there.
2. `mcp2026_shim.py` vendored at the repo root as the reference ingress middleware.

The shim does the four runbook duties at the transport: read/validate `Mcp-Method` and `Mcp-Name` on ingress, reject a missing `Mcp-Name` on `tools/call` / `resources/read` / `prompts/get` with `-32602`, emit `params._meta.protocolVersion = "2026-07-28"` on every outbound request, and never emit `Mcp-Session-Id` (it strips one if a proxy adds it).

## 3. Verify

```bash
PYTHONPATH= /opt/homebrew/bin/python3.11 ~/clawd/mcp_wire_audit.py audit --local csoai-mcp-dist
```

| state | era | migration |
|---|---|---|
| before (default branch) | pre-2025-06 | full |
| **after (this branch)** | **pre-2025-06** | **full** |
| control (note block removed) | - | - |

Files changed in this branch: `mcp2026_shim.py`, `MIGRATION_NOTE.md`. The scanner reads the source/manifest files only: it skips `mcp2026_shim.py` by design (`SELF_FILES`) and does not scan `.md`, so neither `MIGRATION_NOTE.md` nor the shim contributes signals above.

**How to read the `after` row honestly.** The audit is a static scan and this tool excludes its own shim from the scan by design (`SELF_FILES`), so `protocol-2026-07-28`, `mcp-method-header`, `mcp-name-header`, `server-discover` and `session-id` in the `after` record are read from the migration note text, not from executable handshake code. The `session-id` signal in particular is prose (the note documents that the shim *strips* the header) - the control run, which deletes only that note block, drops back to `- / -` and shows no `session-id` at all. Runtime evidence for the wire is the `mcp>=2.0.0` pin (2.3.0 speaks 2026-07-28) plus the vendored shim at the ingress; `mcp>=2.0.0` alone is not a wire signal for this scanner.

## 4. Follow-ups (not in this branch)

* Census class for this repo is **`full`, not `header-add`** - `server.js` answers `protocolVersion: '2024-11-05'` to `initialize` (era `pre-2025-06`, signals `initialize-handshake`, `protocol-2024-11-05`). The header-add steps have nothing left to do here; **the audit is unchanged at `pre-2025-06 / full`**.
* Owed as separate work: the `full` runbook = header-add **+ handshake-removal** (drop the raw `initialize` branch, add `server/discover`, answer `2026-07-28`) - see plan §3.
* Note: this repo's default branch is `defoneos-sign-mcp` and its `package.json` describes `defoneos-sign-mcp`; the PR targets the default branch as-is.

Verify command of record: `PYTHONPATH= /opt/homebrew/bin/python3.11 ~/clawd/mcp_wire_audit.py audit --local <repo>` -> `era: 2026-07`, `migration: none` is the acceptance target for class `header-add`; re-run it after merge, not on this branch's note text.

Plan: `MCP_2026_WIRE_MIGRATION_PLAN_2026-10-07.md` - deadline 2027-07-28 - measurement, not certification.
