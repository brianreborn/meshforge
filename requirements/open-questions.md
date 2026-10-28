# Open questions

Each is tracked as a GitHub issue. Q1 is the only joint question both perspectives raised.

1. **[RESOLVED 2026-10-28 by Brian] Who approves persistent installs on unattended hosts (phones, a headless FreeBSD box)?** Answer: the owner (Brian) MUST personally approve every persistent install on an unattended host (RFC 2119 MUST). The host's local approval (MF-LOC-045) and a signed federated approval (MF-FED-040) MAY still be required in addition, but neither substitutes for owner approval. (MF-LOC-040, 045, 046; MF-FED-042)
2. Reference OpenPGP stack: Sequoia (RFC 9580) or GnuPG? Affects smartcards on FreeBSD and Termux. (MF-FED-010, 013)
3. Bridging Kerberos and WoT: map `svc/host@REALM` to an OpenPGP UID via a signed directory attribute, or via a separate bridge keyring? (MF-FED-020, 022)
4. Lattice components: are realm-path and tagset enough, or are numeric confidentiality levels (classic MLS) needed? (MF-FED-030)
5. Demotion scope: per service process or per conversation/session? Long-lived Rakazo bots could stay demoted indefinitely. (MF-FED-031)
6. Minimum audit witness count for a mostly offline phone fleet. (MF-FED-090)
7. Should Rakazo instances federate directly, or only through MeshForge bridges? (MF-FED-022)
8. Will Virtu upstream accept a req.yaml and recovery-verb contract, or does MeshForge carry the adapter externally? (MF-FED-062, 100)
9. Who holds the owner key when Brian is unavailable: escrow, or an m-of-n admin quorum? (MF-FED-010, 012)
10. Reference implementations: must both validators run on FreeBSD and Termux? (MF-FED-013, MF-LOC-001)
11. Windows: `advisory` only, or require AppContainer/Job Objects for `contained`? (MF-LOC-030, 031)
12. Allocator ownership: does Virtu allocate guests and the host allocator only bare metal, or does the host allocator own both? (MF-LOC-060, 065)
13. GPU sharing (legacy GPUs, Vulkan): time-slicing declared in req.yaml, or exclusive only? (MF-LOC-061)
14. Will Rakazo accept a non-Docker "computer" provider (libvirt via Virtu, FreeBSD jail)? (MF-LOC-070)
15. Is familia ROADMAP.md stale or generated? It should be fixed or removed before reverse-synthesis. (MF-FED-102, G-09, G-17)
