# Quick Start — 5 Minutes to Smarter Claude

## 1. Install the Plugin

```bash
claude plugins install ios-toolkit@<marketplace>
```

Or for local use, copy to your Claude plugins directory.

## 2. Set Up Your CLAUDE.md

Copy the template into your project root and fill in 5 values:

```bash
cp /path/to/ios-toolkit/templates/CLAUDE.md.template CLAUDE.md
```

Open `CLAUDE.md` and replace:

| Placeholder | Replace With |
|------------|-------------|
| `{PROJECT}` | Your Xcode project name (e.g., `MyApp`) |
| `{SCHEME}` | Your build scheme (e.g., `MyApp`) |
| `{BUNDLE_ID}` | Your bundle identifier (e.g., `com.company.myapp`) |
| `{TEAM_ID}` | Your Apple Team ID (e.g., `ABC123DEF4`) |
| `{SIMULATOR_ID}` | Output of `xcrun simctl list devices \| grep Booted` |

## 3. (Optional) Install Git Hooks

```bash
cp /path/to/ios-toolkit/templates/hooks/pre-commit .git/hooks/pre-commit
cp /path/to/ios-toolkit/templates/hooks/commit-msg .git/hooks/commit-msg
chmod +x .git/hooks/pre-commit .git/hooks/commit-msg
```

## 4. Start Claude Code

```bash
claude
```

Skills load automatically. No configuration needed.

## 5. Test It

Try these prompts to confirm skills are active:

| Prompt | Expected Skill |
|--------|---------------|
| "implement a transfer service with tests" | forge-development |
| "run a security scan on this project" | cipher-security |
| "commit this" | heimdall-git |
| "profile the app's memory usage" | prometheus-performance |
| "review this file" | ios-review |
| "add push notifications" | aegis-notifications |
| "prepare for App Store submission" | cupertino-apple |

Claude will follow phase gates automatically — structured output, verification steps, and self-audit included.

## Done

22 skills with behavioral enforcement are now active. See `README.md` for the full skill reference.
