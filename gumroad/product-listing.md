# iOS Claude Code Toolkit

**21 battle-tested Claude Code skills for iOS development -- built across 75 real production sessions.**

---

You're using Claude Code for iOS development, but it doesn't know your stack. It doesn't know SwiftUI patterns, Xcode build commands, Stripe integration, or App Store submission requirements. Until now.

The iOS Claude Code Toolkit is a drop-in skills library that turns Claude Code into a senior iOS engineer. Every skill was built and refined across 75+ real development sessions shipping a production iOS app -- not in a weekend hackathon.

---

## What's Inside

### 21 skills across 10 categories:

**Development & Code Quality (2 skills)**
Build, test, and review iOS projects with architecture guardrails and TDD enforcement.

- `forge-development` -- TDD-first development with RED-GREEN-REFACTOR enforcement, hexagonal architecture guardrails, protocol-based DI patterns, Swift concurrency best practices
- `ios-review` -- Structured code review against 40+ iOS-specific criteria: memory leaks, retain cycles, main-thread violations, API misuse, accessibility gaps

**Security & Compliance (2 skills)**
Audit your app against OWASP Mobile Top 10 and run security-focused differential reviews on every PR.

- `cipher-security` -- OWASP Mobile Top 10 scanning, keychain audit, jailbreak detection review, SSL pinning validation, biometric auth verification, financial compliance checks
- `trailofbits-differential-review` -- Security-focused PR review modeled on Trail of Bits methodology with vulnerability pattern detection and injection vector analysis

**Payments (2 skills)**
Stripe CLI workflows, webhook forwarding, test card numbers, subscription lifecycle, Connect accounts, and Identity verification flows.

- `hermes-stripe` -- End-to-end Stripe integration (PaymentSheet, Connect onboarding, Identity KYC, webhook handling, test mode workflows)
- `stripe-best-practices` -- Stripe-specific patterns (idempotency keys, payment intent lifecycle, SCA/3DS handling, metadata conventions, error code resolution)

**Database (2 skills)**
Schema design, migration safety, query optimization, Row Level Security policies, and edge function patterns for Supabase PostgreSQL.

- `atlas-database` -- Migration authoring, RLS policy design, index optimization, schema evolution with backward compatibility and rollback scripts
- `supabase-postgres-best-practices` -- Query optimization, connection pooling, edge function patterns, realtime subscriptions, storage policies, PostgREST filter syntax

**DevOps & Deployment (3 skills)**
CI/CD pipelines, Fastlane automation, and App Store submission from archive to review.

- `velocity-fastlane` -- Fastlane lane authoring, match code signing, TestFlight distribution, screenshot automation, build number management
- `pipeline-cicd` -- GitHub Actions for Xcode: build matrices, test parallelization, artifact caching, simulator management, release automation
- `cupertino-apple` -- App Store submission checklist, Human Interface Guidelines validation, privacy nutrition labels, App Review guideline pre-check

**Monitoring & Debugging (3 skills)**
Device logs, crash investigation, and Sentry-aware code review that cross-references changes against known crash sites.

- `loki-logs` -- Device log streaming, Console.app filtering, OSLog structured logging, crash log symbolication
- `sentry-code-review` -- Sentry-aware PR review that cross-references changes against open issues and verifies error handling covers known crash sites
- `sentry-find-bugs` -- Sentry issue investigation: reads stack traces, identifies root causes, suggests fixes, verifies against historical crash patterns

**Performance (1 skill)**
Instruments profiling workflows for memory, CPU, scroll performance, and app launch optimization.

- `prometheus-performance` -- Time Profiler, Allocations, Leaks, Network profiling. Memory budget enforcement, CPU spike analysis, scroll performance diagnostics

**Accessibility (1 skill)**
WCAG 2.1 AA compliance for iOS with VoiceOver audit and Dynamic Type validation.

- `athena-accessibility` -- VoiceOver audit, Dynamic Type scaling, color contrast validation, accessibility label/hint authoring, trait assignment, rotor support

**Localization (1 skill)**
Multi-locale string management with pluralization, currency formatting, and RTL layout support.

- `babel-localization` -- `.xcstrings` catalog workflows, pluralization rules, currency/date/number formatting by region, RTL layout support, locale-specific testing

**Notifications & Orchestration (2 skills)**
Push notification setup and multi-agent coordination for complex tasks.

- `aegis-notifications` -- APNs configuration, push certificate management, local notification scheduling, content extensions, rich media payloads
- `maestro-orchestrator` -- Multi-agent coordination for large tasks: decomposes work into parallel assignments, manages dependencies, aggregates results

**Documentation (1 skill)**
Single source of truth documentation maintenance with automated updates.

- `documentation` -- Session protocols, CHANGELOG authoring, STATUS/BACKLOG updates, architecture decision records, DocC comment generation

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
The patterns are based on stable iOS development practices (SwiftUI, async/await, SPM, XCTest) and Claude Code's skill format. Both evolve slowly. If a major breaking change happens, updates will be available to existing customers within 12 months of purchase.

**Is this a subscription?**
No. One-time purchase. You own the files forever. Free updates for 12 months from purchase date.

**What if it doesn't work for my project?**
The skills use automatic project discovery, so they adapt to your setup. If something doesn't fit, the markdown files are easy to modify. And if you're genuinely unhappy, there's a 30-day refund policy.

**Do I need to use all 21 skills?**
No. Use what you need. Each skill is independent. Most developers start with `forge-development` (builds), `ios-review` (code review), and `heimdall-git` (commits), then expand from there.

---

**iOS Claude Code Toolkit** -- Stop teaching Claude about iOS every session. Start shipping.
