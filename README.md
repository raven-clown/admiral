# admiral

admiral is a multi-cluster management platform for Kubernetes, written in Rust. It aims to be a drop-in replacement for Rancher: you point it at the clusters you already run, bring over your users, roles and projects, and keep working without rebuilding anything.

> **Status: design in progress.** There is no code to run yet. The design is in [docs/design.md](docs/design.md). Nothing below works today; it is the scope the project is planned around.

## Why

Rancher is the most complete open-source cluster manager, and the most common complaints about it are about weight and speed:

- It needs a whole Kubernetes cluster just to run the manager.
- The UI slows down as the number of workloads grows.
- Upgrades have to go one minor version at a time.

admiral keeps the feature set and changes the foundation.

## Planned design

- **One binary.** The server runs anywhere: on a laptop with SQLite, or highly available with Postgres. It does not need a Kubernetes cluster of its own.
- **Light agent.** A small agent in each managed cluster connects out to the server, so managed clusters open no inbound ports.
- **Fast at scale.** Listing, search and filtering across every cluster happen on the server, from an index the agents keep up to date, with live updates streamed to the UI.
- **Drop-in for Rancher.** Import Rancher-managed clusters as they are, run side by side during the move, and migrate users, auth providers, roles, projects, catalogs and Fleet repositories with `admiral migrate rancher`. A compatibility layer keeps kubeconfig paths and the API calls used by the Rancher Terraform provider working.
- **Any Kubernetes.** Standard clusters (RKE2, k3s, EKS, AKS, GKE and others) and flotilla, a Rust orchestrator that serves the Kubernetes API.

## Planned scope

- Cluster import and provisioning
- Central auth (local, OIDC, GitHub, LDAP, SAML, SCIM) and RBAC across clusters, projects and namespaces
- Cross-cluster resource explorer, logs, exec and shell
- App catalog with Helm and OCI charts
- GitOps that reads existing Fleet repositories
- Monitoring, logging, backups and policy
- Web UI, CLI and a public REST API

The phased plan, the feature comparison with Rancher, Lens, Portainer, Headlamp and the OpenShift console, and the benchmarks behind each "better" claim are in the design.

## Contributing

The contributor guide and issue templates will follow once the design is approved. Until then, ideas and questions are welcome in issues.

## License

[Apache-2.0](LICENSE)
