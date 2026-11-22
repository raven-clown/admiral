# admiral design

Status: draft for review, 2026-11-22.

admiral is a multi-cluster management platform for Kubernetes, written in Rust. The goal is every feature Rancher has, done better, plus a move from Rancher that works on day one: import the clusters you have, bring your users, roles and projects, run side by side, then switch off Rancher.

This document covers who admiral is for, the architecture, identity and access, the UI, compatibility with Rancher, a feature matrix against Rancher, Lens, Portainer, Headlamp and the OpenShift console with Advanced Cluster Management (ACM), the phased plan, testing, benchmarks and failure handling.

## 1. Goals

**Users.** Platform teams of every size, from two clusters on one machine to more than a thousand clusters in an enterprise. The same build has to be light at the small end and hold up at the large end.

**Success means:**

1. A Rancher user can move to admiral without rebuilding clusters, re-creating users or rewriting Git repositories, and can run both during the move.
2. Every Rancher feature has a matching admiral feature, or a written reason it is out of scope (section 9).
3. Each "better" claim in section 8 is either plainly true by design or backed by a published, repeatable benchmark (section 11). Where it is not yet measured, the matrix says so.

**Non-goals.** Operating system management (Elemental), virtualization (Harvester) and a bundled service mesh. Section 9 explains each.

## 2. What people dislike about the tools today

Collected from the Rancher and SUSE forums, the Rancher Slack archive, release notes and comparison write-ups, November 2026.

- **Rancher needs a Kubernetes cluster to run.** State lives as CRDs in the local cluster's etcd. Since 2.13 the local cluster must also have the API aggregation layer.
- **Rancher's UI slows down as workloads grow.** Reports span many versions (2.6 Cluster Explorer, 2.8.2 login pages taking minutes, 2.12 lists that hang). The usual advice is to give the Rancher pods more CPU and memory. Server-side pagination (`ui-sql-cache`) became the default in 2.12 and needs ephemeral disk on every node running agents.
- **Rancher upgrades go one minor at a time,** from the latest patch of one minor to the latest patch of the next. Kubernetes support windows are short (2.13 dropped 1.31 and older).
- **Fleet runs out of room in etcd at scale.** It creates one BundleDeployment per bundle and cluster (twice that for Helm bundles). SUSE's own test at 100,000 BundleDeployments had to grow etcd from 2 GB to 8 GB, turn off per-cluster status reporting, and found failures hard to diagnose. Git polling at scale causes rate limits.
- **Rancher Extensions** had a path traversal hole (CVE-2026-25705) and SAML had a replay hole (CVE-2026-44946).
- **Lens** is a single-user desktop app with a proprietary free tier. **Headlamp** is the official successor to the archived Kubernetes Dashboard, but its multi-cluster support is basic. **Portainer** covers Docker and Kubernetes but stays shallow on Kubernetes. **OpenShift console with ACM** is strong on fleet policy and search but tied to OpenShift.

## 3. Architecture

```
            browser (SolidJS UI)     admiral CLI     Terraform / scripts
                       \                 |                 /
                        +---- REST + SSE (OpenAPI) -------+
                                         |
                              +---------------------+
                              |    admiral-server   |   one binary, UI embedded
                              |  api  auth  rbac    |
                              |  index  audit       |   SQLite (single node)
                              |  compat  migrate    |   or Postgres (HA)
                              +---------------------+
                                ^       ^       ^
              mTLS HTTP/2 tunnels, opened by the agents (outbound only)
                                |       |       |
                     +----------+  +----------+  +----------+
                     |  agent   |  |  agent   |  |  agent   |
                     | k3s/RKE2 |  | EKS/AKS  |  | flotilla |
                     +----------+  +----------+  +----------+
```

### 3.1 admiral-server

- One Rust binary (tokio, axum). The UI is built at release time and embedded.
- State in SQLite for a single node, or Postgres for high availability. With Postgres, run any number of replicas behind a load balancer. One replica holds the leader role through a Postgres advisory lock and runs the reconcilers; every replica serves the API and holds agent tunnels.
- No Kubernetes cluster is needed to run it. It ships as a binary, a container image and a Helm chart.
- Upgrades are database migrations. Every release is tested to upgrade from each of the previous four minor versions in one step. The server takes a backup before it migrates.

### 3.2 admiral-agent

- A small binary deployed in each managed cluster as a Deployment with a ServiceAccount. It uses only built-in kinds, so it runs on clusters without CRD support (flotilla before its phase 3).
- It dials out to the server. Managed clusters open no inbound ports.
- It registers with a one-time token, receives a client certificate signed by the server's CA, and renews it before expiry.
- It watches resources with kube-rs and sends the server only index deltas: identity, labels, owner references and a status summary. Full objects stay in the cluster and are fetched through the tunnel when a user opens one.
- It forwards user requests (API calls, logs, exec, port-forward) through the tunnel and sets Kubernetes impersonation headers, so the managed cluster's own RBAC decides every request.

### 3.3 Tunnel

- gRPC over HTTP/2 with mutual TLS (tonic, rustls).
- One long-lived bidirectional stream carries control messages: register, heartbeat, index deltas, RBAC sync. Separate HTTP/2 streams carry proxied requests and byte streams for logs, exec and port-forward, so one slow log stream never blocks others.
- On reconnect the agent sends its last acknowledged resource versions and only the changes since are replayed. If the cluster compacted past them (410 Gone), the agent relists that kind.

### 3.4 ClusterDriver

A trait in the agent that hides the cluster type.

- `KubernetesDriver` (kube-rs) covers every conformant cluster. flotilla serves the Kubernetes API itself (kubectl and kube-rs work against it, standard auth, list and watch with `resourceVersion`), so this driver covers flotilla too.
- `FlotillaDriver` adds lifecycle work that sits outside the Kubernetes API: create servers, join nodes, back up, restore and upgrade. admiral will call a flotilla lifecycle API or Rust crate for this, not shell out to its CLI. This is phase 4 and depends on flotilla exposing that interface.
- Dependency on flotilla: RBAC sync needs ClusterRole, Role, ClusterRoleBinding and RoleBinding. flotilla's phase 1 list of core kinds does not name them yet, so until it serves them admiral runs flotilla clusters in `observe` mode for RBAC.

### 3.5 Index and query

- The server stores one row per resource across every cluster: cluster, group, version, kind, namespace, name, uid, labels, owner references, a status summary, resource version and update time. Secrets are indexed by name and type only.
- Search uses SQLite FTS5 or Postgres `tsvector` plus ordinary indexes on the common filters.
- The query language is small and readable: `kind:Pod status:CrashLoopBackOff cluster:prod-* label:app=web ns:payments`. Every UI list, the CLI and the API use the same parser.
- Lists are paged, sorted and filtered on the server. Clients subscribe to a query over SSE and receive row-level changes.
- Target (benchmark B2): first page of 100,000 pods across 500 clusters in under 200 ms at p99.

### 3.6 API

- REST and JSON, described by OpenAPI 3.1, versioned under `/api/v1`. The UI, the CLI and the Terraform provider use this API and nothing else.
- Long operations (migration, upgrades) are jobs with their own resource and an SSE progress stream.
- Errors use RFC 9457 problem details with a stable `type` per error.
- `/k8s/clusters/<id>` proxies the Kubernetes API of a managed cluster, the same path Rancher uses, so existing kubeconfig files keep working after a DNS change (section 6).

### 3.7 Workspace layout

| Crate | Purpose |
| --- | --- |
| `admiral-core` | Domain types and errors, no I/O |
| `admiral-store` | sqlx over SQLite and Postgres, migrations, backup |
| `admiral-tunnel` | Protocol messages, agent and server ends |
| `admiral-driver` | `ClusterDriver` trait, Kubernetes and flotilla drivers |
| `admiral-index` | Index writes, query parser, subscriptions |
| `admiral-auth` | Sessions, local users, OIDC, GitHub, later LDAP, SAML, SCIM |
| `admiral-rbac` | Role templates, bindings, effective access, rendering to Kubernetes RBAC |
| `admiral-audit` | Audit events, hash chain, exporters |
| `admiral-compat` | Rancher API facade and data mapping |
| `admiral-migrate` | `admiral migrate rancher` |
| `admiral-api` | axum routes, OpenAPI, SSE |
| `admiral-server`, `admiral-agent`, `admiral-cli` | Binaries |
| `web/` | SolidJS UI |

## 4. Identity and access

### 4.1 Sign-in

| Method | Phase | Notes |
| --- | --- | --- |
| Local users | P1 | argon2id passwords, passkeys (WebAuthn), TOTP |
| OIDC | P1 | Any provider: Keycloak, Entra ID, Okta, Google, Dex |
| GitHub | P1 | Orgs and teams become groups |
| LDAP and Active Directory | P2 | Group sync, nested groups |
| SAML | P2 | Assertion replay protection from the start |
| SCIM | P2 | User and group provisioning |
| GitLab | P2 | Groups become groups |

Sessions are server-side with an idle timeout and an absolute limit. API tokens are scoped (global, cluster or project), have an expiry, and are stored as hashes.

### 4.2 Roles, bindings and projects

- Role templates at three levels, as in Rancher: global, cluster and project. This keeps migrated data one to one.
- Bindings attach a role template to a user or a group synced from a provider.
- The agent renders role templates into ClusterRole, Role and binding objects in the managed cluster, labelled as owned by admiral. It watches them; a manual edit is reported and repaired.
- A project is a set of namespaces with resource quotas and default limits. Namespaces join a project through the label `field.cattle.io/projectId`, the same label Rancher uses. A project can span several clusters, which Rancher cannot do. Projects imported from Rancher start in their original cluster.
- On flotilla the driver maps the same model onto flotilla's RBAC once it serves the RBAC kinds.

### 4.3 What admiral adds

- **Effective access.** Two questions answered from the UI, CLI and API: "who can do this verb on this resource, in every cluster" and "what can this person or group do". The answer combines admiral bindings with RBAC objects that already exist in each cluster.
- **Just-in-time access** (P2). Request a role for a set time, optionally with approval. The binding is removed when the time runs out.
- **No long-lived kubectl tokens.** `admiral login` is a kubectl exec credential plugin. It signs in with the OIDC device flow or the browser and hands kubectl short-lived tokens. Long-lived tokens remain for automation, always scoped and with an expiry.
- **Tamper-evident audit log.** Every request through the server is recorded with user, cluster, verb, resource and result. Each entry carries the hash of the one before it, so a deleted or edited entry breaks the chain, and `admiral audit verify` reports where. Export to OTLP, syslog and webhooks.

## 5. User interface

- **Stack.** TypeScript and SolidJS, Kobalte for accessible primitives, Tailwind for styling, CodeMirror 6 for YAML, xterm.js for shells. Fine-grained reactivity updates one table cell when one watch event arrives, instead of re-rendering the table.
- **Look.** An admiral design system with light and dark themes, a compact or comfortable density switch, English and Thai from the start, and WCAG 2.2 AA.
- **Fleet overview** is the home page: every cluster's health on one screen, including version drift, certificates close to expiry, unhealthy nodes, failing workloads and disconnected agents. Each card links to the cause.
- **Command palette (⌘K or Ctrl+K)** jumps to any resource in any cluster and runs actions: restart, scale, open logs, copy a kubeconfig.
- **Explorer.** Virtualized tables fed by server-side queries, live updates without reloads, the query language from 3.5, saved views, a URL for every view, and bulk actions across clusters.
- **Resource page.** YAML editing with schema validation and a diff before every apply. Events. A graph of owners and dependents (Deployment to ReplicaSet to Pod to Service and Ingress).
- **Logs and shell.** Stream logs from many pods in many clusters at once, with search and pause. Exec and node shells in the browser.
- **Keyboard first.** `j` and `k` to move, `/` to search, `g c` to go to clusters, and everything also works with a mouse.
- **Performance budget,** enforced in CI on every pull request: no more than 300 KB of gzipped JavaScript on first load, interactions answered within 100 ms, a 100,000-row table scrolling at 60 frames per second (benchmark B3).
- **Extensions** (P7) run in sandboxed iframes and reach only the API calls their manifest declares and an admin approved.

## 6. Moving from Rancher

### 6.1 How the move works

1. Install admiral next to Rancher.
2. `admiral migrate rancher plan --url https://rancher.example.com` reads Rancher through its API with an admin token and writes a report: what will move, what will not, and why.
3. `admiral migrate rancher apply` installs admiral-agent in each cluster next to cattle-cluster-agent and copies the data. It can run again without creating duplicates.
4. Each cluster starts in `observe` mode. Move it to `shared`, then `owned`, when you are ready:
   - `observe`: admiral reads only. Rancher keeps writing RBAC.
   - `shared`: admiral writes only objects it created.
   - `owned`: admiral manages the cluster and Rancher's objects can be removed.
5. Point the Rancher hostname at admiral. Kubeconfig files that use `/k8s/clusters/<id>` keep their URLs.
6. `admiral migrate rancher detach` removes cattle-cluster-agent and Rancher's objects from owned clusters.
7. `admiral migrate rancher rollback` removes everything admiral wrote, cluster by cluster.

### 6.2 Compatibility table

"Drop-in" means it works with no change by the user.

| Rancher item | admiral | Phase |
| --- | --- | --- |
| Clusters Rancher manages (imported, RKE2, k3s, hosted) | Drop-in. Imported with an agent, no rebuild | P1 |
| Projects (namespace label `field.cattle.io/projectId`) | Drop-in. Same label | P1 |
| Local users | Moved. Passwords carry over where Rancher's stored hash format can be verified, and are rehashed with argon2id at first sign-in. Other users get a reset link | P1 |
| Auth providers: GitHub, OIDC, Keycloak OIDC, Entra ID | Moved with their settings and secrets | P1 |
| Auth providers: LDAP, Active Directory, SAML | Moved | P2 |
| Global roles, role templates, bindings | Moved one to one | P1 |
| Kubeconfig URLs (`/k8s/clusters/<id>`) | Drop-in after the DNS change | P1 |
| API tokens and kubeconfig tokens | Not drop-in. Rancher stores token hashes, so users sign in again or download a new kubeconfig | P1 |
| Catalog repositories (ClusterRepo) | Moved | P2 |
| Installed Helm apps | Drop-in. They are Helm releases in the cluster and admiral reads them | P2 |
| Terraform provider `rancher2` | The resources and data sources the provider's own acceptance tests exercise, served by `admiral-compat`. The pass rate is published per release | P2 |
| Fleet GitRepos and repositories with `fleet.yaml` | GitRepos moved. Repositories work unchanged | P3 |
| Lifecycle of clusters Rancher provisioned (upgrade, node pools) | Adopted in P4. Until then they are managed but not upgraded by admiral | P4 |
| rancher-monitoring and rancher-logging | Detected and used as they are | P5 |
| rancher-backup data | Not compatible. admiral has its own backup | P1 |
| Rancher CLI | Commands people use most, measured from issue reports and docs | P7 |
| Rancher UI extensions | Not compatible. Different UI and a sandboxed model | Not planned |
| Steve `/v1` API used by the Rancher UI | Only calls external tools depend on | P7 |

The Rancher API is large. admiral implements the parts that tools call, measured by those tools' own test suites, and lists the rest here.

## 7. Registry integration

admiral talks to OCI registries through the distribution API, so it works with any registry. With the sibling Rust registry it also shows repositories, tags, signatures and scan results next to the workloads that run them (P6), and distributes pull secrets to projects (P2).

## 8. Feature matrix

Legend: **Yes** built in. **Partial** limited or needs extra parts. **No** not available. **n/a** does not apply to the product's design. The admiral column gives the phase. "Better" says what admiral does differently, or "Parity" when it only matches. Claims marked with a benchmark id (B1 to B8) are targets until section 11 has numbers.

Competitor marks reflect public documentation and release notes as of November 2026. Corrections are welcome as issues.

### 8.1 Install, footprint and upgrades

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Runs without a Kubernetes cluster for the manager | No | n/a (desktop) | Yes | Yes | No | P1 | No cluster to run, one binary with SQLite |
| High availability | Yes (3-node cluster) | n/a | No | n/a (stateless) | Yes | P1 | HA with Postgres replicas, no etcd to tune |
| Upgrade across several minor versions at once | No | Yes | Yes | Yes | Partial (EUS paths) | P1 | Tested one-step upgrades from the last four minors |
| Managed clusters need no inbound ports | Yes | No | Yes (Edge agent) | No | Yes | P1 | Parity |
| Manager memory and CPU at 1,000 clusters | Needs tuning | n/a | n/a | n/a | Needs tuning | P1 | Target lower than Rancher (B1) |
| Agent footprint per cluster | cattle-cluster-agent | n/a | Agent | n/a | klusterlet | P1 | Target smaller than cattle-cluster-agent (B5) |
| Time from install to first cluster visible | Minutes, after a cluster exists | Seconds | Minutes | Seconds | Hours | P1 | Target under 5 minutes from nothing (B7) |

### 8.2 Clusters

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Import any conformant cluster | Yes | Yes (kubeconfig) | Yes | Yes (kubeconfig) | Yes | P1 | Parity, plus Rancher-managed clusters as is |
| flotilla clusters | No | Partial (kubeconfig) | No | Partial (kubeconfig) | No | P1 | Only manager with flotilla lifecycle (P4) |
| Provision k3s or RKE2 on your machines | Yes | No | Partial | No | No | P4 | Preflight checks before every change |
| Create hosted EKS, AKS, GKE | Yes | No | Partial | No | Partial (OpenShift only) | P4 | Parity |
| Cluster API providers | Yes (Turtles) | No | No | No | Partial | P4 | Day-2 operations for every provider, not only RKE2 snapshots |
| Kubernetes upgrades from the UI | Yes | No | No | No | Yes (OpenShift) | P4 | Preflight report of deprecated APIs in use before the upgrade |
| etcd snapshot and restore | Yes (RKE2, k3s) | No | No | No | Partial | P4 | Parity |
| Certificate rotation | Yes | No | No | No | Yes | P4 | Expiry shown fleet-wide on the home page from P1 |
| Docker or Swarm hosts | No | No | Yes | No | No | P7 | Through a driver for the sibling container platform |

### 8.3 Explore and operate

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Search across all clusters | No | No | No | Partial | Yes | P1 | Query language, live results, saved views |
| Large lists stay fast | Partial | Yes | Partial | Yes | Yes | P1 | Server-side paging and live rows (B2, B3) |
| Logs from many pods and clusters at once | No | No | No | No | No | P1 | Better: one stream with search |
| Exec and shell in the browser | Yes | Yes | Yes | Yes | Yes | P1 | Parity |
| Diff before applying YAML | No | No | No | No | No | P1 | Better: every edit shows a diff |
| Owner and dependent graph | Partial | No | No | Yes | Yes (topology) | P1 | Graph crosses Services and Ingresses |
| Command palette over resources in every cluster | Partial | Partial | No | Partial | Partial | P1 | Jumps to instances, not only types |
| Kubeconfig through the manager | Yes | n/a | Yes | No | Partial | P1 | Short-lived tokens through `admiral login` |
| Fleet health overview | Partial | No | Partial | No | Yes | P1 | Version drift, expiring certs, agent status on one page |
| Bulk actions across clusters | No | No | No | No | Partial | P1 | Select in any list, act on all |
| Port forwarding | Partial | Yes | No | Yes | No | P1 | Through the tunnel, no direct access needed |

### 8.4 Identity and access

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Local users | Yes | n/a | Yes | No | Yes | P1 | Passkeys and TOTP built in |
| OIDC | Yes | n/a | Yes | Yes | Yes | P1 | Parity |
| GitHub | Yes | n/a | Yes | No | Yes | P1 | Parity |
| LDAP and Active Directory | Yes | n/a | Yes | No | Yes | P2 | Parity |
| SAML | Yes | n/a | No | No | No | P2 | Replay protection from the start |
| SCIM | Partial (preview) | No | No | No | No | P2 | Generally available |
| Global, cluster and project roles | Yes | No | Partial | No (cluster RBAC only) | Yes | P1 | Parity |
| Projects grouping namespaces | Yes | No | No | No | Yes (1 namespace each) | P1 | Projects across clusters |
| Who can do what, across clusters | No | No | No | No | No | P1 | Better: effective access queries |
| Time-limited access with approval | No | No | No | No | No | P2 | Better: built in |
| RBAC drift repair | Partial | No | No | No | Partial | P1 | Reports and repairs manual edits |
| Audit log | Yes | No | Yes | No | Yes | P1 | Hash chain with `admiral audit verify` |

### 8.5 Apps and GitOps

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Helm chart catalog | Yes | Yes | Yes | Yes | Yes | P2 | Parity |
| OCI chart registries | Yes | Partial | Partial | Partial | Partial | P2 | Parity |
| Diff before install or upgrade | No | No | No | No | No | P2 | Better: rendered manifest diff |
| Rollback | Yes | Yes | Partial | Yes | Yes | P2 | Parity |
| Install one app on many clusters | Yes (Fleet) | No | No | No | Yes | P2 | Pick clusters by query |
| Built-in GitOps | Yes (Fleet) | No | Yes | Partial (Flux plugin) | Yes (Argo CD) | P3 | State in a database, not etcd (B4) |
| Reads Fleet repositories unchanged | Yes | No | No | No | No | P3 | Parity with Rancher |
| Preview the diff per cluster before sync | No | No | No | No | Partial | P3 | Better: per-cluster preview |
| Rollout in waves with health gates | Partial | No | No | No | Partial | P3 | Waves stop on failed health checks |
| Git webhooks instead of polling | Yes | No | Yes | No | Yes | P3 | Parity |

### 8.6 Observability

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Install a monitoring stack | Yes | Partial | No | Partial | Yes | P5 | Uses an existing stack if one is there |
| Metrics on resource pages | Yes | Yes | Partial | Partial | Yes | P5 | Parity |
| Alerts | Yes | No | No | No | Yes | P5 | Fleet-wide alert view |
| Logging pipeline | Yes | No | No | No | Yes | P5 | Vector or Fluent Bit, OpenTelemetry |

### 8.7 Backup, policy and security

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Back up the manager | Yes | n/a | Yes | n/a | Yes | P1 | One command, automatic before upgrades |
| Back up workloads | No | No | No | No | Yes (OADP) | P6 | Through Velero, managed fleet-wide |
| Policy across clusters | Partial | No | No | No | Yes | P6 | Kyverno and Gatekeeper from one place |
| CIS benchmark scans | Yes | No | No | No | Yes | P6 | Parity |
| Image scan results next to workloads | Partial (NeuVector) | No | No | No | Partial (separate product) | P6 | From the sibling registry or any scanner |
| Pod security admission settings | Yes | No | No | No | Yes | P6 | Parity |

### 8.8 API and ecosystem

| Feature | Rancher | Lens | Portainer | Headlamp | OpenShift + ACM | admiral | Better |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Documented public API | Yes | No | Yes | No | Yes | P1 | OpenAPI 3.1, the UI uses only this API |
| CLI | Yes | No | No | No | Yes | P1 | Same query language as the UI |
| Terraform provider | Yes | No | Yes | No | No | P2 compat, P7 own | Existing `rancher2` configs keep working |
| UI extensions | Yes | Yes | No | Yes | Yes | P7 | Sandboxed |
| Translations | Yes | No | No | Yes | Yes | P1 | English and Thai first |
| Desktop app | No | Yes | No | Yes | No | Not planned | Web only |
| License | Apache-2.0 | Proprietary free tier | zlib (CE) and commercial | Apache-2.0 | Commercial (OKD is open) | Apache-2.0 | Parity with the open ones |

## 9. Out of scope, and why

- **Operating system management (like Elemental).** A different product with its own lifecycle. admiral provisions Kubernetes on machines you already have.
- **Virtualization (like Harvester).** Out of scope. A Harvester cluster can still be imported as a Kubernetes cluster.
- **A bundled service mesh.** Rancher deprecated its Istio integration in 2.12. Meshes install from the catalog like any app.
- **A desktop app.** The web UI covers the same ground, and one UI is easier to keep fast.

## 10. Phased plan

Each phase ends in a release. Each task becomes one issue in the GitHub Project and its phase's milestone.

### P1 MVP: see and control

1. Cargo workspace, CI on Linux, macOS and Windows, lint, release skeleton.
2. Store: sqlx over SQLite and Postgres, migrations, backup and restore, leader lock.
3. Tunnel: protocol, registration tokens, CA and client certificates, reconnect and replay.
4. `KubernetesDriver`: watches, index deltas, request proxy, impersonation.
5. Index: schema, query language parser, subscriptions over SSE.
6. API: routes, OpenAPI document, problem details, jobs.
7. Local users: argon2id, passkeys, TOTP, sessions, scoped tokens.
8. OIDC and GitHub sign-in with group sync.
9. RBAC: role templates, bindings, rendering in the agent, drift repair.
10. Projects: namespace label, quotas, default limits, projects across clusters.
11. Effective access queries.
12. Audit log with hash chain, `admiral audit verify`, OTLP, syslog and webhook export.
13. Kubeconfig proxy at `/k8s/clusters/<id>` and the `admiral login` credential plugin.
14. Logs, exec, shell and port-forward streams.
15. UI foundation: design system, themes, density, English and Thai, command palette.
16. UI: fleet overview, explorer, resource page with diff before apply, relationship graph.
17. UI: logs, shell, access and audit views.
18. CLI.
19. `admiral migrate rancher` plan, apply, rollback and detach, with ownership modes.
20. flotilla end-to-end suite.
21. Packaging: binaries, container images, Helm chart, install guide.
22. Benchmark harness and first published results for B1, B2, B3, B5 and B7.

### P2 Apps and identity

1. Helm and OCI catalogs, install, upgrade and rollback with a rendered diff.
2. Install one app on many clusters by query.
3. LDAP and Active Directory, SAML, SCIM, GitLab.
4. Just-in-time access with approvals.
5. Migration of catalogs and LDAP, AD and SAML providers from Rancher.
6. `admiral-compat` for the `rancher2` Terraform provider, with the acceptance test pass rate published (B8).
7. Pull secret distribution from registries to projects.

### P3 GitOps

1. GitOps engine that reads `fleet.yaml` and GitRepo semantics.
2. Webhooks with polling only as a fallback.
3. Per-cluster diff preview, drift detection, waves with health gates.
4. State in the database, sized for 1,000 clusters and 100,000 deployments (B4).
5. Fleet migration from Rancher.

### P4 Provisioning and lifecycle

1. k3s and RKE2 on existing machines.
2. Cluster API providers with day-2 operations.
3. Create and import EKS, AKS and GKE.
4. `FlotillaDriver` lifecycle.
5. Kubernetes upgrades with preflight reports, certificate rotation, etcd snapshots and restore.
6. Adopting clusters Rancher provisioned.

### P5 Observability

1. Monitoring stack install or detection, metrics on resource pages.
2. Alerts with a fleet-wide view.
3. Logging pipeline with Vector or Fluent Bit and OpenTelemetry.

### P6 Backup, policy and security

1. Workload backup through Velero.
2. Kyverno and Gatekeeper policy across clusters.
3. CIS benchmark scans and compliance reports.
4. Image scan results and signatures from the sibling registry and other scanners.
5. Pod security admission settings per project.

### P7 Ecosystem

1. admiral's own Terraform provider.
2. Extension SDK with sandboxed iframes.
3. Edge scale: several thousand clusters.
4. Remaining `/v3` and `/v1` calls that tools use, and the common Rancher CLI commands.
5. Driver for the sibling container platform (Docker and Swarm style hosts).

## 11. Testing and benchmarks

### 11.1 Tests

- Tests come first with every change. Table-driven where there is a table. Unit tests use no network.
- CI runs unit tests on Linux, macOS and Windows. End-to-end suites run on Linux against real clusters.
- End-to-end suites run against k3d and kind clusters, and a flotilla suite runs against flotilla.
- **Migration suite.** Install a real Rancher in k3d, seed users, roles, projects and catalogs, run `admiral migrate rancher plan` and `apply`, then check with effective access that every user has exactly the access they had before. Run rollback and check the cluster is as it was.
- **Terraform compatibility.** Run the `rancher2` provider's acceptance tests against admiral and publish the pass rate.
- **UI.** Playwright for flows, axe for accessibility, and the performance budget from section 5.

### 11.2 Benchmarks

All benchmarks use kwok to simulate clusters and nodes, run Rancher and admiral on the same hardware, live in the repository so anyone can rerun them, and are published every release, including results where admiral loses.

| Id | What is measured | Target |
| --- | --- | --- |
| B1 | Manager memory and CPU at 10, 100 and 1,000 clusters | Lower than Rancher at every size |
| B2 | List and search latency at p50 and p99, 100,000 pods across 500 clusters | First page under 200 ms at p99 |
| B3 | UI time to interactive and scrolling of a 100,000-row table | Interactive under 1.5 s, 60 frames per second |
| B4 | GitOps at 1,000 clusters and 100 bundles | No storage tuning needed, status reported for every cluster |
| B5 | Agent memory and CPU per cluster | Smaller than cattle-cluster-agent |
| B6 | Upgrade time across four minor versions | Under 5 minutes for a single node |
| B7 | Time from nothing to first cluster visible | Under 5 minutes |
| B8 | `rancher2` acceptance tests passing against admiral | Published per release, 100% of covered resources |

## 12. Failure handling

- **Agent disconnected.** The UI keeps the last known state, marked stale with the time it was last seen. The agent reconnects with backoff and replays only what changed.
- **Server down.** Clusters keep running, since admiral is not in their data path. A break-glass kubeconfig, available to admins only and recorded when used, gives direct access.
- **Migration fails halfway.** `apply` runs again without creating duplicates, preflight checks run before any write, and `rollback` removes what admiral wrote.
- **Upgrade fails.** The server backs up before migrating and can restore that backup and start the previous version.
- **Database unavailable.** The API answers 503 with a problem detail. Agents keep their tunnels and buffer index deltas up to a limit, then relist when the server comes back.

## 13. Security

- Credentials for providers and clusters are encrypted at rest with a key from the environment or a KMS.
- Strict Content Security Policy, SameSite cookies and CSRF tokens in the UI.
- Text from clusters is escaped before it reaches the browser or a terminal.
- Agents hold only the permissions they need, listed in the Helm chart and the install manifest.
- Releases are signed and come with an SBOM and provenance.
- `SECURITY.md` explains how to report a vulnerability privately.
