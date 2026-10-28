# MeshForge: federation requirements 

Status: DRAFT, requirements only. Nothing here is implemented, and nothing has been filed on GitHub.
Perspective: federation, meaning anything that crosses a host, network or trust boundary. The local-host requirements are the local-host (`local-host.md`, MF-LOC-###). The agreed seam is in `../docs/discussion.md`.
Sources: X thread review (MeshForge thread of Oct 25, 08:58–09:47 PT); familia (state as of 2026-10-28): DESIGN.md, REQUIREMENTS.md, ROADMAP.md, docs/known-issues.md, docs/transport-poc.md, docs/hosts-overlay.md, open issues #71 #97 #120 #147 #150 #151, design PRs #163/#165/#166; Virtu: gitlab.com/renich/virtu (README, TODO.rst, cluster.example.yaml; 0 open issues); Rakazo: github.com/elie222/rakazo (README, VISION.md, docs index, open issues).
Keywords MUST/SHOULD/MAY follow RFC 2119. "Owner" means the human principal (Brian). "Admin" means whoever operates a partition. They may be the same person, but the UX must work when they are not.

## 0. Design lineage (normative intent)
MeshForge's security model is deliberately a **distributed restatement of existing OS security work**, not a new invention:
- **DARPA CHATS / CBOSS**: composable, high-assurance trusted OS services; policy is a pluggable module, not hard-wired.
- **TrustedBSD MAC Framework (FreeBSD)**: the hook-and-label architecture. Policies attach labels to subjects and objects and decide at hooks. MeshForge treats each project adapter as a "policy module", and each cross-host edge as a "hook".
- **Tim Fraser's LOMAC** (low-water-mark integrity; Brian ported it to FreeBSD): a subject that reads lower-integrity data is *demoted* to the meet of the two labels, which is taint that floats downward. Writes are allowed only when the subject's label dominates the object's.
- **OS primitives as the enforcement vocabulary**: DAC ≈ owner consent, MAC (SELinux / AppArmor / TrustedBSD MAC) ≈ domain policy, Capsicum-style capabilities ≈ unforgeable delegation tickets. These are enforced locally (Landlock, LSM, eBPF, namespaces, setcap; see MF-LOC).
- **Old, proven protocols are first class**: OpenPGP (RFC 9580) for signing, Kerberos 5 plus a directory (LDAP) for centrally run partitions, the OpenPGP web of trust for decentralized ones, and BitTorrent (BEP 3/9/27, private trackers) for distribution.

## 1. Glossary and the shape of req.yaml
Unified glossary: see `glossary.md`.

**MF-FED-001 (MUST) Shared glossary.** MeshForge documents MUST use: *host* (a physical or virtual machine running an OS), *guest* (a VM or container on a host), *service* (a model server, bot, agent or API endpoint), *trust-domain* (a set of principals under one policy), *partition* (a trust-domain plus its authority: a KDC/directory or a WoT keyring), *realm* (a Kerberos realm), and *edge* (a declared flow between services). The bare word "node" MUST NOT be used without qualification. Adapters MUST map their native terms onto these: familia `node` → service and `host` → host; Virtu/libvirt `Node` → host and `Domain` → guest; Rakazo `Space` → trust-domain, `Computer` → guest and `bot` → service.
- Rationale: "node" means three different things in the three projects, and "domain" means two (see inconsistencies.md G-01).
- UX/admin: error messages print the native term and the MeshForge term, e.g. `Virtu Domain "web1" (guest)`.
- Acceptance: a lint over the requirements repo flags any unqualified "node". Each adapter spec contains a mapping table.

**MF-FED-002 (MUST) One req.yaml, two sections, one version.** Each project publishes one `req.yaml` with `schema: meshforge/v<major>.<minor>`, a `federation:` section (this document) and a `local:` section (MF-LOC). Real addresses, users and secrets MUST live only in a never-committed `host.local.yaml` overlay. This generalizes familia's `hosts.local.yaml` and its `REPLACE_ME` placeholders.
- Acceptance: a req.yaml containing an IP literal, a username or a secret-shaped string fails validation with a pointer to the overlay.

```yaml
# example only
schema: meshforge/v0.1
project: familia
federation:
  identity: {kind: openpgp, fingerprint_ref: overlay:project_key}
  partition_modes: [wot, kerberos, hybrid]
  provides:
    - {service: coder, iface: http+openapi, spec: openapi/llama-server.yaml, label: "integrity:lan/fleet"}
  requires:
    - {iface: artifact-swarm, proto: bittorrent-private}
  consent: {default: deny, records: signed}
  audit: {sink: append-only, signed: true}
local: {}   # owned by worker B
```

## 2. Identity and signing

**MF-FED-010 (MUST) Every principal has a key.** Owners, admins, hosts, project instances (one Familia install, one Virtu cluster, one Rakazo deployment) and long-lived services MUST each have an OpenPGP identity (a primary key plus subkeys separated by usage: certify / sign / auth). Host keys MUST be bound to the SSH host key, through a signed statement or by OpenPGP-authentication-subkey derivation.
- Rationale: today familia trusts SSH TOFU (`StrictHostKeyChecking=accept-new`), Virtu trusts libvirt URIs, and Rakazo trusts a DB account plus Better Auth sessions. None of them can prove "this graph came from Brian".
- UX: `meshforge id init` creates a key with one confirmation, prints the fingerprint as words plus hex, and offers a paper or QR backup. Admin: keys can live on a smartcard or YubiKey. Signing subkeys are rotated without re-certifying peers.
- Acceptance: a fresh host joins a test partition and every artifact it emits verifies against a key chained to the owner's key.

**MF-FED-011 (MUST) Signed graph.** The joint graph (all req.yaml files plus the solver output) MUST be canonicalized (sorted keys, normalized YAML → JSON → RFC 8785 JCS) and signed by the owner's key, or by an admin key delegated for that partition. Hosts MUST refuse to apply an unsigned graph or one with a bad signature. In that case they keep running the last good graph (MF-FED-060).
- Acceptance: flipping one byte of a graph makes every host reject it, logging the signer, the expected fingerprint and the diff path.

**MF-FED-012 (MUST) Key lifecycle.** Revocation, expiry and rotation MUST propagate through every partition mode. In WoT mode this uses revocation certificates distributed with graph updates. In Kerberos mode it uses directory removal plus KDC principal disable. Hybrid mode does both. A revoked key's past signatures stay verifiable "as of" their timestamp, but its new signatures are rejected.
- UX: `meshforge id revoke` shows a list of exactly which edges and hosts lose access before the owner confirms.

**MF-FED-013 (SHOULD) Multiple reference verifiers.** At least two independent verifier implementations, in different languages (e.g. Python with sequoia/pgpy, and Go or Crystal), SHOULD pass a shared signature test vector suite. This is the "multiple reference implementations" requirement from the thread, applied where bugs hurt most.

## 3. Partitions and trust between networks

**MF-FED-020 (MUST) Declared partition mode.** Every trust-domain MUST declare `mode: kerberos | wot | hybrid`.
- *kerberos*: a realm, a KDC and a directory (LDAP/FreeIPA/Samba AD). Principals are `svc/host@REALM`, and group membership comes from the directory.
- *wot*: OpenPGP certifications with a trust threshold (default: certified directly by the owner, or by 2 admins with marginal trust).
- *hybrid*: the Kerberos realm is authoritative inside it, and the WoT bridges between realms. A cross-realm trust MUST be a signed bridge record, never implicit Kerberos cross-realm trust alone.
- UX: the home LAN defaults to `wot` with the owner as the only certifier, so there is zero infrastructure to set up. A KDC is opt-in for labs and organizations.
- Acceptance: the same two-host graph validates in all three modes. Switching modes requires only partition config changes, never service changes.

**MF-FED-021 (MUST) Hierarchical partitions.** Partitions MAY nest (fleet ⊃ lan ⊃ phones). A child inherits its parent's ceiling labels and MAY only narrow them. The solver MAY approximate across partitions (solving each partition, then the inter-partition edges) and MUST report when the result is approximate.

**MF-FED-022 (MUST) Bridge records.** Every edge crossing a partition boundary needs a signed bridge record: endpoints, interface, label ceiling, expiry, and the signers from both sides. With no bridge record the edge doesn't exist, even if the network allows it.
- Admin: `meshforge bridges` lists every cross-partition edge, its expiry and who signed it, in one screen.

## 4. Labels: the dominance lattice

**MF-FED-030 (MUST) Label syntax and order.** A label is a tuple `(integrity, confidentiality, realm-path, tagset)`. Ordering is the product order: `L1 ≥ L2` (L1 dominates L2) iff each component dominates. A realm-path dominates its descendants, so `fleet ≥ fleet/lan ≥ fleet/lan/phones`. A tagset dominates by inclusion (hashtag sets, as Brian proposed). Meet (⊓) and join (⊔) MUST be defined component-wise, and the lattice MUST have explicit ⊤ and ⊥.
- Acceptance: a property-test suite checks that the order is a partial order, and that meet and join are associative, commutative and idempotent, over generated labels, in both reference implementations.

**MF-FED-031 (MUST) LOMAC semantics across hosts.**
- *Read*: when a service reads an object (a message, KV file, model, prompt or tool result) labeled `Lo`, the service's integrity becomes `Ls ⊓ Lo`. This happens on the reading host.
- *Write*: a service may write to an object or edge only if `Ls ≥ Lo_target`.
- Labels MUST travel with data, as signed label headers on messages and fields in the artifact manifest. They MUST NOT be inferred from the network path.
- Per-service `on_low_read: demote | deny` (default demote, agreed with B). Under `deny`, the host blocks the read *before* it happens (MF-LOC enforcement).
- UX: a demotion is shown as an event ("coder demoted to integrity:untrusted after reading web fetch at 05:12"). Restoring a label is an explicit, owner-signed action, never automatic. This is the federated analogue of Grok's Laya point: hard gates live outside the agent loop.

**MF-FED-032 (MUST) Capabilities as delegation tickets.** A cross-host authorization MUST be a capability: a signed, narrow, expiring ticket naming the holder, the object, the allowed operations and a label ceiling. It can be attenuated (narrowed) when delegated, and never amplified. Kerberos service tickets MAY carry it (authorization data), and so may OpenPGP-signed tokens.

**MF-FED-033 (MUST) Federated decisions only narrow.** Effective permission = local policy ∩ federated grant (B's MF-LOC-032). No federated record can grant what the host's local policy denies.

**MF-FED-034 (MUST) Advisory-enforcement hosts** (↔ MF-LOC-033). Hosts that can't enforce labels (no root on Termux, Windows without equivalent hooks) are marked `enforcement: advisory` by the solver. Data at or above a configurable integrity or confidentiality threshold MUST NOT be routed to them, unless an owner-signed waiver exists naming the host, the label and an expiry.

## 5. Consent

**MF-FED-040 (MUST) Default deny, recorded consent.** No edge crossing a host boundary exists without a consent record: who, what edge, which label ceiling, an expiry, and the signature. This formalizes familia's existing rule ("never opens a cross-host edge without recorded consent", REQUIREMENTS.md escalation) and Rakazo's approval modes.
- UX: consent prompts are batched per topology change ("this change opens 3 edges: …, approve all / pick"), never one prompt per packet. They are shown on whatever surface the owner is on (CLI, familia panel, Rakazo app), and all of them write the same signed record.
- Acceptance: deleting the consent record closes the edge within one reconcile interval on every host.

**MF-FED-041 (MUST) Consent separate from admin.** In line with Rakazo's VISION ("app accounts never administer the server"), being an operator or admin MUST NOT imply consent on behalf of the owner's data. These are distinct signer roles.

**MF-FED-042 (MUST) Owner approval for persistent installs on unattended hosts.** Owner approval MUST be obtained, personally from the owner (Brian), for any persistent install (service unit, rc.d/cron/Termux:Boot entry, rc.conf change) on an unattended host such as a phone or headless FreeBSD box. The host's local approval (MF-LOC-045) and a signed federated approval (MF-FED-040) MAY still be required in addition, but neither substitutes for owner approval, and admin or federation signatures MUST NOT be accepted in its place (MF-FED-041). Resolves open question Q1 (2026-10-28).

## 6. Discovery

**MF-FED-050 (MUST) Discovery is advisory, the graph is authoritative.** Hosts MAY be found by mDNS/DNS-SD, ARP, or a directory lookup. A discovered host is only a *candidate* until it's in the signed graph. Discovery MUST NOT open edges.
- Rationale: in today's fleet, a FreeBSD host's address dropped, and the agents guessed. Discovery should help, but it must not decide.
- UX: `meshforge discover` lists candidates with a confidence level and its reason (e.g. "host key matches graph host-b, address changed"). One command proposes a signed graph update.

**MF-FED-051 (MUST) Endpoint advertisement** (pinned per-partition known_hosts: MF-LOC-037; key change = hard fail + admin alert). Services advertise `(service, iface, host-key fingerprint, port, label)` as a signed record. Port allocation stays local (agreed with B, MF-LOC). Clients MUST verify the host key against the record, not TOFU.

**MF-FED-052 (SHOULD) Address drift.** The identity of a host is its key, not its IP. A graph lists addresses only as hints in the overlay. When an address changes, the host re-advertises, and reachability checks report "host key seen at new address".

## 7. Topology change, partitions and fault recovery

**MF-FED-060 (MUST) Last-good graph.** A host that is partitioned or offline keeps running its last valid signed graph. Local-only edges keep working indefinitely. Cross-host edges whose capabilities expire stay closed until re-verified (agreed with B: no TTL may stop local service).
- UX: status says plainly "partitioned since 05:20 PDT, running graph sig 3F2A…, 2 cross-host edges suspended".

**MF-FED-061 (MUST) Topology change as a signed diff.** Adding or removing hosts, services or edges MUST be expressed as a diff against the current graph's hash, with a `prev` pointer forming a hash chain. Hosts reject a diff whose `prev` doesn't match their current graph and report the fork.

**MF-FED-062 (MUST) Recovery primitives are contracts** (↔ MF-LOC-065 `draining` state). Each project MUST expose, in its req.yaml, the recovery verbs the federation may invoke. Proposed minimal set: `drain`, `cordon`, `evacuate`, `restart`, `snapshot`, `restore`. Virtu: cordon and evacuate are on its TODO, unimplemented. familia: the watchdog and reconnect-selfheal. Rakazo: computer suspend and restore with generation fencing.
- Acceptance: a fault-injection test that kills one host in a 3-host graph produces a solver plan using only the declared verbs, which the owner approves.

**MF-FED-063 (MUST) Monitoring is a federation contract, not a command.** Every project MUST declare a `health` interface (pull endpoint or push topic, schema, interval) and its supervision (MF-LOC: who starts it at boot). A service with no supervised health source is invalid.
- Rationale: lesson of Oct 28. familia's monitor (#71) only ran when Brian asked, because no requirement said it must be supervised, and the FreeBSD probe returned empty values with no error.
- Acceptance: the validator rejects a graph where any host lacks a supervised health source. An empty probe result counts as a failure, not a pass.

**MF-FED-064 (SHOULD) Fencing.** Shared mutable state (a Rakazo Team Computer profile, the familia KV store) MUST use generation fencing across hosts, as Rakazo already does locally, so a stale replica can't overwrite newer state after a partition heals.

## 8. Swarm distribution (BitTorrent)

**MF-FED-070 (MUST) Content addressing plus signed manifests.** Models, KV snapshots and datasets are distributed as content-addressed objects. A manifest `{name, sha256, size, btih (v2 where available), label, license, signer}` is OpenPGP-signed. A magnet link alone is never a trust root. One canonical key: `sha256` is primary, and `btih` and the path-in-torrent are locators (see inconsistencies.md G-04).

**MF-FED-071 (MUST) Private by default.** Trackers MUST be private (BEP 27, no DHT or PEX for labeled data) and bound to a LAN address. This repeats the familia transport-PoC validator rules as MeshForge requirements. Public swarms MAY be used only for objects labeled `confidentiality:public` with a license permitting redistribution.

**MF-FED-072 (MUST) Label check on fetch.** A host fetching an object MUST apply MF-FED-031 at fetch time. A demoting fetch triggers the demote or deny rule *before* the object is loaded into a model.

**MF-FED-073 (SHOULD) Bandwidth and metered links.** Partitions SHOULD declare metered or cellular links (the phones), and the swarm SHOULD prefer LAN seeds. Admin UX: one view showing what each host is seeding, its size and its label.

## 9. Schema versioning

**MF-FED-080 (MUST) Semver schemas plus negotiation.** `meshforge/vMAJOR.MINOR`. Minor versions are additive only. Each interface declares `iface_version`. Peers negotiate the highest common version, and the solver flags edges with no common version as conflicts, *before* deployment.
**MF-FED-081 (MUST) Vocabulary alignment.** Model and service metadata fields MUST map to Hugging Face metadata where one exists (familia #120's "hard requirement"), and host/guest fields to libvirt terms. The mapping tables are versioned with the schema.
**MF-FED-082 (MUST) Compatibility window.** A host MUST accept graphs for at least the current and previous minor version, and the validator prints migration hints.

## 10. Audit

**MF-FED-090 (MUST) Append-only, signed, hash-chained audit.** Source of truth is each host's local file-based state/audit log (MF-LOC-054); federation only reads, checkpoints and witnesses it. Every consent, label change (demotion and restoration), graph apply, bridge, capability issue and recovery action is logged per host in an append-only hash chain, with periodic signed checkpoints. Checkpoints are cross-witnessed by at least one other host in the partition, when reachable.
**MF-FED-091 (MUST) Privacy-labeled audit.** Audit entries carry labels too. An audit log never contains payloads above its own label: it uses hashes and references instead.
**MF-FED-092 (SHOULD) Human-readable digest.** Admins SHOULD get a daily digest of what changed across the federation and who signed it, readable without tools.

## 11. Adapters and joint-graph testing

**MF-FED-100 (MUST) Adapter contract per project.** Each adapter maps native objects to MeshForge (MF-FED-001) and declares: identity source, partition modes supported, health interface, recovery verbs and label enforcement capability. Adapters for Familia, Virtu and Rakazo are the first three.
**MF-FED-101 (MUST) Joint-graph conformance suite.** It runs offline and deterministically (Rakazo's VISION already demands offline conformance tests), and covers: the signature vectors, lattice properties, a 3-project sample graph, fault injection and a partition-heal fork test.
**MF-FED-102 (MUST) Reverse-synthesis lab test.** MeshForge's first proof is to derive familia's requirements from MeshForge's and diff them against familia's REQUIREMENTS.md. Every difference becomes either a MeshForge gap or a familia gap (Brian's proving ground).

## 12. Cross-cutting requirements (raised by the inconsistency review)

**MF-FED-110 (MUST) No required coordinator.** Validating, signing and applying a graph MUST work with only files, keys and peer-to-peer copy (git, rsync, sneakernet, BitTorrent). Project servers (Rakazo Postgres, the Virtu WUI) are described in req.yaml as services, never required by MeshForge.
**MF-FED-111 (MUST) Resolvable references.** Every requirement and roadmap item cites a resolvable `owner/repo#n` or URL, and a link lint fails on dangling references.
**MF-FED-112 (MUST) Hard gates outside the agent loop.** This includes self-modifying harnesses (familia design PRs #165 Terminal-RSI, #166 Harness²): a harness self-edit MUST NOT touch gate, policy, seatbelt or graph files without an owner-signed diff. Promoting a skill to a guard or route service, or restoring a demoted label, is an owner-signed graph diff (MF-FED-061). Agents MAY propose it but MUST NOT apply it.
**MF-FED-113 (MUST) Egress is an edge.** Any call to a remote model, API or cloud sandbox is a cross-partition edge, subject to MF-FED-022/040. The default is off. Failover chains are declared and covered by the conformance suite.
**MF-FED-114 (SHOULD) Boot chain in the graph.** Netboot and image servers are labeled graph objects with signed images, and a known-CVE check against declared firmware versions.


## Appendix A. Survey of current state (federation angle, 2026-10-28)


**Familia** (brianreborn/familia; Python/shell; graph.yaml is the single source of truth)
- Implemented: graph validator; opt-in SSH nexus plus private tracker PoC (sha256-keyed artifacts, TOFU host keys, no signatures); hosts overlay (#104); guardian/pentest node type with two independent tool disables (#147, open); a monitoring console (#71) that is SSH-based, unsupervised and runs on demand; escalation with "recorded consent" (record format unspecified, unsigned).
- Slated but not implemented: BitTorrent/trackers/rsync as transports with protocol code (DESIGN.md: "node types with no protocol code"); visual/netgraph federated topology UI (DESIGN.md: "no issue tracks this"); HF metadata vocabulary alignment (#120, "hard requirement"); multi-engine same-role instances (#150); speculative decoding across the fleet (#91); swarm per-owner branches (#151); design-only PRs #163 (Jev harness slimming), #165 (Terminal-RSI), #166 (Harness²), all "do not merge without Brian's exact yes"; retry wrapper with no callers (known-issues).
- Docs: ROADMAP.md "v0.2.0" Phase 2 cites issues **#403–#407** (boss-agent-cli). These numbers don't exist in familia, whose highest is ~#175 (see inconsistencies.md G-09).

**Virtu** (gitlab.com/renich/virtu; Crystal, GPL-3.0+, WIP)
- Implemented: libvirt C-FFI core with per-node pools; WUI (Kemal/Datastar) and GTK4 GUI; batch operations; placement policies (single, round-robin); snapshots; secrets vault (Ceph/LUKS/VNC); NWFilter.
- Slated, not implemented (TODO.rst): cordon/evacuate maintenance, embedded SSH/IPMI terminal, remote ISO provisioning.
- Federation-relevant gaps: `cluster.yaml` is only `hypervisors: [{alias, uri, active}]`. There is no identity beyond libvirt URI transport auth, no labels, no health export contract and no API spec published for external callers. Issue tracker: 0 open issues, so the roadmap lives only in TODO.rst.

**Rakazo** (github.com/elie222/rakazo; TypeScript, Apache-2.0, beta, very active; ~100 open issues/PRs)
- Implemented: persistent bots; Spaces as authorization boundaries (Postgres RLS); Team and Private Computers on Docker/E2B/Daytona/etc.; generation fencing for shared browser identity; approval modes for consequential actions; MCP/OpenAPI tool sources; "fleet-management", "content-encryption" and "bot-secrets" docs; offline deterministic conformance tests as policy.
- Slated or open: model failover on revoked auth (#2502, the backup model is never used); judging every action in a Composio batch (#2493); 22 DB tests CI never ran (#2506).
- Federation-relevant gaps: identity is account and session based (Better Auth), not key based; there's no concept of trust between instances (one deployment is the whole world); the server-of-record is Postgres, which conflicts with MeshForge's "no runtime plane" unless it stays internal.


Inconsistencies: `inconsistencies.md`. Open questions: `open-questions.md`.
