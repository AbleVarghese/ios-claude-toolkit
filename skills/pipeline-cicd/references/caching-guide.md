# CI Caching Guide

Strategies for caching iOS build dependencies in GitHub Actions to reduce build times and costs.

---

## SPM Dependency Caching (Highest Impact)

SPM resolution downloads and compiles all dependencies from scratch. On a clean build, this can take 3-8 minutes depending on dependency count.

```yaml
- name: Cache SPM Dependencies
  uses: actions/cache@v4
  with:
    path: |
      .build
      ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
    key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
    restore-keys: |
      ${{ runner.os }}-spm-
```

**Key strategy:**
- Primary key: hash of `Package.resolved` (exact dependency version match)
- Restore key: OS prefix only (partial match -- uses last cache even if versions differ, then SPM updates only what changed)

**Cache hit rate:** ~90% (changes only when dependencies are added/updated)
**Time saved:** 3-8 minutes per run
**Storage cost:** 200-500 MB per cache entry

**When to invalidate:**
- `Package.resolved` changes (automatic via key hash)
- Swift toolchain version changes (add Xcode version to key: `${{ runner.os }}-spm-xcode16.2-${{ hashFiles(...) }}`)
- Corrupted cache causing build failures (manually delete cache via GitHub API or UI)

---

## DerivedData Caching (Moderate Impact, Use with Caution)

Caching DerivedData speeds up incremental compilation by reusing previously compiled modules. However, stale DerivedData can cause phantom build failures.

```yaml
- name: Cache DerivedData
  if: github.event_name == 'pull_request'  # Only for PRs, NOT release builds
  uses: actions/cache@v4
  with:
    path: ~/Library/Developer/Xcode/DerivedData
    key: ${{ runner.os }}-dd-${{ hashFiles('**/*.swift', '**/project.yml') }}
    restore-keys: |
      ${{ runner.os }}-dd-
```

**Rules:**
- NEVER cache DerivedData for release/distribution builds (always clean build for distribution)
- ONLY cache for PR validation builds where speed matters and correctness has a retry path
- Include `project.yml` (or `.xcodeproj`) in the hash to invalidate when build settings change
- If a PR build fails mysteriously after a cache hit, retry with `actions/cache/restore` disabled

**Cache hit rate:** ~40-60% (invalidated by any Swift file change)
**Time saved:** 1-3 minutes per run
**Storage cost:** 500 MB - 2 GB per cache entry

**When NOT to use:**
- Release builds (stale artifacts in production)
- Nightly regression (want clean state for accurate results)
- After Xcode version changes (incompatible DerivedData format)

---

## Homebrew Caching (Low Effort, Consistent Savings)

Tool installation (SwiftLint, XcodeGen) via Homebrew takes 1-3 minutes. Caching avoids redownloading.

```yaml
- name: Cache Homebrew
  uses: actions/cache@v4
  with:
    path: |
      ~/Library/Caches/Homebrew
      /usr/local/Cellar/swiftlint
      /usr/local/Cellar/xcodegen
    key: ${{ runner.os }}-brew-${{ hashFiles('.github/workflows/ci.yml') }}
    restore-keys: |
      ${{ runner.os }}-brew-
```

**Key strategy:**
- Hash the workflow file itself (tool versions are specified there)
- When you update tool versions in the workflow, cache automatically invalidates

**Cache hit rate:** ~95% (changes only when CI config changes)
**Time saved:** 1-3 minutes per run
**Storage cost:** 50-200 MB per cache entry

---

## Cache Storage Limits

GitHub Actions provides:
- **10 GB** cache storage per repository (all branches combined)
- Caches not accessed in **7 days** are automatically evicted
- When storage limit is reached, oldest caches are evicted first

**Optimization strategies:**
1. Use specific cache keys (avoid storing large caches that rarely hit)
2. Add branch name to cache key for branch-specific caches if main and feature branches diverge significantly
3. Monitor cache usage: GitHub API `GET /repos/{owner}/{repo}/actions/cache/usage`
4. Periodically clean up stale caches: `gh actions-cache delete --all` or selective deletion

---

## Cache Key Design Patterns

### Versioned Cache (explicit invalidation)
```yaml
key: ${{ runner.os }}-spm-v2-${{ hashFiles('**/Package.resolved') }}
# Bump v2 -> v3 when you need to force cache invalidation
```

### Branch-Specific Cache
```yaml
key: ${{ runner.os }}-spm-${{ github.ref_name }}-${{ hashFiles('**/Package.resolved') }}
restore-keys: |
  ${{ runner.os }}-spm-${{ github.ref_name }}-
  ${{ runner.os }}-spm-main-
  ${{ runner.os }}-spm-
# Feature branch first, then main branch, then any branch
```

### Xcode-Version-Specific Cache
```yaml
key: ${{ runner.os }}-spm-xcode16.2-${{ hashFiles('**/Package.resolved') }}
# Prevents Xcode 16.0 cache from being used by Xcode 16.2 (incompatible)
```

---

## Cost-Benefit Summary

| Cache Type | Setup Effort | Hit Rate | Time Saved | Storage | Recommendation |
|---|---|---|---|---|---|
| SPM Dependencies | Low (5 lines) | ~90% | 3-8 min | 200-500 MB | Always use |
| Homebrew Tools | Low (5 lines) | ~95% | 1-3 min | 50-200 MB | Always use |
| DerivedData | Medium (careful) | ~40-60% | 1-3 min | 500 MB-2 GB | PR builds only |
| Xcode (pre-installed) | N/A | N/A | N/A | N/A | Use `setup-xcode` action |

**Total impact with SPM + Homebrew caching:** 4-11 minutes saved per run.
At 10 CI runs per day, that's 40-110 minutes of CI time saved daily.
At GitHub Actions macOS runner pricing (~$0.08/min), that's $3-9 saved daily, or $90-270/month.
