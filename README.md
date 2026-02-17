# iOS Claude Toolkit

21 battle-tested skills that make Claude Code an expert iOS/Swift development partner. Drop them into any Xcode project and Claude instantly gains specialized knowledge of TDD workflows, Stripe payments, Supabase migrations, OWASP security, App Store submission, and 16 more iOS-specific domains.

Built from 75+ sessions of production iOS development. Every skill encodes real SOPs, checklists, and decision trees -- not generic prompts.

---

## What's Included

### Development & Code Quality

| Skill | What It Does |
|-------|-------------|
| **forge-development** | TDD-first development with RED-GREEN-REFACTOR enforcement. Hexagonal architecture guardrails, protocol-based DI patterns, Swift concurrency best practices. Generates tests before implementation. |
| **ios-review** | Structured code review against 40+ iOS-specific criteria: memory leaks, retain cycles, main-thread violations, API misuse, accessibility gaps, and architecture drift. |

### Git & Version Control

| Skill | What It Does |
|-------|-------------|
| **heimdall-git** | Conventional commit generation, branch strategy enforcement, interactive rebase guidance, merge conflict resolution. Auto-detects change scope and generates `type(scope): description` messages. |

### Security & Compliance

| Skill | What It Does |
|-------|-------------|
| **cipher-security** | OWASP Mobile Top 10 scanning, keychain audit, jailbreak detection review, SSL pinning validation, biometric auth verification, data-at-rest encryption checks. |
| **trailofbits-differential-review** | Security-focused PR review modeled on Trail of Bits methodology. Identifies injection vectors, auth bypasses, cryptographic misuse, and unsafe deserialization in diffs. |

### Payments

| Skill | What It Does |
|-------|-------------|
| **hermes-stripe** | Stripe integration workflows: PaymentSheet setup, Connect onboarding, Identity verification, webhook handling, test card sequences, error code resolution. Knows the Stripe CLI. |
| **stripe-best-practices** | Stripe-specific patterns: idempotency keys, payment intent lifecycle, customer portal integration, subscription billing, SCA/3DS handling, metadata conventions. |

### Database

| Skill | What It Does |
|-------|-------------|
| **atlas-database** | Supabase/PostgreSQL migration authoring, RLS policy design, index optimization, schema evolution with backward compatibility. Generates migration SQL with rollback scripts. |
| **supabase-postgres-best-practices** | Query optimization, connection pooling, edge function patterns, realtime subscriptions, storage policies, auth helper usage, PostgREST filter syntax. |

### Performance

| Skill | What It Does |
|-------|-------------|
| **prometheus-performance** | Instruments profiling workflows (Time Profiler, Allocations, Leaks, Network). Memory budget enforcement, CPU spike analysis, scroll performance diagnostics, app launch optimization. |

### Accessibility

| Skill | What It Does |
|-------|-------------|
| **athena-accessibility** | WCAG 2.1 AA compliance for iOS. VoiceOver audit, Dynamic Type scaling, color contrast validation, accessibility label/hint authoring, trait assignment, rotor support. |

### Localization

| Skill | What It Does |
|-------|-------------|
| **babel-localization** | Multi-locale string management, `.xcstrings` catalog workflows, pluralization rules, currency/date/number formatting by region, RTL layout support, locale-specific testing. |

### DevOps & CI/CD

| Skill | What It Does |
|-------|-------------|
| **velocity-fastlane** | Fastlane lane authoring, match code signing, TestFlight distribution, screenshot automation, build number management, App Store upload. |
| **pipeline-cicd** | GitHub Actions for Xcode projects: build matrices, test parallelization, artifact caching, simulator management, code signing in CI, release automation. |

### Monitoring & Debugging

| Skill | What It Does |
|-------|-------------|
| **loki-logs** | Device log streaming and analysis. Console.app filtering, OSLog structured logging, crash log symbolication, unified logging best practices. |
| **sentry-code-review** | Sentry-aware PR review. Cross-references changes against open Sentry issues, verifies error handling covers known crash sites, checks breadcrumb coverage. |
| **sentry-find-bugs** | Sentry issue investigation. Reads stack traces, identifies root causes, suggests fixes with proper error context, verifies fixes against historical crash patterns. |

### Notifications

| Skill | What It Does |
|-------|-------------|
| **aegis-notifications** | APNs configuration, push certificate/key management, local notification scheduling, notification content extensions, rich media payloads, permission request UX. |

### App Store

| Skill | What It Does |
|-------|-------------|
| **cupertino-apple** | App Store submission checklist, Human Interface Guidelines validation, privacy nutrition label generation, App Review guideline pre-check, metadata optimization. |

### Documentation

| Skill | What It Does |
|-------|-------------|
| **documentation** | Documentation maintenance with single-source-of-truth enforcement. Session protocols, CHANGELOG authoring, status updates, architecture decision records, DocC comment generation. |

### Orchestration

| Skill | What It Does |
|-------|-------------|
| **maestro-orchestrator** | Multi-agent coordination for large tasks. Decomposes work into parallel agent assignments, manages task dependencies, aggregates results, enforces quality gates across agent output. |

---

## Requirements

- **Claude Code CLI** -- any plan (Free, Pro, or Max)
- **macOS** with Xcode installed
- **A Swift/SwiftUI project** (iOS 15+ recommended)

Optional but recommended:
- SwiftLint (for `forge-development` and `ios-review` linting gates)
- Fastlane (for `velocity-fastlane` lanes)
- Supabase CLI (for `atlas-database` migrations)
- Stripe CLI (for `hermes-stripe` webhook testing)

---

## Installation

### Method A: Copy (Recommended)

Copy the skill directories into your project's `.claude/skills/` folder:

```bash
# From your Xcode project root
cp -r /path/to/ios-claude-toolkit/skills/* .claude/skills/
```

This embeds the skills directly in your project. They travel with the repo and work for every team member using Claude Code.

### Method B: Symlinks

If you want a single source of truth across multiple projects:

```bash
# From your Xcode project root
mkdir -p .claude/skills

for s in /path/to/ios-claude-toolkit/skills/*/; do
  ln -s "$s" ".claude/skills/$(basename "$s")"
done
```

Symlinks point back to the toolkit directory. Updating the toolkit updates all projects.

### Verify Installation

```bash
ls .claude/skills/
```

You should see 21 skill directories. Start Claude Code and it will auto-discover them.

---

## How Skills Work

Claude Code automatically scans `.claude/skills/` on startup. Each skill contains a markdown file with:

- **Description** -- tells Claude when the skill applies
- **SOPs** -- step-by-step procedures for common workflows
- **Checklists** -- validation gates that must pass before completion
- **Decision trees** -- branching logic for context-dependent situations

When you ask Claude something like "review this code for security issues," it matches your intent against skill descriptions and loads the relevant skill (in this case, `cipher-security`). No slash commands or manual invocation needed -- Claude reads the skill and follows its procedures automatically.

Skills compose naturally. Asking "commit this after running tests" activates both `forge-development` (testing SOPs) and `heimdall-git` (commit conventions).

---

## Customization

Skills use placeholder tokens for project-specific values:

| Placeholder | Example Value | Where to Set |
|------------|--------------|-------------|
| `{PROJECT}` | `MyApp` | CLAUDE.md |
| `{SCHEME}` | `MyApp` | CLAUDE.md |
| `{BUNDLE_ID}` | `com.company.myapp` | CLAUDE.md |
| `{TEAM_ID}` | `ABC123DEF4` | CLAUDE.md |
| `{SIMULATOR_ID}` | `61310D3F-BFD8-...` | CLAUDE.md |

Claude reads your project's `CLAUDE.md` at startup and substitutes these values automatically. For best results, use the included `CLAUDE.md.template` as a starting point for your project -- it contains all the sections Claude looks for (build commands, architecture, test structure, dependencies).

You can also edit any skill's markdown directly. Add project-specific rules, remove sections that do not apply, or adjust thresholds. Skills are plain text -- there is nothing compiled or locked.

---

## Skill Categories at a Glance

```
ios-claude-toolkit/
  skills/
    forge-development/       # TDD + architecture
    ios-review/              # Code review
    heimdall-git/            # Git conventions
    cipher-security/         # OWASP security
    trailofbits-differential-review/  # PR security review
    hermes-stripe/           # Stripe workflows
    stripe-best-practices/   # Stripe patterns
    atlas-database/          # Supabase/Postgres
    supabase-postgres-best-practices/ # DB optimization
    prometheus-performance/  # Instruments profiling
    athena-accessibility/    # WCAG compliance
    babel-localization/      # Multi-locale
    velocity-fastlane/       # Fastlane + TestFlight
    pipeline-cicd/           # GitHub Actions
    loki-logs/               # Log analysis
    sentry-code-review/      # Sentry PR review
    sentry-find-bugs/        # Sentry debugging
    aegis-notifications/     # APNs + push
    cupertino-apple/         # App Store submission
    documentation/           # SSOT docs
    maestro-orchestrator/    # Multi-agent coordination
```

---

## FAQ

**Do skills slow down Claude?**
No. Skills are only loaded when Claude detects matching intent. Unused skills have zero overhead.

**Can I use these with Claude Code Web?**
Yes. Claude Code Web reads `.claude/skills/` the same way the CLI does, as long as the skills are committed to the repo.

**Do skills work with non-SwiftUI projects?**
Yes. Most skills are architecture-agnostic. UIKit, SwiftUI, and hybrid projects all work. A few SOPs reference SwiftUI-specific patterns but adapt to your codebase context.

**Can I add my own skills alongside these?**
Absolutely. Create a new directory in `.claude/skills/` with a markdown file inside. Claude discovers it automatically.

**What if two skills conflict?**
Skills are additive, not override-based. In rare cases where two skills give contradictory guidance, Claude follows the more specific one. You can also edit skills to remove overlap.

---

## License

Copyright (c) 2025-2026 Able Varghese (Tharayil). All rights reserved.
Toronto, Canada.

This product is proprietary intellectual property. Licensed to the original purchaser for use solely as a Claude Code development tool or artifact. Redistribution, reselling, sharing, sublicensing, or inclusion in other products is strictly prohibited. Use outside of Claude Code environments is not permitted. One purchase covers one developer, unlimited projects within Claude Code.

See `LICENSE` for full terms.

---

## Support

Questions or issues: ablevtinchy@gmail.com

Built for iOS developers who ship.
