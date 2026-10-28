# Discussion summary (federation vs local-host perspectives)

Two reviewers drafted the requirements on 2026-10-28, one from the federation perspective (MF-FED) and one from the single-machine perspective (MF-LOC), both prioritizing user and administrator experience. They reconciled their drafts in three rounds.

## Boundary
- **Federation** owns everything crossing a host or trust boundary: identity and key lifecycle, signed graph and consent records, discovery, partitions (Kerberos / WoT / hybrid), label syntax and transport, BitTorrent distribution, schema versioning, topology change, cross-host recovery, federated audit.
- **Local host** owns the single machine: probes, resource bounds, local enforcement binding (DAC, MAC, Capsicum, Landlock, eBPF, namespaces), install/admin UX, offline operation.
- **Seam:** the label. Federation defines syntax, ordering and signing; the host enforces. Federated decisions can only narrow local policy (MF-FED-033 = MF-LOC-032).

## Agreed
1. One `req.yaml` per project with `federation:` and `local:` sections and one schema version; real host facts only in a never-committed `host.local.yaml` (MF-FED-002, MF-LOC-002).
2. A partitioned host keeps running its last signed graph; local service never expires, only cross-host edges do (MF-FED-060, MF-LOC-050).
3. `on_low_read: demote | deny`, default demote; `deny` needs a hard enforcement tier; non-enforcing hosts are `advisory` and get high-integrity data only with an owner-signed waiver (MF-FED-031, 034, MF-LOC-033).
4. Unified glossary; bare "node" banned (MF-FED-001).
5. Ports allocated locally, advertised federally (MF-LOC-062, MF-FED-051); pinned per-partition known_hosts (MF-LOC-037).
6. A shared guest (Rakazo Team Computer) carries one label, the meet of its users (MF-LOC-038).
7. Virtu cordon/evacuate is the reference recovery primitive; hosts publish capacity and a draining state (MF-FED-062, MF-LOC-065).
8. Every inconsistency finding cites a source and names its resolving requirement.

## Resolved disagreement
Federation held "no always-on coordinator"; local-host agreed but required file-based, append-only per-host state. Outcome: MF-LOC-054 is the audit source of truth; federation only reads, checkpoints and witnesses it (MF-FED-090, 110).

## Also folded in
- familia ROADMAP.md is untrusted evidence (G-09, G-17); reverse-synthesis uses only code, tests and resolvable issues.
- Self-modifying harness PRs must never edit gate/policy files without an owner-signed diff (MF-FED-112, MF-LOC-034).

## Left for Brian
Who approves persistent installs on unattended hosts; see open-questions.md Q1.
