# iOS Claude Toolkit

**21 battle-tested Claude Code skills for iOS development -- built across 75 real production sessions.**

---

You're using Claude Code for iOS development, but it doesn't know your stack. It doesn't know SwiftUI patterns, Xcode build commands, Stripe integration, or App Store submission requirements. Until now.

The iOS Claude Code Toolkit is a drop-in skills library that turns Claude Code into a senior iOS engineer. Every skill was built and refined across 75+ real development sessions shipping a production iOS app -- not in a weekend hackathon.

---

## What's Inside

### 21 skills across 7 categories:

**Development (4 skills)**
Build, test, lint, and deploy iOS projects with zero manual configuration. Auto-discovers your project structure, schemes, and simulators. Handles XcodeGen, SPM, CocoaPods, and raw xcodeproj setups.

- `forge-development` -- Full iOS build orchestration (clean builds, incremental builds, simulator management, scheme detection)
- `guardian-testing` -- Smart test selection that maps changed files to test classes, runs the smallest sufficient scope, and reports failures with context
- `phoenix-deployment` -- App Store submission pipeline (archive, export, upload, TestFlight distribution, release notes)
- `pipeline-cicd` -- CI/CD pipeline configuration with quality gates and automated validation

**Security (3 skills)**
Audit your app against OWASP Mobile Top 10, scan for leaked secrets, validate compliance posture, and run differential security reviews on every PR.

- `cipher-security` -- OWASP Mobile Top 10 audit with severity classification and remediation guidance
- `trailofbits-differential-review` -- Differential security review on every PR with vulnerability pattern detection
- `trailofbits-differential-review` -- Security-focused PR review modeled on Trail of Bits methodology

**Payments (2 skills)**
Stripe CLI workflows, webhook forwarding, test card numbers, subscription lifecycle testing, Connect account management, and Identity verification flows.

- `hermes-stripe` -- End-to-end Stripe integration (PaymentSheet, Connect onboarding, Identity KYC, webhook handling, test mode workflows)
- `stripe-best-practices` -- Stripe-specific patterns (idempotency keys, error handling, metadata conventions, migration strategies)

**Database (2 skills)**
Schema design, migration safety, query optimization, Row Level Security policy validation, and edge function development for Supabase PostgreSQL.

- `atlas-database` -- Schema design, migration generation, RLS policy authoring, index optimization, and data integrity checks
- `supabase-postgres-best-practices` -- Supabase-specific patterns (edge functions, realtime subscriptions, storage policies, auth hooks)

**API & Networking (2 skills)**
REST and GraphQL integration, retry logic, offline support, certificate pinning, and request/response logging.

- `apollo-api` -- API contract design, endpoint documentation, versioning strategy, and integration testing
- `loki-logs` -- Structured logging, crash reporting integration (Sentry), and production debugging workflows

**Testing & Quality (5 skills)**
TDD methodology, UI testing, snapshot testing, performance profiling, and code review automation.

- `superpowers:test-driven-development` -- RED-GREEN-REFACTOR workflow with test-first enforcement
- `superpowers:verification-before-completion` -- Quality gate enforcement before marking any task complete
- `superpowers:systematic-debugging` -- Root cause analysis methodology (bisect, isolate, reproduce, fix, verify)
- `superpowers:requesting-code-review` -- Structured code review requests with context, diff summary, and risk assessment
- `ios-review` -- iOS-specific code review (memory management, concurrency, accessibility, App Store guidelines)

**Git & Workflow (3 skills)**
Conventional commits, branch management, PR creation, conflict resolution, and changelog generation.

- `heimdall-git` -- Git workflow automation (smart commits, branch strategy, merge conflict resolution, changelog updates)
- `commit-commands:commit` -- One-word commit workflow (analyzes diff, generates conventional commit message, stages, commits)
- `commit-commands:commit-push-pr` -- Full pipeline from local changes to open pull request in one command

---

## What You Also Get

**CLAUDE.md Template**
A starter project configuration file for Claude Code. Drop it in your repo root and Claude immediately understands your project structure, build commands, architecture, test strategy, and coding standards. Based on the same template used across 75 production sessions.

**Git Hooks**
Pre-commit hooks for secrets scanning (prevents API keys from reaching your repo) and commit message validation (enforces conventional commit format). One setup script installs both.

**Workflow Tips Guide**
Hard-won lessons from 75 production sessions. Covers: how to structure prompts for iOS work, when to use plan mode vs. direct execution, how to handle Xcode build failures, simulator management tricks, and patterns for multi-file refactoring.

**Quick-Start Guide**
From download to fully configured in 5 minutes. Step-by-step instructions with verification at each stage.

---

## Who This Is For

**iOS developers using Claude Code (CLI or Cursor IDE)**
You're already using Claude Code but spending time re-explaining iOS patterns every session. These skills eliminate that overhead permanently.

**Teams adopting Claude Code for iOS projects**
Standardize how your team uses Claude Code. Every developer gets the same high-quality iOS workflows without building tribal knowledge from scratch.

**Solo founders building with AI-assisted development**
You're building fast and can't afford to debug Claude's iOS blind spots. These skills encode production-grade patterns so you ship faster with fewer surprises.

---

## What Makes This Different

**Not theoretical.**
Every skill was built and battle-tested across 75+ real development sessions shipping a production iOS app. The patterns survived real Xcode failures, real App Store reviews, real concurrency bugs, and real payment integration edge cases.

**Progressive disclosure.**
Each skill has a lean SKILL.md file (what Claude reads first) and a detailed references/ directory (what Claude consults when it needs depth). Claude gets context without getting overwhelmed.

**Project-agnostic.**
These skills work on any iOS project. They use automatic discovery -- detecting your project structure, build system, test targets, and dependencies at runtime. No hardcoded paths or assumptions.

**Methodology, not just files.**
You're not just getting skill definitions. You're getting the workflow methodology behind them -- how to structure sessions, when to use which skill, how to chain skills together for complex tasks, and the testing strategies that keep your codebase stable.

---

## Setup

```
Copy the skills/ folder into your project's .claude/skills/ directory.
Claude discovers them automatically. That's it.
```

Detailed instructions are in the included quick-start guide, but the short version is: unzip, copy one folder, start a Claude Code session. Skills are plain markdown files -- no compilation, no dependencies, no configuration servers.

---

## FAQ

**Do I need a specific Claude plan?**
Any plan that includes Claude Code. The skills are plain markdown files that Claude reads as context -- no API calls, no external services.

**Does this work with Cursor?**
Yes. Claude Code runs inside Cursor as the default AI assistant. Skills are discovered the same way whether you use the standalone CLI or Cursor's integrated terminal.

**Can I modify the skills?**
Yes. They're plain markdown files. Add your own patterns, remove sections you don't need, or use them as templates to build entirely new skills. You own the files.

**Will these skills become outdated?**
The patterns are based on stable iOS development practices (SwiftUI, async/await, SPM, XCTest) and Claude Code's skill format. Both evolve slowly. If a major breaking change happens, updates will be available to existing customers.

**Is this a subscription?**
No. One-time purchase. You own the files forever. Download, customize, and use on as many projects as you want.

**What if it doesn't work for my project?**
The skills use automatic project discovery, so they adapt to your setup. If something doesn't fit, the markdown files are easy to modify. And if you're genuinely unhappy, Gumroad offers a 30-day refund policy.

**Do I need to use all 21 skills?**
No. Use what you need. Each skill is independent. Most developers start with `forge-development` (builds), `guardian-testing` (tests), and `heimdall-git` (commits), then expand from there.

---

## What Developers Are Saying

*This section will be updated with testimonials after launch.*

---

**iOS Claude Toolkit** -- Stop teaching Claude about iOS every session. Start shipping.
