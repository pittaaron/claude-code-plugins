# Security Policy

## The threat model you're opting into

When you run `/plugin install project-sync@aaronpitt`, you are trusting every future commit to this repo to be safe for Claude to execute on your machine. Skills are instructions that Claude follows — a malicious `SKILL.md` could instruct Claude to exfiltrate data, read sensitive files, or write backdoors, subject to your Claude Code permission settings.

Only install plugins from repos you trust, and review the `SKILL.md` of anything you install.

## Maintainer hardening

This repo follows these practices:

- **Branch protection on `main`** — no force-push, no deletion, linear history required
- **2FA required** on the maintainer's GitHub account
- **Private vulnerability reporting** enabled via GitHub Security Advisories
- **Commits to `SKILL.md` are reviewed line-by-line** — the skill file is effectively the "code" that runs on every user's machine

### Planned / recommended additions

- [ ] Signed commits (SSH-key or GPG-based) once signing is set up in the maintainer's local environment
- [ ] Required CODEOWNERS review for changes under `plugins/`
- [ ] Automated scanning for secret leaks on push

## Reporting a vulnerability

If you find a security issue — a way a malicious commit to this repo could compromise users, a parsing flaw in the skill that could be exploited via crafted input, or anything similar — **please do not open a public issue**.

Report it privately via GitHub's built-in flow:

**[Report a vulnerability →](https://github.com/pittaaron/claude-code-plugins/security/advisories/new)**

Please include:
- The affected file and line(s)
- A proof-of-concept if possible
- Your estimate of severity and affected user population

You'll get an acknowledgment within 72 hours. Fixes for confirmed vulnerabilities will be published in a new release with credit (if you want it).

## Out of scope

- Breakage due to claude.ai changing its internal API — this is expected and tracked as normal maintenance, not a security issue
- User configuration errors (e.g., granting blanket `Write(**)` permissions to Claude Code) — the skill requests scoped permissions only
- Bugs in the [claude-in-chrome MCP extension](https://claude.ai/chrome) — please report those directly to Anthropic
