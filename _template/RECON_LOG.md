# RECON_LOG — <target> (append-only ledger)

Source of truth for what recon was already done, with enough context to decide whether it is still fresh/useful.
A stage marked DONE is not rerun blindly; refresh only for stale evidence, changed scope/behavior, or explicit operator request.

| Date | Stage | Status | Artifact | Delta / reason |
|------|-------|--------|----------|----------------|
|      | 1 passive seed | | | |
|      | 2 enum+resolve | | | |
|      | 3 DNS analysis | | | |
|      | 4 alive probe | | | |
|      | 5 JS analysis | | | |
|      | 6 wayback mining | | | |
|      | 7 port scan | | | |
|      | 8 fingerprint | | | |
|      | 9 github/code leaks | | | |
|      | 10 search-engine dorks | | | |
|      | 11 shodan/censys/fofa | | | |
|      | 12 cloud assets | | | |
|      | 13 mobile recon | | | |
|      | 14 API discovery | | | |
|      | 15 content discovery | | | |
|      | 16 automation/dedup | | | |

Status values: `DONE`, `PARTIAL`, `MISSING`, `N/A`, `REFRESHED`.

## Audit notes

<!-- Example:
| 2026-10-14 | AUDIT | PARTIAL | gaps: DNS, JS secrets | next: DNS only; port scan N/A by rules |
-->
