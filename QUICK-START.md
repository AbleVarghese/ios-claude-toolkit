# Quick Start -- 5 Minutes to Smarter Claude

## 1. Install the Skills

From your Xcode project root:

```bash
mkdir -p .claude/skills
cp -r /path/to/ios-claude-toolkit/skills/* .claude/skills/
```

Verify: `ls .claude/skills/` should show 21 directories.

## 2. Set Up Your CLAUDE.md

Copy the template into your project root and fill in 5 values:

```bash
cp /path/to/ios-claude-toolkit/CLAUDE.md.template CLAUDE.md
```

Open `CLAUDE.md` and replace:

| Placeholder | Replace With |
|------------|-------------|
| `{PROJECT}` | Your Xcode project name (e.g., `MyApp`) |
| `{SCHEME}` | Your build scheme (e.g., `MyApp`) |
| `{BUNDLE_ID}` | Your bundle identifier (e.g., `com.company.myapp`) |
| `{TEAM_ID}` | Your Apple Team ID (e.g., `ABC123DEF4`) |
| `{SIMULATOR_ID}` | Output of `xcrun simctl list devices \| grep Booted` |

## 3. Start Claude Code

```bash
claude
```

Skills load automatically. No configuration needed.

## 4. Test It

Try these prompts to confirm skills are active:

| Prompt | Expected Skill |
|--------|---------------|
| "run a security scan on this project" | cipher-security |
| "commit this" | heimdall-git |
| "profile the app's memory usage" | prometheus-performance |
| "review this file" | ios-review |
| "add push notifications" | aegis-notifications |
| "prepare for App Store submission" | cupertino-apple |

Claude will follow the skill's SOPs automatically -- structured output, checklists, and validation gates included.

## Done

21 skills are now active across your project. See `README.md` for the full skill reference and customization options.
