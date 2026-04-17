# Mutation and chaos catalog

> **When to read:** Workflow Step 5.
> Use these as ready-made "break it on purpose" probes.

| Mutation | Surface | Expected signal | If silent = bug | Minimum level |
|---|---|---|---|---|
| Replace the image/artifact ref with a wrong tag | build/deploy | explicit pull/build/deploy failure | hidden fallback or wrong artifact pulled | T0/T2 |
| Remove or rename a secret | secrets | startup/auth failure with useful log | empty/default secret path accepted | T2/T3 |
| Corrupt quoting/escaping in generated config | config/render | validation or startup failure | malformed config accepted silently | T0/T2 |
| Change exposed port but not internal bind | network | external path fails deterministically | operator path hides wrong external behavior | T2/T3 |
| Revoke or rename a role/grant | identity | least-privilege path fails clearly | role contract drift stays invisible | T3 |
| Run deploy twice after partial cleanup | deploy/rerun | converges or fails loudly with cleanup clue | snowflake state or half-broken service | T2/T3 |
| Add stale cache/artifact or wrong generated manifest | build/runtime | checksum/signature or render check fails | stale state survives | T0/T2 |
| Reduce disk budget / fill volume | storage | warning or graceful degradation | startup/runtime failure appears too late | T4 |
| Inject latency or packet loss to a critical dependency | network/deps | timeout, retry, or graceful degradation | hang or false health | T4 |
| Shift clock / timezone | time | expiry/TTL/session checks fail predictably | time bugs invisible until prod | T4 |
| Remove a monitoring check or make it always green | observability | alerting/smoke path catches the gap | silent blind spot | T3/T4 |
| Keep scanner green but skip real runtime smoke | CI/release | release gate should still fail | false green pipeline | T4 |
| Break a doc command or wrapper alias | docs/ops | executable-doc or smoke fails | operators follow stale instructions | T0/T3 |
| Deny file ownership / labels / permissions | storage/runtime | startup or write path fails with clear reason | permission drift discovered late | T2/T4 |
| Force dependency absence at startup | deploy/runtime | startup order or fallback path is exercised | hidden hard dependency stays unknown | T4 |

## Selection rule

For every critical surface, pick at least:
- **one contract mutation**
- **one degraded-mode mutation**
- **one mutation** if rerun or operations matter

## Silent-pass rule

If a mutation that should break safety passes silently, treat it as a **Tier 3** finding unless blast radius is clearly minor.
