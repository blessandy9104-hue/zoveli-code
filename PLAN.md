# Zoveli Code — Integration Plan

**Status:** Proposed — no source has been imported or modified yet.
**Date:** 2026-11-07
**Owners:** Andy (`blessandy9104-hue`, `blessandy9104@gmail.com`)
**Upstream:** [`anomalyco/opencode`](https://github.com/anomalyco/opencode), branch `dev`, commit `ece56ca58419083275f2a2a81ac28d05f4a1b89b` (version 2.2.5)
**Product repo:** [`blessandy9104-hue/zoveli-ai`](https://github.com/blessandy9104-hue/zoveli-ai), branch `main`

---

## 1. Executive summary

Zoveli Code will be a Zoveli-branded, MIT-licensed derivative of OpenCode. It
keeps OpenCode's coding-agent runtime and adds a first-party Zoveli integration
layer so that verified math, local memory, vault data, and Zoveli's trust model
are available inside the agent.

The recommended architecture is a **companion fork**:

- This repository hosts the OpenCode derivative and a Zoveli plugin.
- The Zoveli AI monorepo remains the source of truth for `mathcore`, memory,
  vault, and local tools.
- Integration happens over Zoveli's existing **MCP** surface, not by copying
  Zoveli logic into the fork.

This keeps upstream merges practical, avoids a second copy of Zoveli's runtime,
and lets both projects ship independently.

**No code changes will be made until this plan is approved.**

---

## 2. Goals and non-goals

### Goals

1. Ship a local-first coding agent under the Zoveli name that can edit files,
   run commands, manage sessions, and work with local or cloud models.
2. Make Zoveli's verified-math and memory tools available to the agent through
   MCP.
3. Preserve Zoveli's trust contract: verified results are explicit, auditable,
   and never silently approximated.
4. Keep an upstream-tracking fork that can merge OpenCode changes with
   predictable, limited conflicts.
5. Produce reproducible Windows and macOS builds with clear attribution.
6. Default to local models and local tools; cloud providers are opt-in.
7. Guarantee **user sovereignty**: every shipped capability is available to the
   machine's owner, with no hidden gates, artificial quotas, or mandatory
   approvals. Restrictions are opt-in policies the owner chooses.

**Governing principle:** Zoveli Code is the user's personal possession. The
owner decides what to run, which providers to use, which MCP servers to expose,
which files to edit, and which commands to execute. The product must not stop,
nanny, or silently downgrade a capability. Safety features exist to protect the
owner's data from third parties and to make consequences visible — never to
overrule the owner. Any restriction we ship must be transparent, configurable,
and disabled by default unless the owner enables it.

### Non-goals (for the first release)

- Replacing the Zoveli AI desktop app or its assistant UI.
- Merging the OpenCode source tree into the `zoveli-ai` monorepo.
- Reimplementing `mathcore` or the memory store inside this repository.
- Hosting a public multi-tenant Zoveli Code service.
- Renaming every internal `@opencode/*` package or `OPENCODE_*` environment
  variable before the first release.
- Training on, collecting, or transmitting user data.

---

## 3. What exists today

### 3.1 OpenCode (upstream snapshot)

Relevant facts verified in the source snapshot:

| Area | Location | Notes |
| --- | --- | --- |
| Build tool | `package.json` | Bun `1.4.4`; `bun install`, `bun dev`, `bun run check`. |
| CLI entrypoint | `packages/cli/src/index.ts` | Commands: `run`, `serve`, `mcp`, `plugin`, `models`, `session`, `service`, `debug`, `upgrade`, `uninstall`. |
| TUI | `packages/tui/src` | Full-screen terminal client; product name appears in title and command IDs. |
| Plugin API | `packages/plugin/src/README.md` | `ctx.mcp.transform`, `ctx.tool.transform`, `ctx.tool.hook("execute.before"/"execute.after")`, `ctx.session.hook("context")`, `ctx.skill.transform`. |
| MCP | `packages/core/src/config` | Local servers use `{ type: "local", command: [...] }`; remote servers use `{ type: "remote", url }`. |
| Global paths | `packages/util/src/global.ts` | `const app = "opencode"`; XDG roots for data, cache, config, state. |
| Product strings | CLI/TUI/desktop | `"OpenCode"` in terminal title, help, desktop constants, docs links. |
| Packaging | `script/pack.ts` | `bun run pack` builds npm packages and archives. |
| License | `LICENSE` | MIT, Copyright (c) 2025 opencode. |

### 3.2 Zoveli AI (product monorepo)

Relevant facts verified in the source:

| Area | Location | Notes |
| --- | --- | --- |
| Runtime | `packages/host` | OpenAI-compatible and Anthropic chat loop; in-memory history (40-message cap); tool dispatch. |
| Math | `mathcore`, `packages/math-intelligence`, `packages/math-domains` | Verified computation and audit certificates. |
| Local MCP bridge | `scripts/zoveli-mcp-server.mjs` | Stdio JSON-RPC bridge. Reads `GET /api/tools`, calls `POST /api/run` on a loopback Zoveli server. Requires `ZOVELI_URL` (default `http://127.0.0.1:8080`). |
| Remote MCP | `scripts/remote-mcp-server.mjs`, `packages/host/src/remote-mcp` | Streamable HTTP `/mcp`; bearer auth; math-only allowlist by default. |
| Tool surface | `docs/guides/remote-mcp.md` | `mathcore_*`, `math_*`, memory, vault, sysinfo, code, web, scheduler, persona tools. |
| Defaults | `README.md` | Local-first, optional Ollama, optional cloud keys, no telemetry. |
| Tooling | `package.json` | Node `>=22.16`, pnpm `11.7.0`. |

The existing local MCP bridge is the key integration point: it already exposes
Zoveli's live tool catalog and dispatches tool calls to the local server. Zoveli
Code does not need to know how memory or `mathcore` work internally.

---

## 4. Architecture decision

### Option A — Companion fork over MCP (recommended)

```text
+---------------------------+
| Zoveli Code CLI / TUI     |
| (OpenCode derivative)     |
+-------------+-------------+
              |
              | plugin + MCP
              v
+---------------------------+
| @zoveli/code-plugin       |
| - registers MCP server    |
| - persona / instructions  |
| - verification hooks      |
| - local-first defaults    |
+-------------+-------------+
              |
              | stdio JSON-RPC
              v
+---------------------------+
| zoveli-mcp-server.mjs     |
| (existing Zoveli bridge)  |
+-------------+-------------+
              |
              | HTTP loopback
              v
+---------------------------+
| Zoveli local server       |
| mathcore / memory / vault |
+---------------------------+
```

**Pros:** clean ownership; upstream merges stay small; reuses Zoveli's tested
MCP surface; no duplicated math or storage logic; each project can release
independently.
**Cons:** requires a running Zoveli server for math/memory tools; two processes
to install and diagnose.

### Option B — Vendor OpenCode into `zoveli-ai`

**Pros:** one install, one monorepo.
**Cons:** two large toolchains (Bun + pnpm/Node); heavy merge burden; mixes
product boundaries; likely duplicates or replaces Zoveli's host loop.

### Option C — Upstream OpenCode plus an external plugin only

**Pros:** least code.
**Cons:** not a Zoveli-branded product; no control over TUI, defaults, paths, or
packaging; fails the "modify it to the Zoveli AI project" requirement.

### Decision

**Adopt Option A.** Build Zoveli Code as a fork with a small, well-defined
Zoveli integration plugin. Revisit Option B only if a single-binary install
becomes a hard product requirement.

---

## 5. Integration design

### 5.1 Zoveli plugin (`packages/plugins/zoveli` in this repo)

A first-party OpenCode plugin, modeled on `packages/plugins/figma`.

**Responsibilities:**

1. **Register the Zoveli MCP server** through `ctx.mcp.transform` when the user
   has not configured one:

   ```ts
   await ctx.mcp.transform((editor) => {
     if (editor.get("zoveli")) return
     editor.set("zoveli", {
       type: "local",
       command: ["node", "<resolved path>/scripts/zoveli-mcp-server.mjs"],
       environment: { ZOVELI_URL: "http://127.0.0.1:8080" },
     })
   })
   ```

2. **Inject the Zoveli system policy** through the session context hook:
   - Prefer `mathcore_*` / `math_*` tools for calculations.
   - Never present an unverified result as verified.
   - Keep vault and memory data local.
   - Ask before destructive shell commands or writes outside the workspace.
3. **Annotate verified math results** with `ctx.tool.hook("execute.after")`:
   - Detect `mathcore_*` and `math_*` tool results.
   - Attach a concise receipt: verified value, operation, audit identifier.
   - Preserve the original result; never rewrite the numeric answer.
4. **Add Zoveli skills/instructions** through `ctx.skill.transform` or an
   `AGENTS.md`/instructions file:
   - `zoveli-math`: when and how to call verified math.
   - `zoveli-memory`: local memory and vault usage.
   - `zoveli-trust`: how to describe verification to the user.
5. **Health and diagnostics:** detect a missing Zoveli server and show an
   actionable message (`Start Zoveli with pnpm start or launch the desktop app`).

**Open spike:** confirm whether `execute.after` can attach structured metadata
that the TUI renders as a badge, or whether the first release should append a
short text receipt to the tool output. Do this before committing to a UI.

### 5.2 MCP modes

| Mode | Server | Use case |
| --- | --- | --- |
| Local stdio | `scripts/zoveli-mcp-server.mjs` | Default. Full local tool set; requires a loopback Zoveli server. |
| Remote HTTP | `scripts/remote-mcp-server.mjs` (`/mcp`) | Optional. Math-only by default; bearer token; for hosted or remote math verification. |

Remote mode is opt-in. It must never become the default for vault or memory
data.

### 5.3 Verification and trust UX

The first release must make verification visible without adding noise:

- **Verified:** green/positive marker and a one-line receipt.
- **Partially verified:** yellow marker naming the missing cross-reference.
- **Unverified:** neutral marker; no implication of verification.
- **Audit:** a command or detail view that exposes the underlying record.

The exact TUI treatment depends on the spike in 5.1. Desktop/web rendering can
follow in a later phase.

### 5.4 Local-first defaults and owner authority

- **Default permission mode: `owner`.** All tools are enabled, including shell,
  file writes, network access, MCP servers, and subagents. There are no
  mandatory approval prompts and no capability gates. The owner can opt into
  `ask` or `deny` rules through configuration if they want restrictions.
- Default to Ollama and a model detected during install. Cloud providers are
  available whenever the owner configures a key; nothing is hidden or blocked.
- Updates: `notify` by default; automatic installation is the owner's choice.
- No telemetry. No analytics endpoints added by this fork.
- MCP bridge defaults to loopback for convenience. Remote/HTTP exposure is a
  supported capability: the owner can bind it, configure tokens, and use it.
  The existing Zoveli remote bridge keeps its own authentication rules; Zoveli
  Code does not add a second gate on top.
- Every restriction must be documented, visible in config, and reversible. The
  product never silently reduces the tool list, model list, provider list, or
  filesystem scope based on a policy the owner did not choose.

### 5.5 Branding and rename strategy

A full rename of every internal identifier would create permanent merge
conflicts. Use a **two-tier rename**:

**Tier 1 — user-facing, first release:**

- Product name: `Zoveli Code`.
- CLI binary: `zoveli.ky` (displayed as `Zoveli.ky` in docs and installers).
- TUI title, help text, splash, update text, docs links.
- Installer names, artifact names, release notes.
- Global app root: `zoveli-code` in `packages/util/src/global.ts`, so Zoveli
  Code does not share config/data with an installed OpenCode.
- Default theme: a Zoveli theme (colors and name).

**Tier 2 — internal, deferred:**

- Keep `@opencode/*` package names, `OPENCODE_*` environment variables, and
  internal command IDs in the first release.
- Add `ZOVELI_CODE_*` aliases and document both names.
- Rename packages only if upstream divergence becomes unmanageable.

**Rule:** every branding edit must be isolated in a small set of files, listed
in `docs/branding.md`, so upstream merges have a predictable conflict surface.

---

## 6. Repository and upstream workflow

### 6.1 Import strategy

Two viable import shapes:

1. **Preserve upstream history (recommended):** push the existing OpenCode
   commit graph as the initial history of this repository, add
   `upstream = https://github.com/anomalyco/opencode.git`, and develop on
   `main`. Future merges use `git merge upstream/dev`.
2. **Fresh snapshot:** single initial commit of the source tree. Smaller repo,
   but upstream merges require manual patches.

**Recommendation:** preserve history. The clone is already available at
`F:\opencode-dev`; it will be re-pointed at the new repository after approval.
The initial push is large (about 680 MB of packed Git history), but it is a
one-time cost and preserves attribution.

### 6.2 Branch model

```text
upstream/dev  (read-only)
      |
      v
main          (Zoveli Code product branch, default)
      |
      +-- feature branches (short names, hyphen-separated)
```

- Default branch: `main`.
- Feature branches: `zoveli-plugin`, `branding-pass`, `session-store`.
- Commits: conventional style (`feat(plugin): ...`, `chore(branding): ...`).
- Author: `Andy <blessandy9104@gmail.com>`.
- Merge cadence: weekly, or before each Zoveli Code release; never merge
  upstream directly into a feature branch without a baseline test run.

### 6.3 Upstream divergence budget

Keep the diff from upstream limited to:

- `packages/plugins/zoveli/**` (new code).
- Branding files (CLI/TUI/desktop strings, theme, global app root).
- Default config and documentation.
- A small number of test fixtures.

Anything larger requires a written justification in the PR description.

---

## 7. Phases, deliverables, and acceptance criteria

Estimates are rough and assume one developer.

### Phase 0 — Plan approval (0.5 day)

**Deliverables**

- This plan, README, NOTICE, LICENSE.
- Approval of the open decisions in section 11.

**Acceptance**

- Owner approves architecture, repo name, binary name, and first-release scope.

### Phase 1 — Import and baseline (1–2 days)

**Deliverables**

- OpenCode `dev` history imported into `zoveli-code`.
- `upstream` remote configured.
- `bun install` and `bun dev` succeed on Windows.
- Baseline `bun run check` results recorded (pre-existing failures listed).
- `docs/baseline.md` with versions, commands, and known gaps.

**Acceptance**

- The unmodified fork builds and starts.
- No source changes beyond repository metadata and attribution files.

### Phase 2 — Branding pass (2–3 days)

**Deliverables**

- `zoveli.ky` binary and CLI help text.
- TUI title/splash/update strings.
- Global app root `zoveli-code`; migration note for anyone with an existing
  OpenCode config.
- Zoveli theme and default config.
- `docs/branding.md` listing every changed string/file.
- Screenshots of the branded TUI.

**Acceptance**

- `zoveli.ky --help` and the TUI show Zoveli Code, not OpenCode.
- Config/data/log paths are under the new app root.
- Upstream functionality is unchanged; baseline tests still pass.

### Phase 3 — Zoveli bridge plugin (3–5 days)

**Deliverables**

- `packages/plugins/zoveli` with MCP registration, persona, and health checks.
- Local stdio MCP connection verified against a running Zoveli server.
- Math tool calls return real `mathcore` results and audit identifiers.
- Missing-server path shows a clear message.
- Unit and integration tests for the plugin and MCP bridge.
- `docs/zoveli-integration.md`.

**Acceptance**

- From Zoveli Code, `mathcore_compute` and at least one `math_*_solve` tool
  return verified results.
- A wrong or missing tool name fails safely.
- The default local flow keeps vault and memory data on the machine; remote
  exposure is an explicit owner-configurable option.

### Phase 4 — Trust UX and memory (4–6 days)

**Deliverables**

- Verification receipts in tool output or TUI metadata (depending on spike).
- Memory and vault tools available through the same MCP bridge by default in
  owner mode, with opt-in policy restrictions for owners who want them.
- Audit view or command.
- Offline-mode test: local model + math + memory with the network disabled.
- Documentation for verified/partially verified/unverified states.

**Acceptance**

- A verified math answer is visibly distinguishable from an unverified one.
- The audit record can be retrieved.
- Offline flow passes without cloud access.

### Phase 5 — Packaging and release (3–5 days)

**Deliverables**

- `bun run pack` artifacts and `zoveli.ky` binaries for Windows and macOS.
- Installer scripts aligned with Zoveli's sovereign-distribution approach.
- SHA-256 checksums and build provenance.
- CI: typecheck, lint, plugin tests, MCP integration test, Windows smoke test.
- Release checklist and rollback instructions.
- Version scheme: `<upstream-version>-zoveli.<n>` (for example
  `2.2.5-zoveli.1`).

**Acceptance**

- A clean machine can install, start, connect to Zoveli, and run a verified
  calculation.
- Artifacts are reproducible from a tagged commit.

**Total:** roughly 3–4 weeks of focused work, excluding upstream review cycles.

---

## 8. Security, privacy, and user sovereignty

Security features protect the owner's data and machine from third parties. They
do not limit what the owner may do with their own software.

- **Owner authority.** The default is full capability for the local user: all
  tools, all providers, all MCP servers, shell access, filesystem access, and
  network access. No mandatory approvals, no silent ceilings, no hidden feature
  flags. Restriction policies are opt-in and reversible.
- **Transparent defaults.** Loopback is the default bind for the local MCP
  bridge because it is the convenient local path, not a prohibition. The owner
  can bind remote, configure tokens, and expose the bridge; the existing Zoveli
  remote bridge keeps its own documented authentication rules.
- **No secrets in logs.** Reuse Zoveli's redaction rules; never print API keys,
  bearer tokens, vault contents, or prompts in diagnostics.
- **Math integrity.** Never rewrite a `mathcore` result. Verification metadata
  is additive.
- **Data locality.** Memory/vault tools stay on the local bridge by default.
  Cloud models receive only what the owner sends in the conversation.
- **Supply chain.** Pin dependencies, review upstream lockfile changes, and
  publish checksums for release artifacts.
- **License hygiene.** Keep `LICENSE`, `NOTICE`, and upstream attribution in
  every artifact and installer.

---

## 9. Testing and CI

### Local checks

From the affected package, not the repository root:

```bash
bun typecheck
bun run test
```

From the repository root:

```bash
bun run check
```

### Test layers

| Layer | What it proves |
| --- | --- |
| Unit | Plugin transforms, tool hooks, path resolution, config parsing. |
| MCP contract | `initialize`, `tools/list`, `tools/call` against the real bridge. |
| Integration | Zoveli server + mathcore + Zoveli Code end to end. |
| Offline | Local model, math, and memory with network disabled. |
| Platform smoke | Windows and macOS launch, TUI render, shell tool, file edit. |
| Upgrade | Existing config/data survives a version upgrade. |

### CI gates

- Typecheck and lint on every PR.
- Plugin and MCP tests on every PR.
- Windows smoke test on `main`.
- Release builds only from tagged commits with checksums.

---

## 10. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Upstream churn creates merge conflicts | High | Medium | Keep branding surface small; weekly merges; conflict list in `docs/branding.md`. |
| Two runtimes confuse users | Medium | High | One install path; health check; clear docs; optional auto-start later. |
| TUI cannot render verification badges | Medium | Medium | Spike first; fall back to text receipts; desktop/web badge later. |
| MCP bridge requires a running server | High | Medium | Detect and guide; add auto-start in a later phase. |
| Owner-mode tools create risk the owner did not intend | Medium | High | Full authority stays the default per the product principle; optional opt-in policies; clear audit trail; no hidden gates. |
| License or trademark mistake | Low | High | Keep NOTICE and MIT text; do not use OpenCode marks as the product name; legal review before public release. |
| Large initial push fails | Low | Medium | Push in stages or use a shallow snapshot if the network rejects it. |
| Windows path/signing issues | Medium | Medium | Test early; follow Zoveli's unsigned-install guide; document SmartScreen. |

---

## 11. Resolved decisions (2026-11-07)

1. **Binary name:** `zoveli.ky` (displayed as `Zoveli.ky`).
2. **Runtime dependency:** companion fork; v0.1 requires an installed/running
   Zoveli AI server for math, memory, and vault tools.
3. **Internal names:** keep `@opencode/*` packages and `OPENCODE_*` environment
   variables in v0.1. Add `ZOVELI_CODE_*` aliases later.
4. **Default model:** detect the best available local model at install time.
5. **First public release:** after Phase 5 (packaged installers).
6. **User sovereignty:** full owner authority by default. All capabilities are
   available; restrictions are opt-in policies only. No hidden limits, no
   mandatory approvals, no capability gates.

---

## 12. Approval record

- [x] Architecture: companion fork over MCP (Option A) approved.
- [x] Repository `blessandy9104-hue/zoveli-code` approved as public.
- [x] Binary/product name approved: `zoveli.ky` / Zoveli Code.
- [x] First-release scope approved: Phase 5.
- [x] Decisions in section 11 resolved.
- [x] Permission granted to import the OpenCode history and begin Phase 1.
- [x] User-sovereignty principle adopted as a governing product rule.

Approved by Andy on 2026-11-07. Phase 1 begins with the source import and
baseline build. No OpenCode source had been modified before this approval.
