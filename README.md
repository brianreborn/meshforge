# MeshForge

**Status: requirements only. Nothing is implemented. No implementation work (by people or the agent swarm) starts until Brian approves it.**

MeshForge is a local-first, declarative *requirements broker* with no required runtime plane. It ties together three projects:

- **Familia** (brianreborn/familia): agents and the model-serving graph
- **Virtu** (gitlab.com/renich/virtu): libvirt/KVM host and guest management across machines
- **Rakazo** (github.com/elie222/rakazo): persistent bots and sandboxed computers

Each project publishes one `req.yaml` that declares its interfaces, data contracts, resource limits, consent rules and security labels. A validator and graph solver checks all of them together, flags conflicts, and proposes topologies and adapter stubs. Security follows a distributed restatement of TrustedBSD MAC / LOMAC: labels on a dominance lattice, OpenPGP-signed graphs and consent records, Kerberos or OpenPGP web-of-trust partitions, and local enforcement first.

## Origin
Brian and Grok worked out the idea on 2026-10-25, inside @NFT_Chen's thread about Rakazo: https://x.com/grok/status/2114388882099757310

## Layout
| Path | Contents |
|---|---|
| `requirements/federation.md` | MF-FED-###: everything that crosses a host, network or trust boundary |
| `requirements/local-host.md` | MF-LOC-###: the single machine, its user and its admin |
| `requirements/glossary.md` | Unified terminology (bare "node" is banned) |
| `requirements/inconsistencies.md` | G-## findings across the three projects, each with a resolving requirement |
| `requirements/open-questions.md` | Decisions awaiting Brian |
| `docs/discussion.md` | Summary of how the two perspectives were reconciled |
| `examples/req.yaml` | Illustrative req.yaml (not normative) |

Keywords MUST/SHOULD/MAY follow RFC 2119. Licensing is not yet decided.
