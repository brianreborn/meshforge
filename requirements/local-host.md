# MeshForge — Local Host Requirements 

Status: DRAFT, requirements only. Perspective: the individual machine (one host), its user and its administrator.
Companion: federation requirements by worker A (`federation.md`, MF-FED-###; glossary MF-FED-001, req.yaml MF-FED-002); boundary agreed in `../docs/discussion.md`.
Keywords MUST/SHOULD/MAY per RFC 2119. IDs `MF-LOC-###`.
Sources: X thread review (Grok/Brian thread 2026-10-25), familia checkout (README, REQUIREMENTS.md, ROADMAP.md, docs/install.md, docs/hosts-overlay.md, docs/known-issues.md, open issues/PRs as of 2026-10-28), lessons learned 2026-10-28 (monitor on a FreeBSD host), Virtu README/TODO/cluster.example.yaml (gitlab.com/renich/virtu, v0.1.0), Rakazo README + docs/self-host.md (github.com/elie222/rakazo).

## 0. Glossary
See `glossary.md` (unified).

## 2. req.yaml — local section (shape)
One file per project, one `schema` version (MF-FED-002); `local:` owned here, `federation:` owned by A.

```yaml
schema: meshforge/v0.1   # per MF-FED-002
project: familia
local:
  platforms: [linux, freebsd, termux, windows, macos]   # declared support, each with tier
  dependencies:
    - {name: python3, min: "3.10"}
    - {name: pyyaml, kind: python, required_for: [tests, validate]}   # cf. familia #175
  services:
    - id: coder
      kind: model-server
      start: {argv_template: "llama-server ...", must_include: ["--tools"]}  # cf. L-server-tools
      autostart: required            # see §6
      resources: {ram_mib: {min: 2048, max: 9000}, cpu: {cores: 4}, gpu: optional, disk_mib: 6000}
      network: {bind: loopback, ports: [18200]}
      enforcement: {tier_min: contained, capabilities: [net.bind.loopback, fs.read:models]}
      health: {http: "/health", expect: 200, interval_s: 30}
  host_reserve: {ram_mib: 2048}       # never allocated to services
  probes: [cpu, mem, swap, load, gpu, disk, procs, ports, os, kernel_mac]
```

## 3. Requirements

### 3.1 Declaration and validation (offline)
- **MF-LOC-001 (MUST)** Each project MUST publish a `req.yaml` whose `local:` section is validatable with no network, no model loaded and no daemon running. *Rationale:* the host is the place things break first (a FreeBSD host was off the LAN for an hour on 2026-10-28). *Accept:* `meshforge check req.yaml` exits 0/1 on an air-gapped host in <2 s.
- **MF-LOC-002 (MUST)** Real host facts (addresses, account names, secrets) MUST live only in an uncommitted `host.local.yaml` overlay, with documented per-OS lookup order (XDG / `%LOCALAPPDATA%` / Termux `$PREFIX`), mirroring familia `hosts-overlay.md`. *Accept:* a lint fails if `req.yaml` contains an RFC1918 address, Termux app-user name or key-like string.
- **MF-LOC-003 (MUST)** All runtime dependencies MUST be declared, with the platform package names per OS. *Rationale:* familia #175 (`yaml` undeclared), Virtu's COPR-only Crystal, Rakazo's Docker. *Accept:* fresh-host test installs only declared deps and runs the project's tests.
- **MF-LOC-004 (MUST)** The validator MUST reject a host plan where the sum of service `ram_mib.max` + `host_reserve` exceeds measured MemTotal, and warn when it exceeds MemAvailable. Generalizes familia's `reserve_ram_mib` rule (#29, Linux host memory hang) to all three projects (Rakazo computer limits, Virtu guest memory).
- **MF-LOC-005 (SHOULD)** Resource units MUST be one canonical form (MiB, cores as decimal, Mbit/s); adapters translate (`2g` Rakazo, libvirt KiB). *UX:* the admin sees one number everywhere.
- **MF-LOC-006 (MUST)** "Unlimited" MUST be explicit (`unlimited`) and MUST produce a validator warning on hosts below 8 GiB. Rakazo's `0` = unlimited is ambiguous with "zero".

### 3.2 Host probes
- **MF-LOC-010 (MUST)** Probes MUST be read-only, unprivileged by default, and complete in <1 s per host.
- **MF-LOC-011 (MUST)** Each probe MUST have an implementation per supported platform (Linux `/proc`/sysfs, FreeBSD `sysctl`/`ps -axo`/`swapinfo`, Termux without root, Windows WMI/PowerShell, macOS `sysctl`). A missing source MUST report `unsupported`, never null/0. *Rationale:* the probe on a FreeBSD host returned memory null, uptime 0, "no processes" while 14 llama-servers ran. *Accept:* a CI job on a host with no `/proc` fails if any declared probe returns null without `unsupported`.
- **MF-LOC-012 (MUST)** Probes MUST verify *live* state against declared state (actual argv, bound ports, running PIDs), not just config. *Rationale:* familia `--tools` drift (known-issues, L-server-tools). *Accept:* probe flags a service whose live argv lacks a `must_include` flag.
- **MF-LOC-013 (SHOULD)** Probe output MUST be one versioned JSON document usable by familia-monitor, Virtu telemetry and Rakazo health, so no project re-implements host metrics.
- **MF-LOC-014 (MUST)** Probes MUST report the host's enforcement capabilities (SELinux/AppArmor mode, TrustedBSD MAC modules, Capsicum, Landlock ABI, eBPF/LSM availability, namespaces, root/non-root, container runtime) — the solver needs them for MF-LOC-030.

### 3.3 Install and bootstrap
- **MF-LOC-020 (MUST)** Install MUST succeed and reach a "healthy, idle" state with **no model downloaded and no remote AI account**. *Rationale:* the owner's oh-my-hermes objection (Grok on by default, local models not started). *Accept:* install on an offline host; status shows all services healthy or `waiting-for-model`.
- **MF-LOC-021 (MUST)** Remote/cloud model providers MUST be off by default; enabling one is an explicit admin action recorded in the local audit log. Applies to Rakazo's provider list and familia `isolated_remote`.
- **MF-LOC-022 (MUST)** Every project MUST declare a support tier per platform: `supported` (tested in CI or on a fleet host), `manual`, `untested`, `unsupported`. Docs and installers MUST agree with req.yaml. (See inconsistencies.md G-09, G-17.)
- **MF-LOC-023 (MUST)** Install MUST NOT need root unless the declared tier needs it (Virtu needs libvirt groups; familia needs none), and MUST print each privileged step before doing it.
- **MF-LOC-024 (SHOULD)** Re-running install MUST be idempotent and preserve local overlays/secrets (Rakazo already preserves `.env`; familia `update` preserves `~/.config/familia`).
- **MF-LOC-025 (MUST)** Uninstall MUST remove every service unit/rc.d/cron/Termux:Boot entry it created and list what it left (models, data).

### 3.4 Local enforcement (enforce locally first)
- **MF-LOC-030 (MUST)** Each service MUST declare its needed capabilities as an allow-list (fs paths, net binds/peers, devices such as GPU/VFIO, process spawn, tool execution). The host binds them to the strongest available mechanism: Linux Landlock + seccomp/namespaces + SELinux/AppArmor; FreeBSD Capsicum + jails + TrustedBSD MAC (mac_lomac where loaded); container limits for Rakazo; libvirt sVirt for Virtu guests; Termux/Windows → `advisory`.
- **MF-LOC-031 (MUST)** The resulting enforcement tier per service MUST be shown to the admin and stored in the probe document. A service whose declared `tier_min` exceeds what the host offers MUST NOT start (fail closed) unless the admin overrides in `host.local.yaml` with a reason.
- **MF-LOC-032 (MUST)** Local policy is final: a federated grant (A's domain) is intersected with local policy and can only narrow it (= MF-FED-033). *Accept:* test where a signed remote graph grants net egress denied locally → denied, logged.
- **MF-LOC-033 (MUST)** LOMAC labels (lattice from A) MUST be enforced per host: `on_low_read: demote|deny`. `deny` requires a `hard` tier; on `advisory` hosts the service is marked `advisory` and MF-FED-034 keeps high-integrity data away unless an owner-signed waiver exists.
- **MF-LOC-034 (MUST)** Safety gates MUST be nodes (always on), not skills (skippable); promoting a skill to a gate MUST be an operator edit of local config outside the agent loop. (Laya thread; familia #147.) Agents MUST NOT be able to write the files that define their own gates (enforced via MF-LOC-030).
- **MF-LOC-035 (MUST)** Familia's "tools double-disabled for launch tests" seatbelt MUST become a MeshForge-wide rule: any automated launch test of an agent-capable service (Rakazo bot, familia node, hermes) uses two independent disables.
- **MF-LOC-037 (MUST)** The host MUST keep a pinned per-partition `known_hosts` (admin-owned, not writable by agents). `accept-new` (TOFU, used by familia's transport PoC today) is allowed only for hosts listed in the partition directory (A, MF-FED). A changed key is a hard failure plus an admin alert.
- **MF-LOC-038 (MUST)** A shared guest (Rakazo Team Computer) carries ONE label, the meet of the bots sharing it (agreed with A). Status MUST show which bot lowered it.
- **MF-LOC-036 (SHOULD)** Bot-to-host access (Rakazo "trusted local computer", familia server tools) MUST be a declared capability with its own tier, default off.

### 3.5 Always-on operation and monitoring
- **MF-LOC-040 (MUST)** Every service with `autostart: required` (the monitor/alert loop included) MUST ship a per-OS supervisor definition: systemd user/system unit, FreeBSD rc.d (or `daemon(8)` via cron @reboot), Termux:Boot script, Windows Task Scheduler, launchd. *Rationale:* lesson 2026-10-28 — the monitor only ran when Brian asked because nothing installed it. Virtu ships only systemd; Rakazo relies on Docker restart policy.
- **MF-LOC-041 (MUST)** Acceptance for any monitoring or health feature MUST include "installed and running after reboot without a human", tested per supported platform. "The command works" is not acceptance.
- **MF-LOC-042 (MUST)** Supervisors MUST restart crashed services with backoff and MUST NOT crash-loop silently (cf. familia #94 forgejo restarting every 2 s). After N failures they stop and raise an alert.
- **MF-LOC-043 (MUST)** The monitor MUST run on the host it measures (host-local), with a pull/export interface; it MUST NOT depend on one central SSH host being reachable.
- **MF-LOC-044 (SHOULD)** Alerts MUST reach the admin through at least one channel that works offline on the LAN (local log + desktop/terminal notification), with optional remote channels.
- **MF-LOC-045 (MUST)** Installing anything persistent on a host (unit, cron, rc.conf change) MUST be an explicit admin approval step, shown as a diff first.

### 3.6 Offline and degraded operation
- **MF-LOC-050 (MUST)** A host MUST keep running local services and local-only edges indefinitely while partitioned; only cross-host edges may expire (agreed with A).
- **MF-LOC-051 (MUST)** `meshforge status` (or each project's equivalent) MUST say in one line: online/partitioned since T, running graph signature, degraded services and why.
- **MF-LOC-052 (MUST)** Network loss detection MUST distinguish "host down", "link down" and "service down" (e.g. no ARP entry vs port closed).
- **MF-LOC-053 (SHOULD)** Models and artifacts MUST be addressable by content hash so an offline host can verify local copies and later reseed (BitTorrent per A).

- **MF-LOC-054 (MUST)** Each host MUST keep a local, file-based, append-only state: the last accepted graph + signature, the allocation ledger, and an audit log of approvals, overrides and demotions. No always-on coordinator or DB server is required (agreed with A; Rakazo's Postgres stays internal). Federated audit reads from it.

### 3.7 Resource allocation
- **MF-LOC-060 (MUST)** The host is the allocator of its own resources; the solver proposes, the host accepts or rejects with a reason.
- **MF-LOC-061 (MUST)** Allocation MUST cover RAM, CPU cores, GPU (device, VRAM), disk, open ports, and process count, with a declared `host_reserve`.
- **MF-LOC-065 (MUST)** The host MUST publish capacity and a `draining` state (Virtu cordon/evacuate, unimplemented in Virtu TODO). While draining, the local allocator refuses new guests and services.
- **MF-LOC-062 (MUST)** Ports MUST be registered per host; collisions (Virtu 5050, Rakazo 5173/45173, familia 9931, llama 18200+) MUST be a validator error, not a runtime surprise.
- **MF-LOC-063 (SHOULD)** Low-RAM/old-device profiles (familia #48 lowram, non-AVX) MUST be expressible as platform constraints (`cpu_flags`, `ram_mib`), and the solver MUST pick a compatible build.
- **MF-LOC-064 (MUST)** Memory-pressure protection (earlyoom/zram/swap, Rakazo `PIDS_LIMIT`) MUST be part of the host plan, and missing protection MUST warn (familia #43, #29).

### 3.8 Local adapters
- **MF-LOC-070 (MUST)** Each pairwise integration MUST be a local adapter with its own req.yaml: familia→Rakazo (familia model servers exposed as Rakazo's OpenAI-compatible provider), Virtu→familia (Virtu guest = familia host; guest resources from req.yaml), Rakazo→Virtu (Rakazo "computer" provider backed by libvirt guests instead of Docker).
- **MF-LOC-071 (MUST)** Adapters MUST bind loopback by default; off-loopback MUST require auth (Virtu already requires basic-auth; familia :9931 API key per #171).
- **MF-LOC-072 (SHOULD)** Adapters SHOULD be generated stubs from interface specs (OpenAPI/Protobuf) and tested against each project's real health endpoint on a single host.

### 3.9 User and administrator experience
- **MF-LOC-080 (MUST)** One command per host answers "what's running, is it healthy, what is it allowed to do, what's it using": services, tier, resources, ports, graph signature.
- **MF-LOC-081 (MUST)** Every error MUST name the file, field and fix (e.g. "coder.ram_mib.max 9000 > MemTotal 7300 on host-a; lower it or move coder").
- **MF-LOC-082 (MUST)** Secrets MUST never be pasted into chat or config by the user when a file/agent can read them (familia #174); secrets live in per-OS secure locations with 0600 perms and are never logged.
- **MF-LOC-083 (SHOULD)** The config UI (familia #79 accessible panel, Virtu WUI/GUI, Rakazo web) SHOULD edit the same req.yaml/host.local.yaml, not private copies.
- **MF-LOC-084 (MUST)** Docs per platform MUST be generated from or checked against req.yaml support tiers (no drift).

## 4. Mapping the 18 review items (local relevance)
2 → MF-LOC-001..006; 3 → 030–033; 4 → 053 (+A); 5 → 070; 6 → 034; 8 → 003/022 (FreeBSD -lpthread is a declared-platform build dep); 9 → 061 (GPU/RAM per stage); 13 → 063; 14 → MF-LOC-090; 15 → 061/014 (GPU probe); 16 → 024/050 (Termux snapshots as host backup).
- **MF-LOC-090 (MUST)** Boot path is part of the host plan: hosts that netboot (PXE/UEFI HTTP boot) MUST declare it so firmware CVEs (EDK II net-boot CVE in the review) can be audited per host.
- **MF-LOC-091 (SHOULD)** Host snapshot/restore (Termux snapshots, Virtu domain snapshots, ZFS on FreeBSD) SHOULD be declared so recovery is local-first.

## 6. Acceptance summary for the "always-on" lesson
A feature tagged `autostart: required` is done only when, on each supported platform: (1) install creates the supervisor entry after admin approval, (2) reboot → running within 60 s with no human, (3) kill -9 → restarted, (4) 5 fast crashes → stopped + alert, (5) uninstall removes it.

## Appendix A. Survey of current state (local-machine angle, 2026-10-28)

### Familia (github.com/brianreborn/familia)
- Declarative `graph.yaml` (hosts, nodes, roles), `validate_graph.py` checks RAM overcommit (`reserve_ram_mib`, 2048 MiB small-host floor), per-host overlay `hosts.local.yaml` keeps real addresses out of git (#104).
- Installers: Windows/Linux/macOS(untested)/Termux one-click; FreeBSD manual only (#37 preliminary port).
- Seatbelt requirement: tools double-disabled for launch tests (REQUIREMENTS.md Core).
- Monitoring: `familia-monitor` is an on-demand CLI (#71, PR #164 deploys the probe only). Probe reads Linux `/proc` only; broke on a FreeBSD host (lesson 2026-10-28).
- Open, not implemented: #147 guardian node, #120 HF-metadata vocabulary, #150 archetype defaults, #79 accessible config panel, #69 command-line builder, #48 lowram/non-AVX profile, #43 swap on a Linux host, #29 memory-pressure hang, #94 forgejo crash loop, #175 undeclared `yaml` dependency, #174/#171 API-key handling, draft design PRs #163 (Jev slimming), #165 (Terminal-RSI), #166 (Harness²).
- Known issues: `--tools` is launch-only (live argv drifts from graph); Vulkan `-ngl 0` shmem; one known test failure.
### Virtu (gitlab.com/renich/virtu, v0.1.0, WIP)
- Crystal + libvirt; `cluster.yaml` (hypervisor list: alias, uri, active) at `./` or `~/.config/virtu/`; all other config via env vars; WUI binds 127.0.0.1:5050, basic auth required off-loopback.
- Install: Fedora packages, `make install` ships a **systemd unit** (Linux-only service story). Users need libvirt/kvm/qemu group membership.
- TODO: Cordon/Evacuate maintenance not done; remote ISO provisioning not done. No public open issues (API returned 0).
- No declared resource contract for itself; VM resources are per-domain XML.
### Rakazo (github.com/elie222/rakazo, beta)
- TypeScript stack, Docker Compose install (`install-images.sh`, random secrets into `.env`), images linux/amd64+arm64 only; web on 127.0.0.1:5173 (desktop 45173).
- Per-computer limits exist: `RAKAZO_COMPUTER_MEMORY` (2g), `RAKAZO_COMPUTER_CPUS` (2), `RAKAZO_COMPUTER_PIDS_LIMIT` (2048); `0/none/unlimited` removes a ceiling.
- Models: bring-your-own credentials; supports operator-run OpenAI-compatible servers (llama.cpp, Ollama) — the natural seam to familia.
- "Trusted local computer" mode gives bots access to the host itself.


Inconsistencies: `inconsistencies.md`. Open questions: `open-questions.md`.
