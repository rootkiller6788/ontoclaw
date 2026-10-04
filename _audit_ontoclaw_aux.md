# Code-Verified Audit — OntoLoop (D:\Opencode\ontuinx)

Method: read source directly; LOC via `find`, test counts via `grep -cE '^\s*#\[(tokio::)?test\]'`; call-site wiring via `grep -rn`. Repo-wide: 102,076 LOC in `src/`, 11,935 LOC in `tests/` (85 integration files).

> **Verification limit:** `cargo test` could not run on this host. It failed compiling the third-party `hex` 0.4.3 dependency, before any project source compiled: `E0463 can't find crate for std`, `failed to mmap ... libcore-*.rmeta`, `os error 1455 (pagefile too small)`. This is a machine virtual-memory/pagefile limit, not a workspace defect. **Verified:** source reads, LOC, static test counts, module structure, public types, call-site wiring, dead-code/stub detection. **Not verified:** that anything compiles or that any test passes. No test-pass claim below is empirical.

---

## 1. Area 2 remainder — GitMemory verdict (sentence that was cut)

**Verdict — competing graph fact store? No, it is a semantic/wiki view + governance ledger, not the authoritative fact store.** It *represents* a graph (`SemanticEdge{from,to,relation,confidence}`, `src/gitmemory_core/semantic_edges.rs:1-8`) and exports node/edge artifacts (`graph_export.rs`), but that graph is derived from markdown projections and lives in files under `.gitmemory/`. Crucially, `ViewPlane` reads the **shared** `graph:{session}:snapshot` produced by the RAG path (`src/gitmemory_core/view_plane.rs:60`) rather than owning it, and it writes no authoritative entity/relationship records to StateStore. Absorbable as a semantic/memory view layer; do not absorb it as "the" graph store.

**Secondary caveat:** the "seven frozen core packages" framing is cosmetic. `repo_core.rs`, `compiler_core.rs`, `context_core.rs` are 14-line manifest shells; `gateway_core.rs` and `patch_core.rs` are likewise `facade_only:true` (`protocol.rs:47`). `GitmemoryCoreKernel::has_exactly_seven_packages` (`mod.rs:96`) asserts a property its own constructor sets (`mod.rs:90` "core package skeleton ready"). The real functionality lives in separate files (`incremental_compiler.rs`, `canonical_repo.rs`, `recall_core.rs`, …), not behind those `CorePackage` structs.

**What it stores:** file-backed `.gitmemory/` control root — `projections/{graph,vector,search}`, `semantic/edge_cache.json`, `checkpoint.jsonl`, `dependency_graph.json`, `hot_index.json`, `schema_registry.json` (`canonical_repo.rs:21`, `incremental_compiler.rs:109-115`, `graph_export.rs:146,166`, `hot_index_updater.rs:67`) — plus StateStore KV prefixes `memory:episode:*`, `memory:recall:route:*`, `memory:patch:review:*`, `memory:compiler:run:*`, `memory:provenance:*`, `memory:ontology:*`, `memory:conflict:*`, `memory:formal-check:*`, `plugin-capability-admission:*`, `capability-admission:*`.

**What it queries:** `RecallCore::plan_with_graph_expansion` (`recall_core.rs`), `RecallPluginRouter::route` (`recall_plugin_router.rs:29`, lexical fallback scans StateStore), `GraphExportService::export_offline` (`graph_export.rs:61`, loads edges from `edge_cache.json`), `ViewPlane::build` (`view_plane.rs:52`).

Totals: 31 files / 7645 LOC / 45 unit tests + `tests/pwiki11_*` (12 files / 13 tests). Public path `plugins::gitmemory_core` via `#[path]` re-export at `src/plugins/mod.rs:3-4`.

---

## 2. Area 3 — Skills / plugins / capability admission / foundry

### Module table

| Path | LOC | Tests | Real/Stub | Purpose |
|---|---|---|---|---|
| `src/skills/mod.rs` | ~? | 1 | Real | `SkillRegistry` register/install/enable/disable; requires `LearningSignal.evidence_ref` |
| `src/skills/foundry/extractor.rs` | 219+ | 1 | Real | First-principles extraction from intake |
| `src/skills/foundry/router.rs` | ~230 | 3 | Real | Route to S1/S2/S3 layer, promotion suggestion |
| `src/skills/foundry/builder.rs` | ~200 | 1 | Real (placeholder MCP schema at `:70`) | Skill skeleton + manifest render |
| `src/skills/foundry/validator.rs` | ~200 | 1 | Real | Frontmatter/trigger/CLI-JSON/S2-smoke/S3-MCP contract checks |
| `src/skills/foundry/packager.rs` | ~100 | 0 | Real | Package / install / enable / disable |
| `src/skills/foundry/feedback.rs` | ~450 | 2 | Real | Promotion gate, feedback classification |
| `src/skills/foundry/intake.rs` | ~10 | 0 | Real | Intake normalization |
| `src/plugins/lifecycle.rs` | 1155+ | 6 | Real | Signed install, verify, rollout, capability admission |
| `src/plugins/host.rs` | ~410 | 2 | Real | Subprocess plugin loader/invoker |
| `src/plugins/facade_guard.rs` | ~210 | 2 | Real | `facade_only` + payload guard |
| `src/plugins/registry.rs` | ~95 | 0 | Real | Plugin registration |
| `src/plugins/resolver.rs` | ~? | 0 | Real | Resolve by id/capability/kind/scopes |
| `src/plugins/signature.rs` | ~? | 0 | Real | HMAC signature material + verify |
| `src/plugins/compatibility_loader.rs` | ~? | 2 | Real | Compatibility discovery |
| `src/plugins/markdown_preprocess.rs` | ~? | 3 | Real | Markdown preprocessing plugin |

Totals: skills 9 files / 1969 LOC / 9 unit tests; plugins 9 files / 3277 LOC / 15 unit tests; integration `tests/pq11_skill_foundry_*` (6 files / 7 tests), `tests/pq7_plugin_lifecycle_signed_e2e.rs` (1 test). No `todo!()`/`unimplemented!()` in any of these.

### KEY VERDICT — can skill/plugin invocation reach resources WITHOUT a single admission gate?

**Yes. There is no single chokepoint; invocation is not gated the way load/install is.**

- Plugin **install / host_load** ARE gated: signature verify + `enforce_plugin_capability_admission` (`src/plugins/lifecycle.rs:557-572`, gate body `:752`). Unsigned install is rejected (`lifecycle.rs:1347` test).
- Plugin **invocation** is NOT: `PluginLifecycleManager::host_invoke` (`lifecycle.rs:656`) calls `self.host.invoke_by_id` directly with only `enforce_manifest_contract` — no capability admission, no ACL, no signature re-check.
- `PluginHost::new` / `invoke` / `invoke_by_id` are **public** (`src/plugins/host.rs:151`, `:206`, `:215`); a holder can spawn the subprocess runtime directly (`host.rs:85`), bypassing lifecycle entirely. Only guard at invoke time is facade/payload checks (`host.rs:242`, `facade_guard.rs:37,90`).
- Service layer: `Mediator::dispatch` routes all domains (`src/services/mediator.rs:223`), but `enforce_gate_token` covers only Provider/Tool/Memory/Relation — **Plugin and SkillFoundry are not token-gated** (`src/contracts/services.rs:39`). The token itself is a plaintext prefix check `token.starts_with("svc-gate:{session}:{scope}:")` with no secret/HMAC (`services.rs:62-83`), so it is forgeable.
- Skills: `SkillRegistry::register` requires a `LearningSignal` with non-empty `evidence_ref` (`src/skills/mod.rs:52-66`), but skills are markdown/prompt artifacts, not executable capabilities; foundry promotion gate at `src/skills/foundry/feedback.rs:109`.
- The strongest genuine gate is for **tools**: capability-intent selector → `CapabilityAdmissionEngine` → `enforce_admission_governance` (ACL + formal checker + conflict manager) at `src/orchestration/mod.rs:1437-1450`.

Net: resources are *mostly* mediated, but "single admission point" overstates it — plugin invoke and skill-foundry sit outside the token gate, and the gate token is not cryptographic.

---

## 3. Area 4 — Production gates

Files: `docs/production_contract.md` (91 lines), `docs/ACCEPTANCE_LAYERS_V1.md` (134), `docs/CONTRACT_MIGRATION_MATRIX_V1.md` (54).

**What the docs specify:** frozen sovereignty chain `constraint_shield → review_gate → wal → harness`; WalTx 5-tuple `state + event_log + evidence_ref + relation + write_proof`; tri-state admission `Allow|RequiresApproval|Blocked`; artifact proof triad `write_proof + hash + evidence_ref`; NoBypass `static + compile + runtime`; canary → auto-rollback. Acceptance layers L0–L4 map to `deploy/scripts/week6_acceptance.*` and `release_gate_report.*`.

**Does enforcing code exist? Mixed:**

| Gate | Code exists | Enforced in runtime? |
|---|---|---|
| Chain **order** | `src/contracts/governance_precedence.rs` (`SovereigntyStage`, `validate_stage_chain`) | **No** — zero callers outside its own unit tests |
| **Constraint Shield** | `src/security/constraint_shield.rs` (177 LOC, real classifier + persisted decision log at `:65`) | **No** — zero callers anywhere in `src/`, `bin/`, or `tests/` (dead code) |
| **ReviewGate** | No `ReviewGate` type. Nearest real: `PatchReviewQueue` (`gitmemory_core/patch_review_queue.rs`) + `SecurityPolicy::review_requirement` (`src/security/mod.rs:236`) | Partial, under different names |
| **WAL** | Real for relations: `RelationFacade::upsert_contract/upsert_edge/append_event` (`src/services/relation_facade.rs:16,104,196`) writes state+event+evidence+write_proof(+hot-index) bundle; proof carries `state_hash`/`replay_fp`; verified by `tests/waltx_production_write_minimal_e2e.rs`. Generic `WalRecord`/`WalStore` (`contracts/storage.rs:182,308`) and `WalTxEnvelope` (`contracts/version_a.rs:73`) have **no implementation anywhere** | Partial; multi-key bundle with fallback, not a real transaction |
| **Harness** | `src/contracts/code_harness.rs` + routing gate `src/lib.rs:507-560` (rejects code tasks not on `harness_facade`, emits evidence) | **Yes, real** |
| **write_proof** | `src/contracts/artifact_delivery.rs:82,149,201` + `src/agent/mod.rs:868,1425,1522` | **Yes, real** for artifact-declaring tasks |
| **evidence_ref** | Pervasive (`memory` LearningSignal guard `src/memory/mod.rs:1023-1036`; artifact delivery; evidence ledgers) | **Yes, real** |
| NoBypass | static scan = unit tests (`tests/no_bypass_static_scan.rs`, `relation_facade.rs:381-445`); **compile gate = `build.rs`** (scans direct `.state_store.`/`.providers.`/`.tools.`/`McpManager` outside an allow-list, panics on violation); runtime = facade events + forgeable tokens | Real, but string-based with large allow-lists |

**Plainly: enforcement is partially real, partially aspirational.** Genuinely enforced: Harness routing, artifact `write_proof`, relation-level WAL proof, `evidence_ref` checks, tool admission/governance, and the `build.rs` compile-time no-bypass scan. Aspirational/dead: Constraint Shield (implemented, never called), the sovereignty *ordering* validator (never called), and the generic `WalStore` contract (trait with no impl). The NoBypass runtime token is a forgeable format string.

---

## 4. Area 5 — NoumenonCore

Crate `trustkernel` v0.1.0, Apache-2.0, edition 2024. Deps: `anyhow, sha2, hmac, hex` only. Features: `core` (default) + `ext-*`. Declares `[package.metadata.release_gates]` with a 3-config test matrix and 8 named critical tests.

**Size / tests:** 20 src files / **4679 LOC** + `tests/invariants.rs` 384 LOC. **55 tests** (47 unit + 8 integration). Static counts; not run (see verification limit).

**Real vs stub:** Core is real. `StaticAttestationVerifier` / `StaticTenantAuthority` / `StaticSupplyChainVerifier` (`src/trust.rs:82-150`) are placeholder-grade defaults meant to be replaced by injection; `EchoCapability` and `InMemory*` ledgers are test doubles. Production implementations are injected by the host.

**Features actually implemented:**
- **Kernel / TCB boundary:** 14-dependency-injected `NoumenonCore` (`src/kernel.rs:26-79`) with an execution state machine that rejects non-mandatory modes and orders identity → supply-chain → rollout → attestation (hard trust) → per-step tenant + hardening → resource reservation → budget reservation before executing (`:79-160`).
- **Signed artifact admission:** cosign-based `SigstoreAdmissionVerifier` with injectable `CommandRunner`, policy-pin and Rekor inclusion checks (`src/admission_sigstore.rs:80-180`). Requires `cosign` on PATH. Crypto elsewhere is HMAC-SHA256, not asymmetric PKI (`src/trust.rs:36-72`).
- **Provenance:** evidence hash-chain ledger with `prev_hash`/`record_hash` (`src/kernel.rs:184+`), lineage types (`provenance_core` in gitmemory is separate); supply-chain envelope (executor/verifier/policy/capability/provenance digests, signer OIDC identity, Rekor refs) in `src/ir.rs`.
- **execution_fingerprint:** `ExecutionRecord` carries `execution_fingerprint` + `FingerprintMaterial` (`src/ir.rs:188`); computed at `src/kernel.rs:252-273`; replay recomputation and mismatch detection at `:308-320`; order-invariance and tamper-rejection covered by tests.
- **TCB minimum check:** `tcb_meets_minimum` (`src/runtime/trust_bridge.rs:657`).
- Remote attestation (HTTP) with TTL + replay-challenge cache: `trust_bridge.rs:80`, `:860`.

**Integrated, not shelf-ware:** consumed by the main crate via `trustkernel` in `src/runtime/attestation_verifier.rs` (568 LOC), `src/runtime/trust_bridge.rs` (1663 LOC), `src/runtime/mod.rs:29`.

**Judgement: absorb as a crate.** It is self-contained (4 tiny deps), DI-based, feature-flagged, has release-gate metadata, and is already wired into the host binary. Caveats to carry over: HMAC not PKI; default verifiers are static placeholders; sigstore path needs `cosign`.

---

## 5. Area 6 — Benchmark & metrics

**Is it a real runnable suite?** The scaffolding is real and wired to the app, but evaluation is weak and the doc is inaccurate.

- **Doc claims:** 240 tasks, 8 categories, splits dev 120 / heldout **80** / stress **40**.
- **Actual data:** master 240, dev 120, heldout **64**, stress **56** (`deploy/benchmarks/benchmark_v1_manifest.json` + `grep -o '"task_id"'` per file). The doc's heldout/stress split is **wrong**.
- **Where data lives:** `deploy/benchmarks/` — `benchmark_v1_master.json`, `benchmark_v1.json`, `benchmark_v1_dev.json`, `benchmark_v1_heldout.json`, `benchmark_v1_stress.json`, `benchmark_v1_manifest.json`, plus `d12_real_tasks_v1.json` (52 cases). The doc links point at external `D:/AutoLoop/autoloop-app/...` even though local copies exist (stale).
- **Categories (8, ×30 each):** `frontend_replica`, `backend_feature`, `bug_fix`, `test_repair`, `refactor`, `deploy_script`, `multi_tool_orchestration`, `permission_reject`.
- **Scripts:** `benchmark_v1_generate.{ps1,sh}`, `benchmark_v1_eval.{ps1,sh}`, `benchmark_v1_compare.{ps1,sh}`. Eval validates dataset schema (8 required fields, ≥8 categories) then runs the app CLI `system benchmark run --artifact-path <dataset> --limit <n> --output <json>`, computing success rate + p50/p95. Rust runner: `run_real_task_benchmark` (`src/command_dispatch.rs:1127`), dispatched at `command_dispatch.rs:1440-1448`.
- **Weakness:** runner "success" = process returned `Ok` and runtime decision ≠ `"reject"` (`command_dispatch.rs:1208-1233`). It never checks the task's `auto_verifier`, `target_artifact_path`, sha256, or `write_proof`; the eval script only checks that the report file exists. `auto_verifier: "artifact_exists_nonempty_sha256_evidence_relation"` is descriptive metadata, never executed. Runner default dataset is `d12_real_tasks_v1.json` (52 prompt-only cases with no verifier/artifact fields), not `benchmark_v1_*`.
- Tasks are generated from per-category prompt templates with scenario substitutions (`bv1-fe-001` "billing dashboard", `bv1-fe-002` "crypto exchange landing", identical verifier strings), so "real-task benchmark" is overstated — templated synthetic prompts.

**Verdict:** runnable harness, real data files, exit-code-based scoring, no independent artifact verification.

---

## 6. Overall verdict — §4.1–§4.10 reuse map

| Plan section | Subject | Verdict |
|---|---|---|
| §4.1 | Production gates / acceptance docs | **Mixed.** Reusable: Harness routing (`lib.rs:507-560`), artifact `write_proof` (`contracts/artifact_delivery.rs`), relation WAL proof (`services/relation_facade.rs`), `build.rs` compile no-bypass scan, tool admission governance. Overstated: Constraint Shield and sovereignty ordering are dead code; `WalStore` is an unimplemented trait; NoBypass runtime token is forgeable. |
| §4.2 | NoumenonCore | **Genuinely reusable as a crate.** Self-contained, DI-based, integrated, 55 tests. |
| §4.6 | Memory / SuperMemory / GraphRAG | **Partially reusable.** Real: pipeline, write reconciliation, scope contracts, HNSW sidecar, StateStore persistence. Overstated: "semantic/vector" retrieval is lexical token overlap; stored embeddings unused in the main search path; GraphRAG queries a fresh single-document graph per iteration. |
| §4.7 | GitMemory / semantic lint / recall / graph export | **Reusable as a semantic/wiki view + governance ledger, not as a graph fact store.** Real compiler/lint/view-plane; `/seven core packages` is cosmetic; do not absorb as authoritative fact store. |
| §4.8 | Skills / plugins / admission / foundry | **Reusable with caveats.** Real lifecycle, signed install, capability admission on load, foundry pipeline. Overstated: no single admission chokepoint — invocation bypasses it; gate tokens are forgeable and don't cover plugin/skill domains. |
| §4.9 / §4.10 | Benchmark & metrics | **Runnable but overstated.** Real data + scripts + Rust runner; scoring is exit-code-based with no artifact verification; doc split counts wrong (64/56 vs 80/40); tasks templated. |

**Highest-confidence absorbables:** NoumenonCore/trustkernel; `src/memory/sidecar.rs` (real HNSW + quantized cold tier); the `gitmemory_core` compiler / semantic-lint / view-plane layer; and the harness + artifact-`write_proof` + tool-admission gates.

**Most overstated:** the "semantic/vector SuperMemory + GraphRAG" retrieval story (§4.6); the "single admission gate" for skills/plugins (§4.8); the "frozen non-negotiable" production gate chain whose order validator and Constraint Shield are unwired (§4.1); and the "real-task benchmark" claim (§4.9/4.10).
