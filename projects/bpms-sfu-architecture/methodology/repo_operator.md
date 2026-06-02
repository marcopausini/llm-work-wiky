.claude/skills/repo-operator/
├── SKILL.md                              # orchestrator (110 lines)
├── references/
│   ├── consistency-check.md              # umbrella — runs the read-only audits in sequence
│   ├── marker-audit.md                   # [TBD]/[STUB]/[ASSUMPTION]/[INFERRED] grammar check
│   ├── frontmatter-validate.md           # per-doc-type YAML schema, status invariants
│   ├── cdc-inventory-sync.md             # FAD §4.5 ↔ MS §6.3 (CLAUDE.md inv. 2)
│   ├── module-inventory-sync.md          # FAD §2 ↔ §6 ↔ arch/modules/ ↔ rtl/ (CLAUDE.md inv. 1)
│   ├── citation-audit.md                 # heuristic; CLAUDE.md inv. 6
│   ├── legacy-term-audit.md              # MDS / rtl_ready / MDS-time / rtl_ready_blocking
│   ├── rtm-regenerate.md                 # ⚠ stability caveat — re-validate post pilot spec
│   └── template-scaffold.md              # ⚠ stability caveat — re-validate post Gastón review
└── scripts/
    ├── grep_markers.sh                   # smoke-tested — read-only, idempotent
    └── grep_legacy.sh                    # smoke-tested — excludes ADR-0005 by design