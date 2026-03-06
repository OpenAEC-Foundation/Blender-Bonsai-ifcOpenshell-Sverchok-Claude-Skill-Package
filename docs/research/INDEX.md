# Research Index

All research documents produced during Phase 2, with status and coverage mapping.

**Last updated**: 2026-03-06
**Total research volume**: ~23,000 lines across 18 active documents

---

## Primary Research (vooronderzoeken)

| Document | Lines | Technology | Prompt | Status |
|----------|-------|-----------|--------|--------|
| vooronderzoek-blender.md | 2,099 | Blender | A + A2 | COMPLETE + ENRICHED |
| vooronderzoek-bonsai.md | 1,759 | Bonsai | A | COMPLETE |
| vooronderzoek-ifcopenshell.md | 2,030 | IfcOpenShell | A + A3 | COMPLETE + ENRICHED |
| vooronderzoek-ecosystem-sources.md | 1,678 | Cross-tech | F | COMPLETE (consolidated from 3 fragments) |

## Supplementary Research (gap domains)

| Document | Lines | Technology | Prompt | Status |
|----------|-------|-----------|--------|--------|
| supplementary-blender-gaps.md | 2,242 | Blender | G | COMPLETE |
| supplementary-bonsai-gaps.md | 1,555 | Bonsai | G | COMPLETE |
| supplementary-ifcos-gaps.md | 1,380 | IfcOpenShell | G | COMPLETE |

## IfcOpenShell Deep-Dive Fragments

| Document | Lines | Focus | Status |
|----------|-------|-------|--------|
| fragments/ifcos-api-categories.md | 1,170 | API module categorization | COMPLETE |
| fragments/ifcos-core-operations.md | 978 | Core CRUD operations | COMPLETE |
| fragments/ifcos-errors-performance.md | 1,108 | Error patterns + performance | COMPLETE |
| fragments/ifcos-schema-versions.md | 833 | Schema version comparison | COMPLETE |

## IfcOpenShell Topic Research

| Document | Lines | Focus | Status |
|----------|-------|-------|--------|
| topic-research/ifcos-core-operations.md | 1,473 | Extended core operations | COMPLETE |
| topic-research/ifcos-errors-performance-research.md | 1,518 | Extended error research | COMPLETE |
| topic-research/ifcos-schema-version-comparison.md | 1,484 | Extended schema comparison | COMPLETE |

## Ecosystem Fragments (consolidated into vooronderzoek-ecosystem-sources.md)

| Document | Lines | Focus | Status |
|----------|-------|-------|--------|
| fragments/ecosystem-claude-platform.md | 449 | Claude skill specification | CONSOLIDATED |
| fragments/ecosystem-openaec-code.md | 521 | building.py, GIS-to-Blender, AEC Scripts | CONSOLIDATED |
| fragments/ecosystem-openaec-ifc.md | 597 | Monty IFC Viewer, INB Template, Nextcloud | CONSOLIDATED |

## Analysis

| Document | Lines | Focus | Status |
|----------|-------|-------|--------|
| scope-analysis.md | 483 | Complete API surface mapping, coverage gaps | COMPLETE (updated after G) |

---

## Coverage by Phase 3 Briefings

Every document above was read by at least one interpretation agent and condensed into briefings for the masterplan agent:

| Briefing | Interpreter agent | Research inputs |
|----------|------------------|-----------------|
| briefing-blender.md | interpret-blender | vooronderzoek-blender, supplementary-blender-gaps, scope-analysis, raw-masterplan |
| briefing-ifcopenshell.md | interpret-ifcos | vooronderzoek-ifcopenshell, supplementary-ifcos-gaps, all 4 ifcos fragments, all 3 topic-research docs, scope-analysis |
| briefing-bonsai.md | interpret-bonsai | vooronderzoek-bonsai, supplementary-bonsai-gaps, scope-analysis |
| briefing-cross-tech.md | interpret-cross-tech | vooronderzoek-ecosystem-sources, scope-analysis, all 3 vooronderzoeken (skimmed) |

Ecosystem fragments (3 files) are covered indirectly: they were consolidated into vooronderzoek-ecosystem-sources.md by f-consolidator agent.

---

## Archive (completed staging data)

| Directory | Contents | Why archived |
|-----------|----------|-------------|
| archive/worker-output/a2/ | A2 enrichment worker output (3 files) | Merged into vooronderzoek-blender.md |
| archive/worker-output/a3/ | A3 enrichment worker output (4 files) | Merged into vooronderzoek-ifcopenshell.md |
| archive/agent-prompts-f/ | Prompt F sub-agent prompts (4 files) | Executed, output in fragments/ |
