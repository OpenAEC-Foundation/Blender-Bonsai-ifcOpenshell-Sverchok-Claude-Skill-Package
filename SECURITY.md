# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 0.1.x | Yes |

## Reporting a Vulnerability

### How to Report
1. Do NOT open a public GitHub issue for security vulnerabilities
2. Email: report via GitHub Security Advisories on this repository
3. Include: description, reproduction steps, potential impact

### What to Expect
- Acknowledgment within 48 hours
- Assessment within 1 week
- Fix or mitigation plan within 2 weeks

### Scope
This project contains documentation and skill files (Markdown). Security concerns include:
- Incorrect code patterns that could introduce vulnerabilities in user projects
- Anti-patterns that bypass security mechanisms in Blender/IfcOpenShell
- Misleading information about authentication or permissions

## Security Best Practices for Users
1. Always verify skill-generated code before running in production
2. Review code that accesses file systems or network resources
3. Keep Blender, IfcOpenShell, and other tools updated to latest versions
4. Never run untrusted Python scripts in Blender without review
