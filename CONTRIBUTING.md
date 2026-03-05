# Contributing

This project develops Claude Skills for AEC technologies (Blender, Bonsai, IfcOpenShell, Sverchok). Contributions are welcome.

## How to Contribute

### Reporting Issues
- Bug reports: incorrect skill content, wrong API references, version errors
- Feature requests: new skills, additional technologies, improved patterns
- Questions: open a GitHub issue

### Submitting Changes
1. Fork the repository
2. Create a feature branch: `git checkout -b phase-X-description`
3. Follow the skill structure (see below)
4. Verify against official documentation (see SOURCES.md)
5. Commit with descriptive message: `Phase X.Y: [action] [subject]`
6. Push and create a Pull Request

## Skill Development Guidelines

### Structure
```
skills/{technology}/{category}/{skill-name}/
├── SKILL.md              # Main file, < 500 lines
└── references/
    ├── methods.md        # API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do
```

### SKILL.md Requirements
- YAML frontmatter with `name` and `description` (including trigger words)
- Under 500 lines (heavy content goes in references/)
- English-only content
- Deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit: annotate code with supported versions

### Content Standards

**DO:**
- Verify all code against official docs (see SOURCES.md)
- Include version-specific information (Blender 3.x/4.x, IFC2x3/IFC4/IFC4.3)
- Provide working examples tested against real APIs
- Document anti-patterns with explanations

**DON'T:**
- Use vague language: "you might consider", "it's often good practice"
- Make assumptions about API behavior
- Copy from unverified sources
- Skip version compatibility info

## Commit Message Format
```
Phase X.Y: [action] [subject]

Types:
- Phase 1.2: Add project requirements
- Phase 5.3: Create blender-syntax-operators skill
- Phase 6.1: Validate Blender skills batch A
```

## Questions?
Open an issue on GitHub or check existing documentation:
- REQUIREMENTS.md - What skills must achieve
- WAY_OF_WORK.md - Development methodology
- SOURCES.md - Approved reference sources
