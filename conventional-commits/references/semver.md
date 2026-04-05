# SemVer 2.0.0 -- Semantic Versioning

> **When to read:** When the user asks about versioning, tags, releases, or "next version".

Reference: https://semver.org/

## Format

```
MAJOR.MINOR.PATCH[-pre-release][+build-metadata]
```

## Conventional Commits -> SemVer mapping

| Commit | Bump SemVer |
|--------|-------------|
| `fix:` | PATCH `x.y.Z+1` |
| `feat:` | MINOR `x.Y+1.0` |
| `BREAKING CHANGE` or `!` | MAJOR `X+1.0.0` |
| `docs:`, `chore:`, `ci:`, `style:`, `refactor:`, `test:` | no bump |
| `perf:` | PATCH (improvement without breakage) |

## Rules (MUST per spec)

- `0.y.z` = initial development, unstable API, anything can change
- `1.0.0` = first declared stable public API
- A published version can NEVER be modified -- publish a new one
- PATCH resets to 0 when MINOR increments
- MINOR and PATCH reset to 0 when MAJOR increments
- Pre-releases have lower precedence: `1.0.0-alpha < 1.0.0-beta < 1.0.0-rc.1 < 1.0.0`
- Build metadata (`+001`, `+sha.5114f85`) does not affect precedence

## Pre-release identifiers

```
1.0.0-alpha        # first internal test
1.0.0-alpha.1      # alpha iteration
1.0.0-beta         # functionally complete, unstable
1.0.0-rc.1         # release candidate
1.0.0              # stable release
```

## Resolve the next bump

When asked "what version next?":

1. Get the last tag: `git describe --tags --abbrev=0`
2. List commits since: `git log <last-tag>..HEAD --oneline`
3. Apply the highest rule found:
   - Any `BREAKING CHANGE` or `!` -> bump MAJOR
   - Any `feat:` -> bump MINOR
   - Any `fix:` or `perf:` -> bump PATCH
   - Otherwise -> no release, or maintenance patch

## Git tags

```bash
# Always annotated (not lightweight) -- preserves author, date, message
git tag -a v1.2.3 -m "release: v1.2.3"
git push origin v1.2.3

# List tags ordered by version
git tag --sort=-version:refname | head -10

# Latest SemVer tag
git describe --tags --abbrev=0

# Release candidate tag
git tag -a v2.0.0-rc.1 -m "release: v2.0.0-rc.1"
```

## CHANGELOG.md

Structure (Keep a Changelog format -- https://keepachangelog.com):

```markdown
# Changelog

## [Unreleased]

## [1.2.0] - 2026-04-05
### Added
- feat(auth): PKCE support for OAuth2 flow

### Fixed
- fix(cache): prevent stale reads after TTL reset

## [1.1.0] - 2026-02-14
...
```
