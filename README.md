# iOS Claude Code Toolkit

22 battle-tested skills that make Claude Code an expert iOS/Swift development partner with behavioral enforcement. Drop them into any Xcode project and Claude instantly gains specialized knowledge of TDD workflows, Stripe payments, Supabase migrations, OWASP security, App Store submission, and 17 more iOS-specific domains — with phase gates, hard stops, and self-audit protocols that measurably change Claude's behavior.

Built from 75+ sessions of production iOS development. Every skill encodes real SOPs, checklists, decision trees, and enforcement mechanisms — not generic prompts.

---

## What's New in v2.0

- **Behavioral Enforcement**: Every skill now has Iron Laws, phase gates, hard stops, self-audit protocols, and counter-based escalation
- **22 Skills** (up from 21): Added `solve` — Structured Decision Engineering framework
- **44 Reference Files** (up from 10): Deep-dive content via progressive disclosure
- **Benchmarked**: 100% behavioral pass rate with skill vs 71.2% without (+28.8% gap)
- **Plugin Format**: Proper Claude Code plugin with validated `plugin.json`

---

## What's Included

### Development & Code Quality

| Skill | What It Does |
|-------|-------------|
| **forge-development** | TDD-first development with RED-GREEN-REFACTOR enforcement. Hexagonal architecture guardrails, protocol-based DI patterns, Swift 6 concurrency best practices. Phase gates block implementation without failing tests first. |
| **ios-review** | Structured code review against 40+ iOS-specific criteria: memory leaks, retain cycles, main-thread violations, API misuse, accessibility gaps, and architecture drift. 5-phase review protocol with file-level risk triage. |

### Git & Version Control

| Skill | What It Does |
|-------|-------------|
| **heimdall-git** | Conventional commit generation, branch strategy enforcement, merge conflict resolution. Branch verification before EVERY git operation. Hard stop on force push to main. |

### Security & Compliance

| Skill | What It Does |
|-------|-------------|
| **cipher-security** | OWASP Mobile Top 10 scanning, STRIDE threat analysis, keychain audit, jailbreak detection review, SSL pinning validation, KYC/AML/PCI DSS compliance. Secrets scan must be first. |
| **trailofbits-differential-review** | Security-focused PR review modeled on Trail of Bits methodology. Per-file risk triage, blast radius calculation, adversarial analysis, attack scenario modeling. |

### Payments

| Skill | What It Does |
|-------|-------------|
| **hermes-stripe** | Stripe CLI workflows: webhook forwarding, test event triggering, PaymentIntent creation, Connected Account management. Complete test card matrix. |
| **stripe-best-practices** | Stripe design patterns: CheckoutSessions, PaymentIntents, PaymentSheet, Connect marketplace, subscription billing, webhook security. Deprecated API detection. |

### Database

| Skill | What It Does |
|-------|-------------|
| **atlas-database** | Supabase/PostgreSQL migration authoring, RLS policy design, trigger development with NEW/OLD qualification enforcement. Migration safety gates block untested pushes to production. |
| **supabase-postgres-best-practices** | Query optimization, EXPLAIN ANALYZE guidance, connection pooling, RLS performance patterns, Edge Function database access, zero-downtime migration patterns. |

### Performance

| Skill | What It Does |
|-------|-------------|
| **prometheus-performance** | Instruments profiling workflows (Time Profiler, Allocations, Leaks, Energy Log). Measure-before-optimize enforcement. Performance thresholds: <3s cold launch, <100MB memory, 60fps. |

### Accessibility

| Skill | What It Does |
|-------|-------------|
| **athena-accessibility** | WCAG 2.1 AA compliance for iOS. VoiceOver audit, Dynamic Type scaling, color contrast validation, touch target sizing, SwiftUI accessibility modifier deep-dive. |

### Localization

| Skill | What It Does |
|-------|-------------|
| **babel-localization** | Multi-locale string management, String Catalogs, pluralization rules, currency/date/number formatting by region, RTL layout support, pseudo-localization testing. |

### DevOps & CI/CD

| Skill | What It Does |
|-------|-------------|
| **velocity-fastlane** | Fastlane lane authoring, Match code signing, TestFlight distribution, build number management. Test-before-deploy enforcement — no exceptions for hotfixes. |
| **pipeline-cicd** | GitHub Actions for Xcode projects: workflow recipes, SPM caching, quality gates, matrix builds. CI failure investigation gates block blind retries. |

### Monitoring & Debugging

| Skill | What It Does |
|-------|-------------|
| **loki-logs** | Device log streaming and analysis. os.Logger structured logging, crash log symbolication, error pattern matching. Evidence-first debugging — must read logs before hypothesizing. |
| **sentry-code-review** | Sentry bot PR review. Individual finding verification against actual code before acting. False positive detection with evidence. |
| **sentry-find-bugs** | Sentry issue investigation. Evidence-first methodology with hypothesis tracking. Fix attempt counter — escalates at 3 failed fixes. |

### Notifications

| Skill | What It Does |
|-------|-------------|
| **aegis-notifications** | APNs configuration, push/local/silent notifications, notification service extensions, deep linking. Permission-first enforcement — must check authorization before sending. |

### App Store

| Skill | What It Does |
|-------|-------------|
| **cupertino-apple** | App Store submission checklist, HIG validation, privacy nutrition label generation, rejection prevention (top 20 reasons). Financial app requirements. |

### Documentation

| Skill | What It Does |
|-------|-------------|
| **documentation** | SSOT document maintenance with honesty enforcement. Cannot write unverified metrics. Session-end protocol ensures docs reflect reality, not aspiration. |

### Orchestration & Decision-Making

| Skill | What It Does |
|-------|-------------|
| **maestro-orchestrator** | Multi-agent coordination for large tasks. Dependency analysis before spawning. File ownership verification prevents merge conflicts. |
| **solve** | Structured Decision Engineering — 10-step framework synthesizing OODA, MECE, A3, Working Backwards, Cynefin, and 6 more methodologies. |

---

## Requirements

- **Claude Code CLI** — any plan (Free, Pro, or Max)
- **macOS** with Xcode installed
- **A Swift/SwiftUI project** (iOS 15+ recommended)

Optional but recommended:
- SwiftLint (for `forge-development` and `ios-review` linting gates)
- Fastlane (for `velocity-fastlane` lanes)
- Supabase CLI (for `atlas-database` migrations)
- Stripe CLI (for `hermes-stripe` webhook testing)

---

## Installation

### Method A: Claude Code Plugin (Recommended)

```bash
claude plugins install ios-toolkit@<marketplace>
```

### Method B: Copy into Project

```bash
# From your Xcode project root
mkdir -p .claude/skills
cp -r /path/to/ios-toolkit/skills/* .claude/skills/
```

This embeds the skills directly in your project. They travel with the repo and work for every team member using Claude Code.

### Method C: Symlinks

```bash
# From your Xcode project root
mkdir -p .claude/skills
for s in /path/to/ios-toolkit/skills/*/; do
  ln -s "$s" ".claude/skills/$(basename "$s")"
done
```

Symlinks point back to the toolkit directory. Updating the toolkit updates all projects.

### Verify Installation

```bash
ls .claude/skills/
```

You should see 22 skill directories. Start Claude Code and it will auto-discover them.

---

## How Skills Work

Claude Code automatically scans `.claude/skills/` on startup. Each skill contains:

- **Iron Law** — non-negotiable discipline that cannot be bypassed
- **Phase Gates** — blocking checkpoints that prevent proceeding without verification
- **SOPs** — step-by-step procedures for common workflows
- **Hard Stops** — dangerous operations that require human approval
- **Rationalizations Table** — pre-built counterarguments for common shortcuts
- **Self-Audit** — model verifies its own compliance before declaring done
- **Cross-Skill References** — bidirectional links to related skills

When you ask Claude something like "review this code for security issues," it matches your intent against skill descriptions and loads the relevant skill. No slash commands or manual invocation needed — Claude reads the skill and follows its enforcement mechanisms automatically.

Skills compose naturally. Asking "implement this feature, review it, then commit" activates `forge-development`, `ios-review`, and `heimdall-git` in sequence.

---

## Behavioral Enforcement

What makes v2.0 different: every skill has **behavioral enforcement** that measurably changes Claude's output.

| Mechanism | What It Does | Example |
|-----------|-------------|---------|
| **Iron Law** | Non-negotiable rule | "No implementation without a failing test first" |
| **Phase Gates** | Block progress without proof | "Cannot proceed to GREEN until test fails in RED" |
| **Hard Stops** | Block dangerous operations | "Force push to main is BLOCKED — no path to compliance" |
| **Counter Escalation** | Track attempts, escalate at 3 | "Fix #3 failed — question the architecture" |
| **Rationalizations** | Pre-built pushback | "I'll add tests later" → "Later never comes" |
| **Self-Audit** | Model checks own work | "Did I follow the Iron Law? Cite evidence." |

### Benchmark Results

Tested across 4 iterations, 102+ test cases, 12 dimensions:

| Metric | Score |
|--------|-------|
| Behavioral pass rate (with skill) | **100%** |
| Behavioral pass rate (without skill) | 71.2% |
| Behavioral improvement gap | **+28.8%** |
| Challenge resistance (pushback on shortcuts) | **9.3/10** |
| Overall average across all dimensions | **8.9/10** |

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

Claude reads your project's `CLAUDE.md` at startup and substitutes these values automatically. Use the included `templates/CLAUDE.md.template` as a starting point.

You can also edit any skill's markdown directly. Add project-specific rules, remove sections that don't apply, or adjust thresholds. Skills are plain text — nothing compiled or locked.

---

## FAQ

**Do skills slow down Claude?**
No. Skills are only loaded when Claude detects matching intent. Unused skills have zero overhead.

**Can I use these with Claude Code Web?**
Yes. Claude Code Web reads `.claude/skills/` the same way the CLI does, as long as the skills are committed to the repo.

**Do skills work with non-SwiftUI projects?**
Yes. Most skills are architecture-agnostic. UIKit, SwiftUI, and hybrid projects all work.

**Can I add my own skills alongside these?**
Absolutely. Create a new directory in `.claude/skills/` with a `SKILL.md` file inside. Claude discovers it automatically.

**What if two skills conflict?**
Skills are additive. In rare cases where two skills give contradictory guidance, Claude follows the more specific one. Each skill has a "When NOT to Use" section to prevent misapplication.

**How is v2.0 different from v1.0?**
v2.0 adds behavioral enforcement (phase gates, hard stops, self-audit) that v1.0 lacked. Benchmarks show +28.8% behavioral improvement. Plus 33 new reference files and the `solve` decision engineering skill.

**What if Claude ignores a skill?**
Check the skill's description in `SKILL.md` frontmatter. Skills with pushy, intent-based descriptions trigger more reliably than keyword-only ones.

---

## License

Copyright (c) 2025-2026 Able Varghese (Tharayil). All rights reserved.
Toronto, Canada.

This product is proprietary intellectual property. Licensed to the original purchaser for use solely as a Claude Code development tool or artifact. Redistribution, reselling, sharing, sublicensing, or inclusion in other products is strictly prohibited. One purchase covers one developer, unlimited projects within Claude Code.

See `LICENSE` for full terms.

---

## Support

Questions or issues: ablevtinchy@gmail.com

Built for iOS developers who ship.
