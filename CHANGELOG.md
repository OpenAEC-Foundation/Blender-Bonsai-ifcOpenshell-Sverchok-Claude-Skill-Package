# Changelog

All notable changes to this project will be documented in this file.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [2.0.0] - 2026-03-07

### Added
- Sverchok skill package: 12 skills across 5 categories (syntax 4, impl 5, errors 1, core 1, agents 1)
- New skills: sverchok-core-concepts, sverchok-syntax-sockets, sverchok-syntax-data, sverchok-syntax-scripting, sverchok-syntax-api, sverchok-impl-custom-nodes, sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-topologic, sverchok-impl-extensions, sverchok-errors-common, sverchok-agents-code-validator
- Total skill count increased from 61 to 73 across 5 technology packages

### Changed
- Updated INDEX.md, README.md, REQUIREMENTS.md to reflect Sverchok completion
- Package table now shows 5 technology packages (Blender, IfcOpenShell, Bonsai, Sverchok, Cross-Tech)

## [1.1.0] - 2026-03-06

### Added
- Phase 8: Workshop workspace deployed to Computational-Design-Day-Delft-March-2026
- 61 skills + Blender MCP + CLAUDE.md + SETUP.md deployed to workshop workspace
- Decisions D-012 through D-015 (workspace deployment, dual CLAUDE.md, no setup script, Blender MCP only)

## [1.0.0] - 2026-03-06

### Added
- 61 skills across 4 technology packages (Blender 26, IfcOpenShell 19, Bonsai 14, Cross-Tech 2)
- Complete skill catalog (INDEX.md)
- Phase 6 validation reports (docs/validation/)
- Skills CLAUDE.md for project-level Claude configuration

### Skill Categories
- 19 syntax skills (API patterns and code reference)
- 22 implementation skills (workflows and best practices)
- 7 error handling skills (diagnostics and recovery)
- 8 core skills (concepts, runtime, versions)
- 5 agent skills (code validators, version migrator, workflow orchestrator)

### Quality
- All 61 skills validated: <500 lines, 3+ reference files, English only, deterministic language
- 244 reference files (methods.md, examples.md, anti-patterns.md)
- Zero Dutch content, zero banned hedging language

## [Unreleased]

### Added
- Phase 2: Deep Research (next)

## [0.1.0] - 2026-03-05

### Added
- Project initialized with 7-phase research-first methodology
- CLAUDE.md with 8 protocols (P-001 through P-008)
- WAY_OF_WORK.md adapted from ERPNext Skill Package
- ROADMAP.md as single source of truth
- REQUIREMENTS.md defining quality guarantees and per-technology requirements
- DECISIONS.md with 9 architectural decisions (D-001 through D-009)
- SOURCES.md with official documentation links for all technologies
- LESSONS.md with initial lessons (L-001 through L-003)
- Separate skill package directories: blender/, bonsai/, ifcopenshell/, sverchok/, aec-cross-tech/
- Raw masterplan with preliminary skill inventory (~43 skills)
- MIT License
- GitHub repo metadata (description, topics)
