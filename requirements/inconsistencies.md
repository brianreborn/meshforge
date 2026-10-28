# Inconsistencies and gaps (merged)

Found by cross-checking the current state of Familia, Virtu and Rakazo, including requirements and standards each has slated but not implemented. Rule (agreed): every finding cites a source and names the requirement that resolves it. Merged from the federation (G-) and local-host (I-) reviews; former IDs in brackets.

| ID | Finding | Evidence | Resolving requirement |
|---|---|---|---|
| G-01 | "node" means a model server (familia), a physical host (Virtu/libvirt); "domain" means a VM (libvirt) and a trust domain (MeshForge thread). [G-01, I-4] | familia graph.yaml, Virtu TODO.rst, thread | MF-FED-001, glossary.md, MF-LOC-070 mapping tables; unqualified "node" is a lint error |
| G-02 | Three unrelated identity models: SSH TOFU `accept-new` (familia), libvirt URI (Virtu), DB accounts/sessions (Rakazo). None can sign a graph. [G-02] | familia transport-poc.md, Virtu cluster.example.yaml, Rakazo stack | MF-FED-010, 011, MF-LOC-037 |
| G-03 | "Recorded consent" (familia), "approval modes" (Rakazo), nothing (Virtu): three unspecified, unsigned formats. [G-03] | familia REQUIREMENTS.md, Rakazo VISION | MF-FED-040, 041 |
| G-04 | Two content keys for one artifact: magnet URI + path (familia DESIGN.md) vs sha256 (transport-poc). [G-04] | familia DESIGN.md, docs/transport-poc.md | MF-FED-070, MF-LOC-053 |
| G-05 | Monitoring is on demand in familia (#71, #164), systemd-only in Virtu, Docker-restart in Rakazo; nothing supervises it on FreeBSD/Termux/Windows. The FreeBSD probe returned empty values with no error. [G-05, I-3] | familia #71, lessons learned 2026-10-28 | MF-FED-063, MF-LOC-040, 041, 011 |
| G-06 | Recovery verbs undeclared: Virtu cordon/evacuate unimplemented, familia selfheal ad hoc, Rakazo suspend/restore internal. [G-06] | Virtu TODO.rst, familia reconnect-selfheal.md | MF-FED-062, MF-LOC-065 |
| G-07 | "No runtime plane" vs Rakazo's always-on Postgres and Virtu's long-running WUI. [G-07] | thread vs Rakazo self-host.md | MF-FED-110, MF-LOC-054 |
| G-08 | No labels anywhere; familia guardian scopes and Rakazo Spaces are unordered boundaries, so taint across them is undefined. [G-08] | familia #147, Rakazo VISION | MF-FED-030, 031, MF-LOC-033 |
| G-09 | familia ROADMAP.md cites issues #403–#407 and "boss-agent-cli" work that don't exist (highest issue ~#175), and claims FreeBSD support "in progress" while install.md says unsupported. [G-09, I-1] | familia ROADMAP.md, install.md | MF-FED-111, MF-LOC-022, 084 |
| G-10 | Model/guest vocabulary: familia graph vs HF metadata (#120, "hard requirement") vs libvirt XML; Rakazo/Virtu have no counterpart. [G-10, I-15] | familia #120, Virtu | MF-FED-081, MF-LOC-013, 070 |
| G-11 | Shared mutable state: Rakazo fences generations locally only; familia KV move has none. [G-11] | Rakazo VISION, familia c2c-stage1.md | MF-FED-064 |
| G-12 | Rakazo Team Computer shares one browser identity across bots: a confused-deputy channel when bots differ in label. [G-12] | Rakazo VISION | MF-FED-031, MF-LOC-038 |
| G-13 | Hosts that can't enforce policy (Termux, Windows) are treated like enforcing hosts. [G-13] | familia android/windows docs | MF-FED-034, MF-LOC-033 |
| G-14 | Safety gates exist only as skippable skills (Laya); Rakazo "trusted local computer" grants host access; design PRs #163/#165/#166 let agents rewrite their own harness. [G-14, I-11, I-14] | Laya thread, familia #147, PRs, Rakazo docs | MF-FED-112, MF-LOC-034, 035, 036 |
| G-15 | Cloud defaults: remote providers on by default (Rakazo onboarding, oh-my-hermes objection); Rakazo failover broken (#2502); familia assumes local models run. [G-15, I-8] | Rakazo #2502, README; X review | MF-FED-113, MF-LOC-020, 021 |
| G-16 | Netboot (UEFI EDK II net-boot CVE) is a trust root no project models. [G-16] | X review item 14 | MF-FED-114, MF-LOC-090 |
| G-17 | ROADMAP claims "FreeBSD requires /proc"; the probe assumed /proc; the correct fix was native sysctl/ps. ROADMAP is untrusted evidence. [G-17, I-2] | familia ROADMAP.md, lessons learned | MF-FED-102, 111, MF-LOC-011 |
| G-18 | Three config models (graph.yaml + overlay, cluster.yaml + env vars, `.env`); the same machine is described three times. [I-5] | each README | MF-LOC-002, 083 |
| G-19 | Resource limits nobody sums per host: familia RAM reserve only, Rakazo per-computer limits with `0` = unlimited, Virtu per-VM XML. [I-6] | familia known-issues, Rakazo self-host.md | MF-LOC-004, 005, 006, 061 |
| G-20 | Uncoordinated port defaults (5050, 5173, 45173, 9931, 18200+); Rakazo silently picks another. [I-7] | READMEs | MF-LOC-062 |
| G-21 | Declared vs live drift: familia `--tools` is launch-only; live argv lacks graph's `server_tools`. [I-9] | familia known-issues | MF-LOC-012 |
| G-22 | Undeclared dependencies: familia #175 (yaml), FreeBSD -lpthread for llama.cpp 0.7.0, Virtu's Fedora-only Crystal packaging. [I-10] | familia #175, X review item 8 | MF-LOC-003, 022 |
| G-23 | Crash loops go unnoticed (familia #94, restart every 2 s). [I-12] | familia #94 | MF-LOC-042 |
| G-24 | Rakazo images are linux/amd64+arm64 only, but the fleet includes FreeBSD and Termux hosts. [I-13] | Rakazo README | MF-LOC-022, 070 |
| G-25 | A test fails on hosts holding real model files: tests depend on host state. [I-16] | familia known-issues | MF-LOC-001 |
