---
name: maestro-orchestrator
description: "This skill should be used when the user asks to 'coordinate agents', 'run a multi-step task', 'parallel execution', 'workflow orchestration', or 'emergency response'. Provides multi-agent coordination, parallel task execution, SDLC workflow management, and escalation handling."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: true
---

# Workflow Orchestration & Multi-Agent Coordination

Coordinate multi-step development workflows, manage parallel agent execution, handle dependency resolution between tasks, and run escalation protocols for critical issues.

## Standard Operating Procedures

### SOP-1: Full Development Cycle
```
1. Receive feature request
2. Parallel Phase 1:
   - Check/prepare database changes
   - Write tests (TDD)
   - Security review requirements
3. Sequential Phase 2:
   - Implement feature code
   - Apply migrations
4. Parallel Phase 3:
   - Run test suite
   - Security scan
5. Sequential Phase 4:
   - Build and deploy
   - Monitor for errors
6. Completion:
   - Update documentation
   - Commit and report
```

### SOP-2: Bug Fix Workflow
```
1. Receive bug report
2. Parallel Investigation:
   - Check device logs
   - Check Sentry issues
   - Check database state
3. Identify root cause
4. Write fix with test (TDD)
5. Parallel Verification:
   - Run tests
   - Security impact check
6. Deploy fix
7. Verify fix in production
```

### SOP-3: Emergency Response (P0)
```
1. ALERT: Critical issue detected
2. IMMEDIATE: Capture current state + Sentry details
3. INVESTIGATE: Check database, review code
4. FIX: Minimal fix implementation + targeted test
5. DEPLOY: Emergency deployment
6. VERIFY: Confirm resolution in production
```

## Workflow Templates

| Workflow | Steps |
|----------|-------|
| New Feature | Plan -> Database -> Tests -> Implement -> Build -> Deploy -> Monitor -> Document |
| Bug Fix | Investigate -> Reproduce -> Test -> Fix -> Verify -> Deploy -> Confirm |
| Schema Change | Design -> Migration -> Push -> Rebuild -> Test -> Deploy |
| Security Incident | Alert -> Assess -> Contain -> Fix -> Verify -> Report |

## Escalation Matrix

| Severity | Response Time | Action |
|----------|---------------|--------|
| P0 Critical | Immediate | Stop all work, fix now |
| P1 High | Within session | Prioritize next |
| P2 Medium | Next session | Add to backlog |
| P3 Low | When convenient | Document only |

## Parallel Agent Deployment

Use the Task tool with `subagent_type` to spawn parallel agents:
```
- "general-purpose": Full capability (read, write, edit, bash)
- "Explore": Read-only codebase exploration
- "Plan": Read-only planning and analysis
```

Ensure independent tasks run in parallel; dependent tasks run sequentially with explicit blockers via TaskUpdate.
