# AGENTS.md

Instructions for coding agents working in this repository. Keep this file current: when a command, the layout or a rule changes, update it in the same commit.

## Project

admiral is a multi-cluster management platform for Kubernetes, written in Rust, built as a drop-in replacement for Rancher. The design is in `docs/design.md`; its section 10 is the phased plan, and each task there is one issue in the GitHub Project. Implementation plans per phase are in `docs/plans/`, starting with `docs/plans/p1-mvp-see-and-control.md`.

There is no code yet. Phase 1, task 1 creates the Cargo workspace; update the commands and layout below when it lands.

## Commands

```shell
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all --check
```

CI runs tests on Linux, macOS and Windows, and lint on Linux. The jobs skip their Rust steps until `Cargo.toml` exists.

## Layout

Planned, from section 3.7 of the design:

- `crates/admiral-core`: domain types and errors, no I/O.
- `crates/admiral-store`: sqlx over SQLite and Postgres, migrations, backup.
- `crates/admiral-tunnel`: agent and server ends of the mTLS gRPC tunnel.
- `crates/admiral-driver`: the `ClusterDriver` trait, Kubernetes and flotilla drivers.
- `crates/admiral-index`: index writes, query language, subscriptions.
- `crates/admiral-auth`, `crates/admiral-rbac`, `crates/admiral-audit`: sign-in, access, audit.
- `crates/admiral-compat`, `crates/admiral-migrate`: Rancher API facade and `admiral migrate rancher`.
- `crates/admiral-api`: axum routes, OpenAPI, SSE.
- `crates/admiral-server`, `crates/admiral-agent`, `crates/admiral-cli`: binaries.
- `web/`: the SolidJS UI.
- `bench/`: benchmark harness and results.

## Rules

- Tests with every change, written first. Table-driven where there is a table. No network and no real cluster in unit tests. No bare sleeps.
- Every test passes on Windows, macOS and Linux.
- Library crates never print and never touch stdout.
- Credentials, tokens and kubeconfig contents never appear in logs, errors or fixtures.
- Text from clusters is escaped before it reaches a browser or a terminal.
- The UI uses only the public API.
- Work on a branch named `feat/<topic>`, `fix/<topic>`, `ci/<topic>`, `docs/<topic>` or `chore/<topic>`. Commit with Conventional Commits messages.
- Merge a squash with `--subject "<pr title> (#<n>)" --body ""` so no co-author trailer lands on main.
- Every pull request gets labels, a milestone and an assignee.
- When a Cargo or npm manifest is added, add its ecosystem to `.github/dependabot.yml` in the same pull request.
- Never mention AI, assistants or code generation in code, comments, docs, commit messages, pull requests or issues. Never add `Co-Authored-By`, `Generated with` or similar trailers.
- No em-dashes. Keep comments short and about why.
- Update CHANGELOG.md under Unreleased for anything users will notice.
