# Contributing to admiral

Thanks for wanting to help. Bug reports, ideas, docs fixes and code are all welcome.

## Before you start

- Read the [design](docs/design.md). The phased plan in section 10 is the roadmap, and every task in it is an issue in the [project board](https://github.com/users/raven-clown/projects) and in its phase's milestone.
- Look through the [open issues](https://github.com/raven-clown/admiral/issues). Issues labelled `good first issue` are small and well scoped, and `help wanted` marks work nobody has picked up yet.
- For anything bigger than a small fix, open an issue or a discussion before writing code, so we agree on the approach first.
- Say on the issue that you are working on it, so two people do not do the same work.
- Questions go to [Discussions](https://github.com/raven-clown/admiral/discussions), not issues.
- Security problems never go in a public issue. See [SECURITY.md](SECURITY.md).

## Set up

There is no code yet. Once the first phase starts, you will need:

- Rust stable (the version is pinned in `rust-toolchain.toml`)
- Node.js LTS and pnpm for the web UI in `web/`
- Docker with k3d or kind for the end-to-end suites

```shell
git clone https://github.com/raven-clown/admiral
cd admiral
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all --check
```

[AGENTS.md](AGENTS.md) lists the layout and the rules below in more detail, and is kept current as the code lands.

## Rules for code

- Write the test first. Table-driven where there is a table. Unit tests use no network and no real cluster.
- Every test must pass on Windows, macOS and Linux.
- Credentials, tokens and kubeconfig contents never appear in logs, errors, fixtures or issues.
- Text from clusters is escaped before it reaches a browser or a terminal.
- The UI uses only the public API. If the UI needs something, the API gets it first.
- Keep comments short and about why, not what.
- Every claim of being faster or lighter than another tool needs a benchmark in `bench/` that anyone can rerun.

## Pull requests

1. Fork the repository and create a branch named `feat/<topic>`, `fix/<topic>`, `docs/<topic>`, `ci/<topic>` or `chore/<topic>`.
2. Use [Conventional Commits](https://www.conventionalcommits.org/) messages, for example `feat(index): add label filters to the query language`.
3. Add a line to `CHANGELOG.md` under Unreleased for anything users will notice.
4. Keep one topic per pull request. Small pull requests get reviewed faster.
5. Fill in the pull request template and link the issue with `Closes #123`.

CI runs the tests on Linux, macOS and Windows, plus lint. Workflows from outside contributors wait for a maintainer to approve them, so a pending check right after you open the pull request is normal.

Pull requests are squash-merged, so the title becomes the commit on main. Please write it as a Conventional Commit too.

## Code of conduct

Everyone taking part agrees to the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

By contributing you agree that your work is licensed under the [Apache License 2.0](LICENSE), the same as the rest of the project.
