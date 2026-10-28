# Glossary (unified)

Defined by MF-FED-001; adapters MUST map native terms onto these. The bare word **node** MUST NOT be used unqualified (see inconsistencies.md G-01).

| Term | Meaning | Native equivalents |
|---|---|---|
| host | One OS instance: bare metal, VM or phone | Virtu/libvirt *Node*; familia *host* |
| guest | A VM or container running on a host | libvirt *Domain*; Rakazo *Computer* |
| service | A long-running process: model server, bot, agent, API endpoint | familia *node*; Rakazo *bot*/backend; Virtu WUI |
| edge | A declared flow between services | familia graph edge |
| trust-domain | A set of principals under one policy. Never "domain" alone | Rakazo *Space* |
| partition | A trust-domain plus its authority (KDC/directory or WoT keyring) | — |
| realm | A Kerberos realm | — |
| label | `(integrity, confidentiality, realm-path, tagset)` on the dominance lattice (MF-FED-030) | — |
| probe | A read-only measurement of a host (MF-LOC-010) | familia monitor probe |
| enforcement tier | `hard` (kernel-enforced), `contained` (container/VM), `advisory` (logged only) (MF-LOC-031, MF-FED-034) | — |
| owner | The human principal (Brian) | — |
| admin | Whoever operates a partition; may or may not be the owner | — |
| host.local.yaml | Never-committed per-host overlay for real addresses, accounts, secrets (MF-FED-002, MF-LOC-002) | familia `hosts.local.yaml` |
