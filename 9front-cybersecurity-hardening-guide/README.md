# 9front Cybersecurity Hardening Guide

**Author:** Anastasios Papalias  
**First Published:** 2026  
**Repository:** github.com/AnastasiosPapalias/Plan9

---

## Two Editions

### Full Edition (Commercial)
- 200 pages · All rights reserved
- 34 chapters + 2 reference chapters + 4 appendices
- Parts: Foundations · Security Mechanisms · Comparative Analysis ·
  Threat Engineering · Practical Labs · Academic Context · Reference

### Community Edition (Free)
- ~25 pages · CC BY 4.0
- 12 chapters in KV-block + table format
- Machine-friendly for AI agent consumption

---

## Companion Files

| File | Description |
|------|-------------|
| `security_dataset.jsonl` | 12 STIX 2.1 attack-pattern objects (9front threat dataset) |
| `diagrams/namespace_tree.md` | Mermaid: Plan 9 namespace hierarchy |
| `diagrams/9p_flow.md` | Mermaid: 9P protocol message flow |
| `diagrams/process_model.md` | Mermaid: rfork isolation model |
| `diagrams/factotum_auth.md` | Mermaid: Three-party factotum authentication |
| `diagrams/attack_surface_map.md` | Mermaid: Complete attack surface map |

---

## STIX 2.1 Threat Dataset

12 attack-pattern entries covering four attack surfaces:

| ID | Name | Surface | ATT&CK | Severity |
|----|------|---------|--------|----------|
| 9f-001 | namespace_escape | namespace | T1574 | HIGH |
| 9f-002 | namespace_injection | namespace | T1574 | HIGH |
| 9f-003 | factotum_rpc_interception | factotum | T1552 | CRITICAL |
| 9f-004 | factotum_key_theft | factotum | T1552.004 | CRITICAL |
| 9f-005 | 9p_unauthenticated_attach | 9p | T1133 | CRITICAL |
| 9f-006 | 9p_fid_exhaustion | 9p | T1499 | MEDIUM |
| 9f-007 | 9p_replay_attack | 9p | T1557 | HIGH |
| 9f-008 | proc_memory_read | /proc | T1055 | CRITICAL |
| 9f-009 | proc_note_injection | /proc | T1055.008 | MEDIUM |
| 9f-010 | srv_rogue_registration | namespace | T1574 | HIGH |
| 9f-011 | rfork_namespace_inheritance | namespace | T1611* | HIGH |
| 9f-012 | fd_leakage | /proc | T1083 | HIGH |

\* T1611 is an analogical mapping. No Plan 9-specific ATT&CK technique exists.

---

## Technical Accuracy Notes

- **Saltzer & Schroeder (1975):** Proc. IEEE Vol. 63 No. 9, Sept. 1975, pp. 1278–1308. Not CACM.
- **dp9ik:** 9front-specific. NOT in Bell Labs auth.pdf. Source: `sys/src/cmd/auth/` in 9front repo.
- **auth.pdf [4] (Cox 2002):** Documents p9sk1 ONLY. Never cite for dp9ik.
- **All ATT&CK mappings:** Analogical. No Plan 9-specific ATT&CK techniques exist.
- **T1574:** Use parent, NOT T1574.006 (dynamic linker — wrong mechanism for bind misuse).
- **T1611:** Analogical for rfork namespace inheritance. Always flagged as analogical.
- **NIST AI RMF (AI 100-1):** Voluntary. Recommends; does not mandate.

---

## License

- Community Edition: CC BY 4.0
- Full Edition: All rights reserved © Anastasios Papalias 2026
- security_dataset.jsonl: CC BY 4.0
