# P1 MVP: see and control. Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A server and agent that import standard Kubernetes and flotilla clusters, including clusters Rancher manages, and give a team fast cross-cluster search, logs, exec, sign-in, RBAC, projects, audit and a migration from Rancher.

**Architecture:** One `admiral-server` binary with SQLite or Postgres and the UI embedded. A small `admiral-agent` per cluster dials out over mTLS gRPC, streams index deltas and proxies user requests with impersonation. The UI, the CLI and every other client use one public REST API with SSE streams.

**Tech Stack:** Rust (edition 2024, tokio, axum, tonic, rustls with ring, kube and k8s-openapi, sqlx, argon2, webauthn-rs, totp-rs, openidconnect, utoipa, clap, tracing, opentelemetry). Web: TypeScript, SolidJS, Kobalte, Tailwind, CodeMirror 6, xterm.js, Vite, pnpm, Vitest, Playwright.

**Spec:** [docs/design.md](../design.md). Read it with this plan; section numbers below refer to it.

## Global Constraints

- Every task is one issue (#3 to #24) in milestone "P1: MVP, see and control" and one pull request with labels, milestone and assignee.
- Tests are written first. Unit tests use no network and no real cluster. No bare sleeps; use `tokio::time::pause` and advance.
- Every Rust test passes on Linux, macOS and Windows. End-to-end jobs run on Linux only.
- Rust toolchain: the latest stable on the day task 1 lands, pinned in `rust-toolchain.toml`. MSRV equals the pinned version.
- No OpenSSL anywhere in the dependency tree; `deny.toml` bans `openssl` and `openssl-sys`.
- Library crates never print. Only the three binaries write to stdout.
- Credentials, tokens, kubeconfig contents and Secret data never appear in logs, errors, fixtures or the index.
- Text from clusters is escaped before it reaches the browser or a terminal.
- The UI calls only `/api/v1` and `/k8s/clusters/<id>`.
- Kubernetes support: the versions Rancher 2.16 supports at the time task 4 lands, plus flotilla.
- Labels admiral writes on cluster objects: `app.kubernetes.io/managed-by=admiral`. Projects use Rancher's `field.cattle.io/projectId`.
- Problem details use `type` values of the form `urn:admiral:problem:<slug>`.
- Defaults: session idle timeout 30 minutes, absolute 12 hours; kube tokens live 15 minutes; agent certificates live 24 hours and renew at two thirds of their life; heartbeat every 10 seconds, disconnected after 30; reconnect backoff from 1 second doubling to 60 with full jitter; page size 100, maximum 1000.
- Every new crate dependency comes in with its task; every new manifest adds its ecosystem to `.github/dependabot.yml`; every new required CI job is added to the `main` ruleset in the same pull request.
- Merge with `gh pr merge <n> --squash --subject "<title> (#<n>)" --body ""`.

## Review Focus

1. **The agent reconnects after the server restarts in the middle of a delta batch, or the cluster answers 410 Gone.** Expected: no missing and no duplicate rows afterwards. Test `reconnect_mid_batch_leaves_index_exact` in task 3 and `watch_relists_on_410_gone` in task 4.
2. **A cluster floods watch events (a rollout of 10,000 pods).** Expected: agent memory stays bounded and the server falls behind gracefully instead of crashing. Test `watch_flood_stays_within_channel_bound` in task 4.
3. **A user is removed from a group at the identity provider.** Expected: access goes away at the next group refresh, without waiting for the user to sign in again. Test `group_removed_at_provider_revokes_on_next_refresh` in task 8.
4. **Rancher data that is unusual but valid:** external principals with no local password, role templates that inherit from others, names at the 63-character limit, Unicode display names. Expected: all of it imports or is reported in the plan with a reason. Tests `external_principals_map_to_provider_identities`, `inherited_role_templates_import_in_dependency_order` and `long_and_unicode_names_preserved` in task 19.
5. **Clocks differ by a few minutes between agent and server.** Expected: certificates and tokens still validate. Test `issued_cert_tolerates_five_minutes_of_skew` in task 3.

---

## File structure

```
Cargo.toml                      workspace
rust-toolchain.toml  deny.toml  rustfmt.toml
crates/
  admiral-core/                 ids, Cluster, OwnershipMode, IndexRow, StatusSummary, errors
  admiral-store/                Store, migrations/{sqlite,postgres}/, one module per domain
  admiral-tunnel/               proto/tunnel.proto, ca.rs, server.rs, client.rs, proxy.rs
  admiral-driver/               lib.rs (ClusterDriver), kubernetes/, status.rs
  admiral-index/                query/ (lexer, parser), index.rs, subscribe.rs
  admiral-auth/                 password.rs, session.rs, token.rs, passkey.rs, totp.rs, oidc.rs, github.rs
  admiral-rbac/                 template.rs, binding.rs, render.rs, project.rs, effective.rs
  admiral-audit/                event.rs, chain.rs, export/{otlp,syslog,webhook}.rs
  admiral-compat/               kube_proxy.rs (/k8s/clusters), rancher ids and mapping
  admiral-migrate/              rancher/{client,plan,apply,rollback,detach}.rs, tests/fixtures/
  admiral-api/                  router.rs, routes/*, sse.rs, problem.rs, openapi.rs
  admiral-server/               main.rs, config.rs, embed.rs
  admiral-agent/                main.rs, config.rs
  admiral-cli/                  main.rs, commands/*
  admiral-bench/                scenario runner and report
web/                            SolidJS app, tests/, tests/fake-api/
charts/admiral-server/  charts/admiral-agent/
bench/scenarios/  bench/results/
docs/install.md  docs/benchmarks.md
```

## Order

Tasks 1 and 2 first. Then 3 and 4 together (the tunnel and the driver share `admiral-core` types only), then 5, 6 and 7. Tasks 8 to 14 depend on 6 and 7. The UI tasks 15 to 17 can start after 6 against the fake API. Task 18 follows 13. Task 19 follows 9 and 10. Tasks 20 to 22 come last.

---

### Task 1: Cargo workspace, core types, CI and release skeleton (#3)

**Files:**
- Create: `Cargo.toml`, `rust-toolchain.toml`, `deny.toml`, `rustfmt.toml`, `crates/*/Cargo.toml`, `crates/*/src/lib.rs` or `main.rs`
- Create: `crates/admiral-core/src/{ids.rs,cluster.rs,index_row.rs,status.rs,error.rs}`
- Modify: `.github/workflows/ci.yml` (add `deny` job), `.github/dependabot.yml` (add `cargo`)
- Create: `.github/workflows/release.yml` (builds on tag `v*`, publishes nothing until task 21)
- Test: `crates/admiral-core/tests/ids.rs`

**Interfaces:**
- Produces:
  - `ClusterId(String)`: `c-` plus 5 lowercase alphanumerics for new clusters; any Rancher id (`c-xxxxx`, `c-m-xxxxxxxx`, `local`) is accepted as is. `ClusterId::new_random() -> ClusterId`, `ClusterId::parse(&str) -> Result<ClusterId, IdError>`.
  - `ProjectId { cluster: Option<ClusterId>, short: String }` with `Display` as `<cluster>:<short>` or `<short>`; short form `p-` plus 5 lowercase alphanumerics.
  - `enum OwnershipMode { Observe, Shared, Owned }` (serde lowercase).
  - `enum DriverKind { Kubernetes, Flotilla }`.
  - `struct Cluster { id, name, driver: DriverKind, mode: OwnershipMode, created_at, last_seen: Option<OffsetDateTime> }`.
  - `struct IndexRow { cluster, group, version, kind, namespace: Option<String>, name, uid, labels: BTreeMap<String, String>, owners: Vec<OwnerRef>, status: StatusSummary, resource_version: String, updated_at }`.
  - `enum StatusSummary { Ready, Progressing, Degraded { reason: String }, Failed { reason: String }, Unknown }`.
  - `enum WatchEvent { Upsert(IndexRow), Delete(ObjectRef), Relisted { ty: ResourceTypeKey, rows: Vec<IndexRow> }, Bookmark { ty: ResourceTypeKey, version: String } }`, with `ObjectRef` and `ResourceTypeKey { group, version, kind }`.

- [ ] **Step 1: Write failing tests** `cluster_id_cases` (table: `c-ab12x` ok, `c-m-1a2b3c4d` ok, `local` ok, empty rejected, uppercase rejected, 64 chars rejected), `project_id_display_roundtrip` (`c-ab12x:p-q9w8e` parses and prints the same), `ownership_mode_serde_lowercase`.
- [ ] **Step 2: Run** `cargo test -p admiral-core`. Expected: FAIL, types missing.
- [ ] **Step 3: Implement** the types above in `admiral-core`, with empty crates for the rest of the workspace so the layout exists.
- [ ] **Step 4: Run** `cargo test --workspace && cargo clippy --workspace --all-targets -- -D warnings && cargo deny check`. Expected: PASS.
- [ ] **Step 5: Add** the `deny` job to CI and to the ruleset's required checks; add `cargo` to Dependabot with a weekly minor and patch group; update AGENTS.md commands and layout.
- [ ] **Step 6: Commit** `feat(core): add workspace, core types and CI for Rust`.

### Task 2: Store over SQLite and Postgres (#4)

**Files:**
- Create: `crates/admiral-store/src/{lib.rs,backend.rs,migrate.rs,backup.rs,leader.rs,clusters.rs}`, `crates/admiral-store/migrations/{sqlite,postgres}/0001_init.sql`
- Test: `crates/admiral-store/tests/{migrate.rs,backup.rs,leader.rs,clusters.rs}`, `crates/admiral-store/tests/common/mod.rs`

**Interfaces:**
- Consumes: `Cluster`, `ClusterId` from task 1.
- Produces:
  - `enum Backend { Sqlite(PathBuf), Postgres(String) }`
  - `Store::open(Backend) -> Result<Store, StoreError>`, `Store::migrate(&self) -> Result<MigrationReport, StoreError>`
  - `Store::backup(&self, to: &Path) -> Result<BackupInfo, StoreError>`, `Store::restore(Backend, from: &Path) -> Result<(), StoreError>`
  - `Store::try_lead(&self, name: &str) -> Result<Option<LeaderGuard>, StoreError>` (Postgres advisory lock; SQLite takes a file lock and is always the leader in its process)
  - `store.clusters()` with `insert`, `get`, `list`, `set_mode`, `touch_last_seen`, `delete`
  - Test helper `for_each_backend(|store| async { ... })`: always SQLite in a temp dir, plus Postgres when `ADMIRAL_TEST_POSTGRES` is set.

- [ ] **Step 1: Write failing tests** `migrate_from_empty_is_idempotent`, `backup_then_restore_roundtrips_rows`, `only_one_leader_at_a_time`, `leader_released_on_drop`, `clusters_crud`, all through `for_each_backend`.
- [ ] **Step 2: Run** `cargo test -p admiral-store`. Expected: FAIL.
- [ ] **Step 3: Implement** with sqlx (`sqlite`, `postgres`, `runtime-tokio`, `tls-rustls-ring`). Backups: `VACUUM INTO` for SQLite; for Postgres a logical dump of admiral's tables to a single compressed file written by admiral, so no `pg_dump` binary is needed.
- [ ] **Step 4: Run** `ADMIRAL_TEST_POSTGRES=... cargo test -p admiral-store` and without the variable. Expected: PASS both. Add a Postgres service container to the Linux test job.
- [ ] **Step 5: Commit** `feat(store): add SQLite and Postgres store with backup and leader lock`.

### Task 3: Tunnel protocol and agent registration (#5)

**Files:**
- Create: `crates/admiral-tunnel/proto/tunnel.proto`, `crates/admiral-tunnel/build.rs`, `crates/admiral-tunnel/src/{ca.rs,token.rs,server.rs,client.rs,proxy.rs,backoff.rs}`, `crates/admiral-store/src/agents.rs` plus migration `0002_agents.sql`
- Test: `crates/admiral-tunnel/tests/{registration.rs,reconnect.rs,streams.rs,certs.rs}`

**Interfaces:**
- Consumes: `ClusterId`, `IndexRow`, `Store`.
- Produces:
  - `Ca::generate(common_name: &str) -> Result<Ca, CaError>`, `Ca::issue_agent_cert(&self, cluster: &ClusterId, csr_der: &[u8], ttl: Duration) -> Result<IssuedCert, CaError>`; `not_before` is backdated 5 minutes.
  - `RegistrationToken`: 32 random bytes, base64url, stored as SHA-256, single use, expires after 1 hour.
  - `trait TunnelHandler { async fn on_deltas(&self, cluster: &ClusterId, batch: DeltaBatch) -> Result<Ack, TunnelError>; async fn on_disconnect(&self, cluster: &ClusterId); }`
  - `TunnelServer::serve(listener, ca: Ca, store: Store, handler: Arc<dyn TunnelHandler>)`
  - `TunnelServer::proxy(&self, cluster: &ClusterId, req: http::Request<Body>) -> Result<http::Response<Body>, TunnelError>`, each call on its own HTTP/2 stream.
  - `AgentClient::register(endpoint, token) -> Result<AgentIdentity, TunnelError>`, `AgentClient::connect(endpoint, identity) -> AgentConn`, `AgentConn::send(batch) -> Ack`, `AgentConn::commands() -> impl Stream<Item = ServerCommand>`
  - `DeltaBatch { rows: Vec<WatchEvent>, last_versions: BTreeMap<ResourceTypeKey, String> }`, using `WatchEvent` from task 1.

- [ ] **Step 1: Write failing tests** `registration_token_is_single_use`, `expired_token_rejected`, `agent_cert_from_other_ca_rejected`, `issued_cert_tolerates_five_minutes_of_skew`, `cert_renewed_at_two_thirds_of_ttl` (paused time), `reconnect_replays_from_last_ack`, `reconnect_mid_batch_leaves_index_exact`, `slow_proxy_stream_does_not_block_control`, `backoff_sequence_is_capped_at_60s`.
- [ ] **Step 2: Run** `cargo test -p admiral-tunnel`. Expected: FAIL.
- [ ] **Step 3: Implement** with tonic and rustls (ring). Control messages: `Register`, `Heartbeat`, `Deltas`, `Ack`, `Command` (RBAC apply, relist). The server never trusts a cluster id the agent sends; it comes from the client certificate.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(tunnel): add mTLS gRPC tunnel with registration and replay`.

### Task 4: KubernetesDriver (#6)

**Files:**
- Create: `crates/admiral-driver/src/{lib.rs,kubernetes/mod.rs,kubernetes/watch.rs,kubernetes/proxy.rs,status.rs}`, `crates/admiral-agent/src/{main.rs,config.rs}`
- Test: `crates/admiral-driver/tests/{status.rs,watch.rs,proxy.rs}` using `tower-test` mock services and recorded JSON in `tests/fixtures/`

**Interfaces:**
- Consumes: `IndexRow`, `StatusSummary`, `WatchEvent`, `AgentConn`.
- Produces:
  - `trait ClusterDriver: Send + Sync { fn kind(&self) -> DriverKind; async fn discover(&self) -> Result<Vec<ResourceType>, DriverError>; fn watch(&self, ty: &ResourceType, from: Option<String>) -> BoxStream<'static, Result<WatchEvent, DriverError>>; async fn proxy(&self, req: ProxyRequest, as_user: &Impersonation) -> Result<ProxyResponse, DriverError>; async fn apply_rbac(&self, desired: &RbacSet, mode: OwnershipMode) -> Result<RbacReport, DriverError>; }`
  - Emits `WatchEvent` from task 1.
  - `Impersonation { user: String, groups: Vec<String>, extra: BTreeMap<String, Vec<String>> }`
  - `fn summarize(kind: &str, obj: &serde_json::Value) -> StatusSummary` in `status.rs`
  - `apply_rbac` is stubbed with `unimplemented` errors until task 9 fills it.

- [ ] **Step 1: Write failing tests** `status_summary_cases` (table: Pod CrashLoopBackOff is `Failed`, Pod Running and Ready is `Ready`, Deployment with unavailable replicas is `Progressing` or `Degraded` by condition, Node NotReady is `Failed`, unknown kind with a Ready condition, unknown kind without conditions is `Unknown`), `secret_rows_carry_no_data`, `watch_relists_on_410_gone`, `watch_resumes_from_given_version`, `watch_flood_stays_within_channel_bound` (10,000 events, bounded channel of 10,000, batches of at most 1,000 rows or 250 ms), `impersonation_headers_set_for_user_and_groups`, `client_auth_headers_never_forwarded`, `discover_skips_resources_without_list_and_watch`.
- [ ] **Step 2: Run** `cargo test -p admiral-driver`. Expected: FAIL.
- [ ] **Step 3: Implement** with kube-rs `watcher` and `discovery`. Uses only built-in kinds and needs no CRDs. Wire the agent binary: read config, register, connect, run one watcher per discovered type.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(driver): add Kubernetes driver and agent watch loop`.

### Task 5: Index and query language (#7)

**Files:**
- Create: `crates/admiral-index/src/{query/lexer.rs,query/parser.rs,query/mod.rs,index.rs,subscribe.rs,scope.rs}`, migration `0003_index.sql` (FTS5 table on SQLite, `tsvector` column and GIN index on Postgres)
- Test: `crates/admiral-index/tests/{parse.rs,apply.rs,search.rs,subscribe.rs}`

**Interfaces:**
- Consumes: `WatchEvent`, `DeltaBatch`, `Store`.
- Produces:
  - `parse(input: &str) -> Result<Query, ParseError>`; `ParseError { message: String, span: Range<usize> }`.
  - Grammar: whitespace-separated terms; `key:value` where key is `kind`, `cluster`, `ns`, `name`, `status` or `label`; `label:` takes `k=v`, `k!=v`, `k` (exists) or `!k`; values allow `*` globs; any other word is free text over name and labels; `-term` negates.
  - `enum AccessScope { All, Clusters(Vec<ClusterId>), Projects(Vec<ProjectId>), Rules(Arc<ScopeRules>) }` (task 11 builds `Rules`).
  - `Index::apply(&self, cluster: &ClusterId, batch: DeltaBatch) -> Result<Vec<RowChange>, IndexError>`
  - `Index::search(&self, scope: &AccessScope, q: &Query, page: PageRequest) -> Result<Page<IndexRow>, IndexError>` with an opaque keyset cursor.
  - `Index::subscribe(&self, scope: AccessScope, q: Query) -> impl Stream<Item = RowChange>`

- [ ] **Step 1: Write failing tests** `parse_cases` (table of valid inputs and their `Query`, invalid inputs and their spans, for example `label:=x` fails at 6..7), `apply_is_idempotent_for_same_resource_version`, `older_resource_version_ignored`, `relisted_removes_rows_not_in_list`, `search_paginates_with_stable_cursor_under_inserts`, `scope_hides_rows_outside_clusters`, `subscription_receives_only_matching_changes`, `same_results_on_sqlite_and_postgres`.
- [ ] **Step 2: Run** `cargo test -p admiral-index`. Expected: FAIL.
- [ ] **Step 3: Implement.** The parser is hand-written; it is small and needs good error spans.
- [ ] **Step 4: Run** the tests on both backends. Expected: PASS.
- [ ] **Step 5: Commit** `feat(index): add cross-cluster index and query language`.

### Task 6: REST API, OpenAPI, jobs and server wiring (#8)

**Files:**
- Create: `crates/admiral-api/src/{router.rs,state.rs,problem.rs,sse.rs,openapi.rs,jobs.rs,routes/{clusters,resources,objects,jobs}.rs}`, `crates/admiral-server/src/{main.rs,config.rs}`
- Test: `crates/admiral-api/tests/{openapi.rs,clusters.rs,resources.rs,objects.rs,jobs.rs}` with `insta` snapshots

**Interfaces:**
- Consumes: `Store`, `Index`, `TunnelServer`.
- Produces:
  - `router(state: AppState) -> axum::Router`
  - Routes: `GET/POST /api/v1/clusters` (POST returns a registration command and token), `GET/PATCH/DELETE /api/v1/clusters/{id}` (PATCH sets `mode`), `GET /api/v1/resources?q=&cursor=&limit=`, `GET /api/v1/resources/stream?q=` (SSE), `GET /api/v1/resources/{uid}/related`, `GET/PUT /api/v1/clusters/{id}/objects/{group}/{version}/{kind}/{namespace}/{name}` (PUT with `?dryRun=true` returns `{ diff }` from server-side apply dry run), `GET /api/v1/jobs/{id}`, `GET /api/v1/jobs/{id}/events` (SSE), `GET /api/v1/openapi.json`.
  - `Problem { type_: String, title: String, status: u16, detail: Option<String> }` rendered as `application/problem+json`.
  - SSE sends a comment heartbeat every 15 seconds.
  - Server config: TOML at the path in `--config`, overridden by `ADMIRAL_*` environment variables.

- [ ] **Step 1: Write failing tests** `openapi_document_matches_snapshot`, `unknown_route_returns_problem_json`, `create_cluster_returns_single_use_command`, `patch_mode_rejects_unknown_value`, `resources_query_error_returns_span`, `sse_heartbeat_every_15s` (paused time), `dry_run_returns_diff_without_writing`, `job_stream_ends_with_result_event`.
- [ ] **Step 2: Run** `cargo test -p admiral-api`. Expected: FAIL.
- [ ] **Step 3: Implement** with axum and utoipa. Authentication is a pass-through extractor that task 7 replaces.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(api): add REST API, OpenAPI document, SSE and jobs`.

### Task 7: Local users, passkeys, TOTP, sessions and tokens (#9)

**Files:**
- Create: `crates/admiral-auth/src/{password.rs,session.rs,token.rs,passkey.rs,totp.rs,ratelimit.rs,extract.rs}`, `crates/admiral-store/src/users.rs` plus migration `0004_users.sql`, `crates/admiral-api/src/routes/auth.rs`
- Test: `crates/admiral-auth/tests/{password.rs,session.rs,token.rs,totp.rs,ratelimit.rs}`

**Interfaces:**
- Produces:
  - `hash_password(&str) -> PasswordHash` with argon2id, memory 19,456 KiB, 2 iterations, parallelism 1.
  - `enum StoredHash { Argon2id(String), Bcrypt(String), Pbkdf2Sha256 { iterations: u32, salt: Vec<u8>, hash: Vec<u8> } }`; `verify(&StoredHash, &str) -> Verify { ok: bool, needs_rehash: bool }`. Task 19 relies on bcrypt and PBKDF2 for Rancher users.
  - `Sessions::create(user: UserId, meta: SessionMeta) -> SessionToken`, `Sessions::touch`, `Sessions::revoke_user(UserId)`, idle 30 minutes, absolute 12 hours.
  - API tokens: `adm_` plus 32 random bytes base64url, stored as SHA-256, `TokenScope::{Global, Cluster(ClusterId), Project(ProjectId)}`, required `expires_at`.
  - TOTP per RFC 6238: 30-second step, 6 digits, accepts one step either side.
  - Login rate limit: 5 failures per 15 minutes per user and IP, then 429.
  - `admiral-server bootstrap-admin` prints a one-time setup URL valid for 1 hour.
  - axum extractor `Authenticated { user: UserId, groups: Vec<GroupRef>, via: AuthMethod }`.

- [ ] **Step 1: Write failing tests** `argon2id_params_match_policy`, `bcrypt_and_pbkdf2_verify_and_ask_for_rehash`, `session_expires_after_idle`, `session_expires_at_absolute_limit`, `token_scope_enforced`, `token_without_expiry_rejected`, `login_rate_limited_after_five_failures`, `totp_accepts_adjacent_step_only`, `passkey_registration_and_login_roundtrip` (webauthn-rs software authenticator).
- [ ] **Step 2: Run** `cargo test -p admiral-auth`. Expected: FAIL.
- [ ] **Step 3: Implement** and replace the pass-through extractor from task 6.
- [ ] **Step 4: Run** `cargo test --workspace`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(auth): add local users, passkeys, TOTP, sessions and API tokens`.

### Task 8: OIDC and GitHub sign-in (#10)

**Files:**
- Create: `crates/admiral-auth/src/{provider.rs,oidc.rs,github.rs,groups.rs}`, migration `0005_providers.sql`
- Test: `crates/admiral-auth/tests/{oidc.rs,github.rs,groups.rs}` with a fake provider served by axum inside the test

**Interfaces:**
- Produces:
  - `trait IdentityProvider { fn id(&self) -> &ProviderId; async fn begin(&self, redirect: Url) -> Result<AuthStart, AuthError>; async fn complete(&self, cb: Callback) -> Result<ExternalIdentity, AuthError>; async fn refresh_groups(&self, who: &ExternalIdentity) -> Result<Vec<GroupRef>, AuthError>; }`
  - OIDC with PKCE, `state` and `nonce`, issuer check, audience check.
  - GitHub groups are `org` and `org/team` slugs.
  - A background job refreshes groups for users with live sessions every 15 minutes and revokes bindings that no longer apply.

- [ ] **Step 1: Write failing tests** `oidc_rejects_wrong_issuer`, `oidc_rejects_replayed_state`, `oidc_rejects_wrong_audience`, `github_teams_become_org_slash_team_groups`, `group_removed_at_provider_revokes_on_next_refresh`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** with `openidconnect`.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(auth): add OIDC and GitHub sign-in with group refresh`.

### Task 9: Role templates, bindings and RBAC sync (#11)

**Files:**
- Create: `crates/admiral-rbac/src/{template.rs,binding.rs,render.rs,builtin.rs}`, `crates/admiral-driver/src/kubernetes/rbac.rs`, migration `0006_rbac.sql`, `crates/admiral-api/src/routes/rbac.rs`
- Test: `crates/admiral-rbac/tests/{render.rs,inherit.rs}`, `crates/admiral-driver/tests/rbac.rs`

**Interfaces:**
- Produces:
  - `RoleTemplate { id, name, level: Level, rules: Vec<PolicyRule>, inherits: Vec<RoleTemplateId>, builtin: bool }`, `enum Level { Global, Cluster, Project }`
  - `Binding { id, subject: Subject, template: RoleTemplateId, target: Target }`, `enum Subject { User(UserId), Group(GroupRef) }`, `enum Target { Global, Cluster(ClusterId), Project(ProjectId) }`
  - Built-in templates named like Rancher's: `cluster-owner`, `cluster-member`, `project-owner`, `project-member`, `read-only`.
  - `render(cluster: &ClusterId, templates, bindings, projects) -> Result<RbacSet, RenderError>`, a pure function.
  - `KubernetesDriver::apply_rbac` fills the task 4 stub: `Observe` never writes; `Shared` creates and updates only objects labelled `app.kubernetes.io/managed-by=admiral`; `Owned` behaves like `Shared` here, and only `detach` (task 19) removes foreign objects.

- [ ] **Step 1: Write failing tests** `render_is_deterministic`, `inherited_templates_flatten`, `inheritance_cycle_is_an_error`, `project_binding_renders_rolebinding_in_each_namespace`, `observe_mode_never_writes`, `shared_mode_leaves_foreign_objects`, `drift_on_managed_role_is_repaired_and_reported`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(rbac): add role templates, bindings and RBAC sync`.

### Task 10: Projects (#12)

**Files:**
- Create: `crates/admiral-rbac/src/project.rs`, migration `0007_projects.sql`, `crates/admiral-api/src/routes/projects.rs`, `crates/admiral-driver/src/kubernetes/quota.rs`
- Test: `crates/admiral-rbac/tests/project.rs`, `crates/admiral-driver/tests/quota.rs`

**Interfaces:**
- Produces: `Project { id: ProjectId, name, clusters: Vec<ClusterId>, quota: Option<QuotaSpec>, default_limits: Option<LimitSpec> }`. A namespace belongs to a project when its `field.cattle.io/projectId` label equals the project's short id. The agent writes a ResourceQuota and a LimitRange named `admiral-project` into each member namespace.

- [ ] **Step 1: Write failing tests** `namespace_label_assigns_project`, `unlabelled_namespace_belongs_to_no_project`, `project_spans_two_clusters`, `quota_applied_to_every_member_namespace`, `rancher_project_id_parses`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(rbac): add projects across clusters with quotas`.

### Task 11: Effective access (#13)

**Files:**
- Create: `crates/admiral-rbac/src/{effective.rs,semantics.rs,scope.rs}`, `crates/admiral-api/src/routes/access.rs`, migration `0008_native_rbac.sql`
- Test: `crates/admiral-rbac/tests/{semantics.rs,effective.rs}`

**Interfaces:**
- Consumes: native Role, ClusterRole and binding objects, which the agent sends with their rules into the `native_rbac` table (extend the task 4 watch for the four RBAC kinds).
- Produces:
  - `Access::who_can(&self, verb: &str, res: &ResourceRef) -> Vec<AccessGrant>`, `Access::what_can(&self, s: &Subject) -> Vec<AccessGrant>`
  - `AccessGrant { subject, cluster, verbs, resources, namespaces, via: GrantSource }`, `enum GrantSource { Admiral(BindingId), Native { kind, namespace, name } }`
  - `Access::scope_for(&self, user: &Authenticated) -> AccessScope`, used by every index search from now on.
  - Routes `GET /api/v1/access/who-can?verb=&resource=`, `GET /api/v1/access/what-can?subject=`.

- [ ] **Step 1: Write failing tests** `rbac_semantics_cases` (table: `*` verbs and resources, `resourceNames`, subresources such as `pods/log`, aggregated ClusterRoles, namespace-scoped RoleBinding to a ClusterRole), `combines_admiral_and_native_grants`, `group_grants_expand_to_members`, `search_respects_scope_for_user`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(rbac): add effective access queries and search scopes`.

### Task 12: Audit log with hash chain (#14)

**Files:**
- Create: `crates/admiral-audit/src/{event.rs,chain.rs,middleware.rs,export/mod.rs,export/otlp.rs,export/syslog.rs,export/webhook.rs}`, migration `0009_audit.sql`
- Test: `crates/admiral-audit/tests/{chain.rs,export.rs}`

**Interfaces:**
- Produces:
  - `AuditEvent { seq: i64, at, actor: Actor, cluster: Option<ClusterId>, verb, resource: Option<ResourceRef>, outcome: Outcome, request_id, prev_hash: [u8; 32], hash: [u8; 32] }`
  - `hash = SHA-256(prev_hash || canonical JSON of the event without hash)`; canonical JSON sorts keys and has no whitespace. The first entry uses 32 zero bytes as `prev_hash`.
  - Appends run in one transaction that locks the single `audit_head` row, so concurrent replicas keep one chain.
  - `Audit::verify(&self, range: RangeInclusive<i64>) -> VerifyReport { checked: u64, first_break: Option<i64> }`
  - Exporters deliver at least once from a stored cursor. Webhooks carry `X-Admiral-Signature: sha256=<hex HMAC>`. Syslog is RFC 5424 over TCP with TLS.
  - axum middleware records every `/api/v1` and `/k8s/clusters` request.

- [ ] **Step 1: Write failing tests** `chain_detects_edited_row`, `chain_detects_deleted_row`, `concurrent_writers_keep_one_chain` (Postgres), `canonical_json_is_key_sorted`, `webhook_signature_verifies`, `exporter_resumes_from_cursor`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(audit): add hash-chained audit log and exporters`.

### Task 13: Kubeconfig proxy and admiral login (#15)

**Files:**
- Create: `crates/admiral-compat/src/kube_proxy.rs`, `crates/admiral-api/src/routes/kube_tokens.rs`, `crates/admiral-cli/src/commands/{login.rs,kube_credential.rs}`, `crates/admiral-cli/src/credcache.rs`
- Test: `crates/admiral-compat/tests/kube_proxy.rs`, `crates/admiral-cli/tests/kube_credential.rs`

**Interfaces:**
- Produces:
  - Route `/k8s/clusters/{id}/{*path}`: authenticates a bearer token, strips client `Authorization` and `Impersonate-*` headers, sets impersonation from the user, forwards through `TunnelServer::proxy`, and streams the response for watches and logs.
  - `POST /api/v1/kube-tokens` returns `{ token, expires_at }` valid for 15 minutes.
  - `admiral login --server <url>` signs in with the OIDC device flow or the browser and writes one kubeconfig context per cluster the user can reach, each with an `exec` entry.
  - `admiral kube-credential --server <url> --cluster <id>` prints a `client.authentication.k8s.io/v1` `ExecCredential` with `expirationTimestamp`, caching the token in the OS keyring (keyring crate) or a 0600 file when there is none.

- [ ] **Step 1: Write failing tests** `rancher_style_path_routes_to_cluster`, `proxy_strips_client_auth_and_sets_impersonation`, `watch_streams_through_proxy`, `exec_credential_json_shape`, `expired_cached_token_is_refreshed`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(compat): add kubeconfig proxy and kubectl credential plugin`.

### Task 14: Logs, exec, shell and port-forward (#16)

**Files:**
- Create: `crates/admiral-api/src/routes/{logs.rs,exec.rs,portforward.rs}`, `crates/admiral-driver/src/kubernetes/streams.rs`, `crates/admiral-core/src/escape.rs`
- Test: `crates/admiral-api/tests/{logs.rs,exec.rs}`, `crates/admiral-core/tests/escape.rs`

**Interfaces:**
- Produces:
  - `GET /api/v1/logs?q=&since=&follow=` streams SSE events `LogLine { cluster, namespace, pod, container, ts, line }` merged by timestamp across pods; at most 50 pods per stream by default.
  - WebSocket `/api/v1/clusters/{id}/exec` passes the `v5.channel.k8s.io` subprotocol through, including resize messages; node shells run a debug pod.
  - WebSocket `/api/v1/clusters/{id}/portforward` for the CLI.
  - `escape_for_display(&str) -> Cow<str>` removes C0 and C1 control characters except tab; HTML escaping is the UI's job.

- [ ] **Step 1: Write failing tests** `log_fanout_merges_by_timestamp`, `log_stream_caps_pods`, `one_stalled_log_stream_does_not_block_another`, `control_sequences_removed_from_log_lines`, `exec_resize_message_forwarded`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(api): add multi-pod logs, exec, shell and port-forward`.

### Task 15: UI foundation (#17)

**Files:**
- Create: `web/package.json`, `web/vite.config.ts`, `web/src/{main.tsx,app.tsx,api/client.ts,i18n/{en.json,th.json},theme/,components/CommandPalette.tsx}`, `web/tests/fake-api/`, `crates/admiral-server/src/embed.rs`
- Modify: `.github/workflows/ci.yml` (add `web` job), `.github/dependabot.yml` (add `npm` for `/web`)
- Test: `web/src/**/*.test.ts(x)` (Vitest), `web/tests/e2e/*.spec.ts` (Playwright)

**Interfaces:**
- Consumes: `/api/v1/openapi.json` from task 6; `pnpm gen:api` generates `web/src/api/schema.d.ts` with openapi-typescript, and CI fails if the generated file is stale.
- Produces: theme tokens (light and dark, compact and comfortable), the `t()` translation function, `useQuery(q)` returning live rows over SSE, the command palette on Ctrl+K and Cmd+K, the app shell with navigation. `admiral-server` serves `web/dist` through rust-embed when built with feature `embed-ui`.

- [ ] **Step 1: Write failing tests** `palette_opens_on_ctrl_k_and_meta_k`, `palette_lists_resources_from_query`, `theme_follows_system_then_persists_choice`, `thai_has_every_english_key`, `shell_has_no_axe_violations`, and a `size-limit` budget of 300 KB gzipped for the first load.
- [ ] **Step 2: Run** `pnpm -C web test && pnpm -C web e2e && pnpm -C web size`. Expected: FAIL.
- [ ] **Step 3: Implement** with SolidJS, Kobalte and Tailwind.
- [ ] **Step 4: Run** the same commands. Expected: PASS. Add the `web` job to the ruleset's required checks.
- [ ] **Step 5: Commit** `feat(web): add UI foundation, themes, i18n and command palette`.

### Task 16: UI fleet overview, explorer and resource page (#18)

**Files:**
- Create: `web/src/routes/{index.tsx,explore.tsx,resource.tsx}`, `web/src/components/{FleetCard.tsx,VirtualTable.tsx,YamlEditor.tsx,DiffView.tsx,RelationGraph.tsx}`
- Test: `web/tests/e2e/{overview,explorer,resource}.spec.ts`

**Interfaces:**
- Consumes: `useQuery`, `/api/v1/resources`, `/resources/{uid}/related`, `/objects/...` with `dryRun`.
- Produces: routes `/`, `/explore?q=`, `/c/:cluster/:group/:version/:kind/:namespace/:name`. Applying YAML always shows the dry-run diff first and needs a second click to confirm.

- [ ] **Step 1: Write failing tests** `overview_shows_disconnected_agent_and_expiring_cert`, `explorer_shows_live_row_update`, `saved_view_url_roundtrip`, `apply_requires_viewing_diff`, `table_scrolls_100k_rows_at_60fps` (Playwright trace, no long frames over 16.7 ms in a 5-second scroll), `labels_with_html_render_as_text`.
- [ ] **Step 2: Run** `pnpm -C web e2e`. Expected: FAIL.
- [ ] **Step 3: Implement** with `@tanstack/solid-virtual` and CodeMirror 6.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(web): add fleet overview, explorer and resource page`.

### Task 17: UI logs, shell, access and audit views (#19)

**Files:**
- Create: `web/src/routes/{logs.tsx,shell.tsx,access.tsx,audit.tsx}`, `web/src/components/{LogStream.tsx,Terminal.tsx}`
- Test: `web/tests/e2e/{logs,shell,access,audit}.spec.ts`

- [ ] **Step 1: Write failing tests** `log_view_streams_two_pods_and_filters`, `log_view_pauses_and_resumes`, `shell_sends_resize_on_window_change`, `who_can_view_lists_grant_sources`, `audit_view_shows_verify_result`.
- [ ] **Step 2: Run** `pnpm -C web e2e`. Expected: FAIL.
- [ ] **Step 3: Implement** with xterm.js for the shell.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(web): add logs, shell, access and audit views`.

### Task 18: CLI (#20)

**Files:**
- Create: `crates/admiral-cli/src/commands/{clusters.rs,get.rs,logs.rs,exec.rs,port_forward.rs,access.rs,audit.rs,version.rs}`, `crates/admiral-cli/src/output.rs`
- Test: `crates/admiral-cli/tests/{get.rs,output.rs,exit_codes.rs}` against an in-process server

**Interfaces:**
- Produces: `admiral clusters list|add|remove|mode`, `admiral get <query>`, `admiral logs <query> [-f]`, `admiral exec`, `admiral port-forward`, `admiral access who-can|what-can`, `admiral audit verify`, `admiral version`, plus `login` and `kube-credential` from task 13. `-o table|json|yaml`. Exit codes: 0 success, 1 error, 2 usage, 3 not signed in or forbidden.

- [ ] **Step 1: Write failing tests** `get_renders_table_snapshot`, `json_output_is_stable`, `exit_code_cases`, `query_error_points_at_span`.
- [ ] **Step 2: Run** `cargo test -p admiral-cli`. Expected: FAIL.
- [ ] **Step 3: Implement** with clap.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): add admiral CLI`.

### Task 19: admiral migrate rancher (#21)

**Files:**
- Create: `crates/admiral-migrate/src/rancher/{client.rs,model.rs,plan.rs,apply.rs,rollback.rs,detach.rs,journal.rs}`, migration `0010_migration_journal.sql` (adds `rancher_ref` columns), `crates/admiral-cli/src/commands/migrate.rs`
- Create: `crates/admiral-migrate/tests/fixtures/rancher-2.13/` through `rancher-2.16/` (recorded `/v3` responses with fake data)
- Test: `crates/admiral-migrate/tests/{plan.rs,apply.rs,rollback.rs,detach.rs}`; end-to-end in `.github/workflows/e2e-migrate.yml`

**Interfaces:**
- Consumes: Rancher `/v3/clusters`, `/v3/users`, `/v3/principals`, `/v3/authconfigs`, `/v3/globalroles`, `/v3/globalrolebindings`, `/v3/roletemplates`, `/v3/clusterroletemplatebindings`, `/v3/projectroletemplatebindings`, `/v3/projects`; `StoredHash` from task 7; `render` from task 9; `Project` from task 10.
- Produces:
  - `MigrationPlan { source_version: String, items: Vec<PlanItem> }`, `PlanItem { kind, rancher_id, action: Action }`, `enum Action { Create, Update, Skip { reason }, Unsupported { reason, phase: String } }`, saved as JSON and printed as a table.
  - `apply` records every write in `migration_journal` and is keyed by `rancher_ref`, so a second run creates nothing new.
  - The agent is installed into each cluster through Rancher's own `/k8s/clusters/<id>` with the admin token, in `observe` mode.
  - `rollback [--cluster]` undoes journal entries in reverse order.
  - `detach --cluster <id> --confirm <id>` works only in `owned` mode, removes `cattle-cluster-agent` and Rancher's RBAC objects in that cluster, and is journalled.
  - Provider client secrets are read from Rancher and stored encrypted; the token used for the migration is never stored.

- [ ] **Step 1: Write failing tests** `plan_matches_snapshot_for_each_version`, `apply_twice_creates_nothing_new`, `rollback_removes_everything_written`, `detach_refused_unless_owned`, `external_principals_map_to_provider_identities`, `inherited_role_templates_import_in_dependency_order`, `long_and_unicode_names_preserved`, `local_user_hashes_import_and_rehash_on_login`, `unsupported_provider_appears_in_plan_with_phase`.
- [ ] **Step 2: Run** `cargo test -p admiral-migrate`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Add** the end-to-end workflow: k3d, the Rancher 2.16 Helm chart, seed users, roles and projects through the Rancher API, run `plan` and `apply`, then check through `who-can` that each seeded user has exactly the access they had, run `rollback`, and check the cluster matches its starting state. It runs nightly and on pull requests labelled `area/migrate`.
- [ ] **Step 6: Commit** `feat(migrate): add admiral migrate rancher`.

### Task 20: flotilla end-to-end suite (#22)

**Files:**
- Create: `.github/workflows/e2e-flotilla.yml`, `tests/e2e/flotilla/`
- Test: import, explorer search, logs, exec, and RBAC in observe mode

- [ ] **Step 1: Write failing tests** `flotilla_cluster_imports_and_indexes_core_kinds`, `flotilla_logs_and_exec_work`, `flotilla_rbac_stays_observe_until_kinds_served`.
- [ ] **Step 2: Run** against a flotilla release. Expected: FAIL until the agent supports it; if no flotilla release exists yet, the job reports that and skips.
- [ ] **Step 3: Fix** whatever the suite finds in the driver, keeping to built-in kinds.
- [ ] **Step 4: Run** the suite. Expected: PASS.
- [ ] **Step 5: Commit** `test(e2e): add flotilla end-to-end suite`.

### Task 21: Packaging and install guide (#23)

**Files:**
- Modify: `.github/workflows/release.yml`
- Create: `charts/admiral-server/`, `charts/admiral-agent/`, `Dockerfile.server`, `Dockerfile.agent`, `docs/install.md`
- Test: `helm lint`, `helm template` snapshots in `charts/*/tests/`, chart-testing install on k3d

**Interfaces:**
- Produces: binaries for Linux (x86_64 and aarch64, musl), macOS (arm64 and x86_64) and Windows (x86_64); images `ghcr.io/raven-clown/admiral-server` and `ghcr.io/raven-clown/admiral-agent` on a distroless static base, running as non-root; charts pushed as OCI to `ghcr.io/raven-clown/charts`; cosign keyless signatures, SBOMs and build provenance attestations for every artifact.

- [ ] **Step 1: Write failing checks** `chart_templates_match_snapshot`, `agent_chart_rbac_is_minimal` (asserts the exact rule list), `images_run_as_nonroot`.
- [ ] **Step 2: Run** `helm lint charts/* && ct install`. Expected: FAIL.
- [ ] **Step 3: Implement** the charts, Dockerfiles and release jobs, and write `docs/install.md` for the binary, the image and the chart.
- [ ] **Step 4: Run** the checks and a dry-run release on a test tag in a fork. Expected: PASS.
- [ ] **Step 5: Commit** `ci(release): add signed binaries, images and Helm charts`.

### Task 22: Benchmark harness and first results (#24)

**Files:**
- Create: `crates/admiral-bench/`, `bench/scenarios/{b1-footprint,b2-search,b3-ui,b5-agent,b7-first-cluster}.toml`, `bench/results/`, `docs/benchmarks.md`
- Test: `crates/admiral-bench/tests/{scenario.rs,report.rs}`

**Interfaces:**
- Produces: `admiral-bench run <scenario> --target admiral|rancher` runs on Linux with kwok clusters, installs the target, loads the scenario, measures, and writes `bench/results/<version>/<scenario>.json` plus a Markdown table. Rancher is pinned to the latest 2.16 patch, on the same machine size.

- [ ] **Step 1: Write failing tests** `scenario_files_parse`, `report_renders_markdown_snapshot`, `percentiles_are_computed_exactly` (table of known samples).
- [ ] **Step 2: Run** `cargo test -p admiral-bench`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** B1, B2, B3, B5 and B7 for both targets and publish the numbers in `docs/benchmarks.md`, including any where admiral loses. Update the "Better" column of design section 8 for each measured row.
- [ ] **Step 5: Commit** `feat(bench): add benchmark harness and first results`.

---

## Done for P1

- Every task above is merged and its issue closed.
- A clean machine can install admiral from `docs/install.md`, import a k3s cluster and a flotilla cluster, sign in with OIDC, and run `admiral migrate rancher plan` and `apply` against the end-to-end Rancher.
- `docs/benchmarks.md` has first numbers for B1, B2, B3, B5 and B7.
- Release `v0.1.0` is tagged on main, with notes from `CHANGELOG.md`.
