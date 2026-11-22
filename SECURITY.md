# Security policy

## Supported versions

There is no release yet. Once there is, fixes go into the latest release and the one before it.

## Reporting a vulnerability

Do not open a public issue. Report it privately through GitHub: open the [Security tab](https://github.com/raven-clown/admiral/security), choose "Report a vulnerability", and include:

- what an attacker can do, and what they need first
- steps or a small proof of concept
- the admiral version, how it was installed, and the Kubernetes distribution and version of the clusters involved

Never include real tokens, kubeconfig files or provider secrets. Redact them from any logs you attach.

You should get a first answer within 7 days. Once a fix is ready we publish an advisory and a release, and credit you unless you ask us not to.

## What counts

admiral holds credentials for many clusters and decides who may reach them, so these matter most:

- a user reaching a cluster, project, namespace or verb their bindings do not allow
- tokens, client certificates, provider secrets or kubeconfig contents leaking into output, logs, errors, files or other hosts
- an agent accepting commands from anything other than its own server, or a server accepting an agent it did not register
- text from a cluster running script in the UI or changing a terminal through control sequences
- audit entries that can be removed or changed without `admiral audit verify` noticing
- an extension reaching an API call its manifest did not declare
