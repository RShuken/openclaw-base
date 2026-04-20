# Adversarial Review of Skill Audit 2026-04-19

**Reviewed by:** Critical Analyst (adversarial spot-check)
**Reviewed at:** 2026-04-19
**Rows sampled (deterministic):** 5, 11, 17, 23, 29, 35, 41, 47, 53, 59
**Mapping:**
- Row 5 = audit #54 `install/setup-macos.md`
- Row 11 = audit #4 `deploy-prompt-guide.md`
- Row 17 = audit #10 `deploy-db-backups.md`
- Row 23 = audit #16 `deploy-mission-control.md`
- Row 29 = audit #22 `deploy-meeting-prep.md`
- Row 35 = audit #28 `deploy-himalaya-email.md`
- Row 41 = audit #34 `deploy-video-gen.md`
- Row 47 = audit #40 `deploy-tiktok-account-setup.md`
- Row 53 = audit #46 `deploy-health-monitoring.md`
- Row 59 = audit #52 `diagnose-cron.md`

---

## Per-row findings

### Row 5 — `install/setup-macos.md`
**Score in ledger:** RED
**Action in ledger:** DELETE (already executed per ledger L281, L290)

**Evidence verification:**
- File is absent from `install/` — the directory contains only `openclaw-install.md`. Deletion confirmed.
- A copy survives at `skills/_archive/setup-macos.md` (not the originally-cited `rel-hq/...` path). Ledger L290 says "Content preserved in `rel-hq/repos/rel-agent-ARCHIVED-2026-04-05/skills/setup-*.md`" — that path was not verified in this review, but the file does exist under `skills/_archive/`, so the work is recoverable either way.
- The audit's core claim — that the file taught OAC enrollment, not OpenClaw install — is plausible and consistent with the operator's standing guidance that OAC-specific content belongs elsewhere. I cannot verify the specific L43-44 Node 20 NodeSource claim for `setup-ubuntu-gce.md` because that row's action was also deletion (I sampled only macOS for Row 5). That claim should be spot-checked separately if it drives the DELETE verdict for the Ubuntu file.

**Things Claude missed:**
- The ledger says "Content preserved in `rel-hq/...`" but an archive copy at `skills/_archive/setup-macos.md` already existed *inside* the repo. Claude didn't mention this — so the "preserved" framing is correct, but via a different path than cited.
- No "replacements" entry was added to the `## Replacements Tracking` section at ledger L63-65 even though three files were deleted. The section is still "(empty — populated as audit progresses)."

**Verdict on this row:** CONFIRMED (with one small doc fix).
**Recommended correction:** Update L290 to also mention `skills/_archive/setup-macos.md` as the in-repo preservation path. Log the three DELETEs in the Replacements Tracking section.

---

### Row 11 — `deploy-prompt-guide.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (light)

**Evidence verification:**
- L4 "OpenClaw Version: 2026.2.26": verified (exact match).
- L76 "Matt Berman's setup using Claude Opus 4.6": verified: *"recommended defaults below come from the reference implementation (Matt Berman's setup using Claude Opus 4.6, battle-tested across 26 systems)"*.
- L80 "Target model | Claude Opus 4.6": verified in the Adaptation Points table.
- L140 "Claude Opus 4.6 (default) | Claude Opus 4.6 tips": verified in the Primary Model → Include Section table.
- Claim "Universal Principles (L189-239) are model-agnostic and sharp": verified — L189-239 covers Principles 1-5, all evergreen.
- Claim "Model-Specific Tips for Claude Opus 4.6, DeepSeek V3.x, GPT 4.x/5.x, Gemini 2.x/3.x, and 'Other Models' (L247-294)": verified at L247, L257, L267, L277, L286. Line ranges are correct within a line or two.

**Things Claude missed:**
- **The entire Claude Opus 4.6 section (L247-255) is one model version behind.** Baseline §W lists `claude-opus-4-7` as the **current flagship** in April 2026, and §8 notes 2026.4.15 defaulted `opus` aliases to 4.7. Audit flags "Claude Opus 4.6 default" but the body text is written specifically about Opus 4.6's idiosyncrasies, not generically about Claude. Needs a content refresh, not just a global s/4.6/4.7/.
- **L140 row says "Claude Opus 4.6 (default)"** — the guide's "default" still names a legacy model. Ledger calls out the label but under-weights that this is what clients will be told to use.
- **L251 the prose** ("*Opus 4.6 is the most sensitive current model…*") contradicts the claim. On 2026.4.15 Opus 4.7 is "most current"; the research in §N also notes Opus 4.7 backlash. Neither nuance is captured.
- **L322 heredoc placeholder warning**: `cat > ... << 'GUIDE_EOF'` is a **quoted heredoc**, so `[MODEL_NAME]` gets written literally. The skill tells the operator to "replace `[MODEL_NAME]` in the first paragraph… before sending this command" — meaning the operator must hand-edit a 5.5KB heredoc before transmission. That's a real usability hazard and the `_authoring/_skill-authoring-guide.md` anti-patterns list explicitly warns against "quoted heredocs with variables." Audit didn't flag.
- **L324 says "guide exceeds 5.5KB. The 8KB session command limit applies":** baseline `feedback_oac_command_line_limit.md` says the *actual* macOS PTY `MAX_CANON` ceiling is ~1024 chars. The 5.5KB guide is well over that, which means the "8KB session command limit" advice may itself be wrong depending on the transport. This is a live-execution hazard, not just staleness.

**Verdict on this row:** UNDERSTATED — YELLOW is roughly right for the universal-principles content, but the Claude section drift and the quoted-heredoc usability bug push this closer to the YELLOW/ORANGE boundary. The "light rewrite" framing under-represents how much Claude-model content needs refreshing.

**Recommended correction:** Upgrade the recommendation to include:
1. Refresh the Opus 4.6 section to Opus 4.7 (with §8 `xhigh` reasoning note where relevant).
2. Flag the quoted-heredoc model-name substitution as an anti-pattern; switch to a two-step write (write generic body, then `sed -i` the placeholder).
3. Re-measure the 8KB claim against actual transport behavior on 2026.4.15.

---

### Row 17 — `deploy-db-backups.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (swap `openclaw telegram send` → `openclaw channels` or Bot API; use native cron)

**Evidence verification:**
- L4 "2026.2.26": verified.
- L6 "Blocked Commands: openclaw telegram send": verified exactly.
- L19 "Schedules hourly cron execution": verified (phrased as "Schedules hourly cron execution").
- Audit says "core logic (GPG encrypt + gog Drive upload + cron) is sound" — broadly true, but see issues below.

**Things Claude missed:**
- **L216-218 embeds `openclaw telegram send` INSIDE the generated script.** The ledger notes this command is blocked in the *skill's own doc block* (L6), but the same blocked command is written into `backup-databases.sh` at L216-218 as the failure-alert mechanism. When the operator deploys, the cron job will silently fail to alert on backup failures. The rewrite needs to replace *both* the outer doc reference AND the generated script body.
- **L273 `tar -C /` with `${MANIFEST/#\//}`**: this tars from filesystem root and strips the leading slash from absolute paths. Safe-ish, but `tar -C /` inside a script executed under cron is a foot-gun for SELinux/Gatekeeper environments. Not flagged.
- **L313-318 retention logic is broken.** The `tail -n +$((RETENTION * 2 + 1))` assumes *exactly* two files per backup (archive + manifest). If any stray file in the "OpenClaw Backups" folder matches `name contains "openclaw-backup-"` (e.g., a manual rename, a user upload), retention silently deletes the wrong files or misses deletions. Not a blocker but real.
- **L324-330 instructs the operator to hand-edit placeholders** (`WORKSPACE_PLACEHOLDER`, `RETENTION_PLACEHOLDER`, `TELEGRAM_GROUP_PLACEHOLDER`, `TELEGRAM_TOPIC_PLACEHOLDER`) inside a quoted heredoc. Same anti-pattern as Row 11; `_authoring/_skill-authoring-guide.md` anti-patterns call this out. Audit didn't flag.
- **L560 uses raw `crontab` for scheduling.** Audit recommends "use native cron" in the Action — but doesn't say how deep the change goes. Every one of L549-569 (pre-flight check, add, idempotency guard, duplicate removal, verification L653) would need rewriting. That's more work than "swap `openclaw telegram send` → `openclaw channels`" implies.
- **L677 troubleshooting row also references `openclaw telegram send`** (duplicating L6's "blocked" claim). The skill is self-contradictory: L6 says the command doesn't work, but L597 and L677 tell the operator to test with it.

**Verdict on this row:** UNDERSTATED — score stays YELLOW but "REWRITE (light)" is probably not enough. There are 3 separate locations where the blocked command appears (L6 header, L216-218 script body, L597/L677 troubleshooting), plus a broken retention algorithm, plus the placeholder-heredoc anti-pattern. Closer to YELLOW/ORANGE boundary.

**Recommended correction:** Re-label the Action as "REWRITE (medium)" and list all three `openclaw telegram send` occurrences that need to change, not just the L6 header.

---

### Row 23 — `deploy-mission-control.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (verify robsannaa repo exists/current; compare feature matrix against bundled Control UI; REPLACE-WITH-BUNDLED if equivalent)

**Evidence verification:**
- Ledger cites "L1 wraps `robsannaa/openclaw-mission-control`." Actually the repo link is on **L3**, not L1: `Install and configure [openclaw-mission-control](https://github.com/robsannaa/openclaw-mission-control)`. Minor line-number miss.
- Claim "our §P research saw `abhi1693/openclaw-mission-control`" — can't verify here, but the file itself points to `robsannaa`, so one of the two repos is the actual source.
- Claim "§8 notes 2026.4.18 shipped Control UI overhaul with presets/quick-create": verified in baseline §8 (*"Control UI settings overhaul + presets + quick-create"*).
- Claim "bundled `openclaw dashboard`" exists: verified in baseline §4C.7 (*"dashboard"* in the subcommand inventory).

**Things Claude missed:**
- **L50 pre-flight uses `openclaw gateway call health`.** This is an unusual invocation not documented in §4C.7's gateway surface (which lists `openclaw gateway status`). The "call health" form may or may not work on 2026.4.15. Not flagged.
- **L77 reads `openclaw config get gateway.auth.token`.** That path naming looks plausible but wasn't verified against baseline §4. If the config key is under a different path on 2026.4.15, the token-extraction one-liner silently yields an empty string and Option A/B both fail silently.
- **L71-86 Option A tells the operator to manually edit a launchd plist.** No instruction for reloading launchctl after editing. Edit-without-reload is a common source of "I changed the config and nothing happened" support tickets.
- **No mention of the 2026.4.18 `models.authStatus` API (60s cache)** or the screen-snapshot feature — the bundled dashboard now exposes these natively. If the recommendation becomes REPLACE-WITH-BUNDLED, this is part of the feature-matrix comparison.
- **No mention that `robsannaa/openclaw-mission-control` is a 3rd-party repo** of unknown maintenance velocity. Introducing an unreviewed external GitHub repo into every base install is the exact anti-pattern baseline §O calls out ("Don't install ClawHub skills without vetting source (341+ malicious skills Feb 2026)"). Dashboards aren't ClawHub skills, but the principle applies.

**Verdict on this row:** UNDERSTATED — security/supply-chain angle on a 3rd-party dashboard deserves more weight. Score probably stays YELLOW but the recommendation should lean harder toward REPLACE-WITH-BUNDLED.

**Recommended correction:** Add a supply-chain note: before preserving this skill, run at minimum `git log -n 20` and `gh repo view robsannaa/openclaw-mission-control --json stargazerCount,pushedAt,isArchived` to confirm the repo is alive. If not, REPLACE-WITH-BUNDLED is the safer action.

---

### Row 29 — `deploy-meeting-prep.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (Codex swap) or REPLACE-WITH-CLAWHUB

**Evidence verification:**
- L4 "2026.2.26", L5 "READY": verified.
- Claim "correctly uses launchd over crontab (crontab hangs on macOS 15 per §4C)": verified at L6 — *"All scheduling via launchd (NOT crontab, which hangs on macOS 15)."*
- Claim "Claude hardcoded — §BB violation": verified at L43 (`${ANTHROPIC_API_KEY}` variable) and L121 Adaptation Points default.
- Claim "raw API for Google Calendar + Notion + Tavily + Claude + Telegram": verified in L39-46 variable table.

**Things Claude missed — this is the big miss of the sample:**
- **L121 literally specifies `claude-sonnet-4-20250514`** as the default synthesis model: *"Claude claude-sonnet-4-20250514 via Anthropic API"*. Per baseline §X, `claude-sonnet-4-20250514` was **deprecated 2026-04-14** and **retires 2026-06-15**. Every client install using this skill today will break in **under 2 months**. This is a higher priority than "Claude hardcoded — §BB violation" and the audit row completely misses it. It should be Priority 1 (baseline §X priority label reserved for imminent retirements).
- **L39-46 variable table still labels Anthropic-only workflow** even though §BB mandates Codex-first default. The Codex-swap recommendation is right, but the audit doesn't flag that this is specifically the most recently-deprecated Sonnet snapshot, not just a "hardcoded Claude reference."
- **L47 `${TELEGRAM_CHAT_ID}` example `{{PRINCIPAL_CHAT_ID}}`** is a client-specific numeric ID (looks like {{PRINCIPAL_NAME}}'s Telegram user ID given the {{COMPANY_NAME}} email on L44). This is the same "client identifier leaked into a base skill" concern flagged elsewhere; audit doesn't call it out for meeting-prep.

**Verdict on this row:** UNDERSTATED. Score should be **YELLOW → ORANGE** because of the retiring-snapshot + {{FAMILY_NAME}} identifiers. The Action should include "URGENT: before 2026-06-15, replace `claude-sonnet-4-20250514`" — this is a hard deadline, not a nice-to-have.

**Recommended correction:** Upgrade score to ORANGE and add a P1 flag for the deprecated Sonnet snapshot. Scrub the engagement-specific defaults (L44 Google Calendar email, L47 Telegram chat ID) from the base skill before any rewrite ships.

---

### Row 35 — `deploy-himalaya-email.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (light — version header)

**Evidence verification:**
- L4 "2026.2.26": verified.
- L5 "READY": verified.
- L6 "Blocked Commands: (none -- uses Himalaya CLI, Node.js scripts, and launchd…)": verified.
- Claim "identical to {{FAMILY_NAME}} ref": not verified in this review (would need diff). Claim "Himalaya + Gmail App Passwords = permanent credentials (no 7-day OAuth expiry)": verified in L11.
- Claim "self-describes as replacing broken gog OAuth path": verified at L11 and L91-98.

**Things Claude missed:**
- **L17 declares scope as "{{AGENT_NAME}} AI executive assistant … on {{PRINCIPAL_NAME}}'s Mac Mini"** — this is again engagement-specific ({{AGENT_NAME}} + {{PRINCIPAL_NAME}} = {{COMPANY_NAME}}). Audit says "identical to {{FAMILY_NAME}} ref" but doesn't acknowledge that the *base* skill still has {{FAMILY_NAME}} framing in its Purpose section.
- **L40 variable `${EMAIL_ADDRESS}` example `edge@{{COMPANY_DOMAIN}}`**: client email leaked into base skill. Audit missed this.
- **L46 `${TELEGRAM_BOT_TOKEN}` described as "{{AGENT_NAME}}'s bot token"** — client-specific naming in base skill.
- **L62 "This skill is macOS-only (Mac Mini)"**: this is an undocumented platform constraint. Audit scoring assumes broad applicability; in reality the skill hard-declares macOS-only. Users with Windows/Ubuntu agents (see row 54/55/56 deletions) will find this doesn't apply. At minimum it should be labeled as macOS-only in the Compatibility block, not halfway down the file.
- **L89 says "Node.js v18+"** — baseline §1 requires OpenClaw to run on **Node 22.14+ (Node 24 recommended)**. The skill's Node check would pass on a system too old to run OpenClaw itself. Not wrong per se (Himalaya wrappers may work on Node 18), but inconsistent with platform minimums.

**Verdict on this row:** UNDERSTATED. Score probably stays YELLOW, but "light — version header" is not enough. The engagement-specific framing + the undeclared macOS-only constraint + the Node 18 version drift warrant "REWRITE (medium)."

**Recommended correction:** Add an explicit `Platforms: macOS only (launchd/Himalaya)` line to the Compatibility block. Generalize the Purpose section to not reference {{AGENT_NAME}}/{{PRINCIPAL_NAME}}. Bump Node requirement to match OpenClaw's >= 22.14.

---

### Row 41 — `deploy-video-gen.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (light — version header; verify Veo 3 API still current)

**Evidence verification:**
- L4 "2026.2.26", L5 "NEEDS AUDIT", L6 "(none identified -- no `openclaw` CLI commands referenced)": all verified.
- Claim "Gemini-based": verified at L86 (`GEMINI_API_KEY`) and L77 "Google Veo 3 (via Gemini API)".

**Things Claude missed:**
- **L30** claims *"Video generation requests are asynchronous and can take 30 seconds to 3 minutes to complete"* and recommends an HTTP client timeout of "at least 5 minutes." Combined with `feedback_oac_tmux_long_running.md` (the 10-min PTY timeout ceiling), any chained video-gen call via the session command API will fail if the job runs more than ~10 minutes. No mention of wrapping in `tmux` like the feedback memory prescribes.
- **L86-87** requires "Veo 3 API access approved (may require waitlist or specific API enablement in the Google AI console)" — unverified against Google's current API surface. The audit flags "verify Veo 3 API still current" but doesn't acknowledge Veo 3 has since been followed by Veo 3.1 / Gemini 3-family video (research needed).
- **L5 "NEEDS AUDIT" in the Compatibility block**: self-declared unverified status. Promoting this to YELLOW without an API-probe is generous. Either probe Veo 3 access, or keep it in "NEEDS AUDIT" limbo with no score upgrade.
- **No `openclaw` CLI refs is correct, but the skill hard-references `${WORKSPACE}/.env`** at L54 (`grep GEMINI_API_KEY ${WORKSPACE}/.env`). Baseline §2 says credentials should live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, NOT in `.env`. The `deploy-memory.md` row (row 12) explicitly praised flagging this as a "gotcha to preserve." Inconsistent scoring: memory got YELLOW with an "EXCELLENT gotcha" note, video-gen reading from `.env` gets YELLOW with no flag.

**Verdict on this row:** OVERSTATED in status promotion (NEEDS AUDIT → YELLOW without the actual audit) and inconsistent with the `.env`-vs-auth-profiles flag applied elsewhere.

**Recommended correction:** Either downgrade to ORANGE until Veo 3 access is probed, or add an explicit "verify Veo 3 availability + move credential read from `${WORKSPACE}/.env` to `auth-profiles.json` per §2" action.

---

### Row 47 — `deploy-tiktok-account-setup.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (light) or REPLACE-WITH-CLAWHUB

**Evidence verification:**
- L4 "2026.2.26", L5 "READY", L6 "(none)": verified.
- Claim "TikTok-specific workflow": verified (entire file is TikTok cookie bootstrap).

**Things Claude missed:**
- **L113-115** writes a cookies.txt file that **hardcodes the expiry `2147483647`** (Jan 19 2038, the 32-bit epoch ceiling). Also writes **both `sessionid` and `sessionid_ss` from the same user-provided value**. TikTok regenerates `sessionid_ss` server-side — reusing the user's sessionid as `sessionid_ss` may actually work, but it's undocumented and could trigger anti-bot flags. Audit didn't flag.
- **L118 note** points to "the base64 file transfer method from `_deploy-common.md`" — but `_deploy-common.md` was scored ORANGE with "major rewrite needed" (row #57). This skill depends on a method in a doc that's about to change. No cross-row dependency warning.
- **L126-150 Playwright verification script** — this will **run Playwright on every install**. Playwright is a ~400MB download with Chromium; requiring it solely to validate a cookie string is heavy and slow. A lightweight `curl` with the sessionid cookie to TikTok's own API (or even just `https://www.tiktok.com/foryou`) would do the same job in 200ms. Audit didn't flag.
- **No prerequisite guard on Playwright availability** — if Playwright isn't installed on the client machine, Step 2.2 fails with a confusing Python error rather than a clean "install Playwright first" message.
- **L86-98 "Why manual creation" section** contains valuable operational knowledge (TikTok anti-bot, why manual is reliable) that should survive any REPLACE-WITH-CLAWHUB recommendation. Audit doesn't flag this as a "preserve" item the way it did for row 13's Keychain incident.

**Verdict on this row:** UNDERSTATED. Score stays YELLOW but the Action should be more specific than "REWRITE (light)." The Playwright-for-cookie-check is heavy; the base64 dependency on a mid-rewrite doc creates coupling.

**Recommended correction:** Swap the Playwright verification for a curl probe. Flag L86-98 as "preserve verbatim" in any replacement path.

---

### Row 53 — `deploy-health-monitoring.md`
**Score in ledger:** ORANGE
**Action in ledger:** REWRITE (unblock; evaluate overlap with bundled `doctor`)

**Evidence verification:**
- L4 "2026.2.26": verified.
- L5 "BLOCKED": verified exactly.
- L6 "openclaw config get, openclaw config set, openclaw cron add, openclaw cron list": all four commands verified in the source.
- Claim "all 4 cmds available on 2026.4.15 per §3/§4C.7": verified against baseline (§3 lists cron subcommands including add/list; §4C.7 lists `config` as a top-level command).

**Things Claude missed:**
- **L9 "WARNING: BLOCKED"**: the skill self-quarantines. On 2026.4.15, this warning is **actively wrong and will dissuade operators from using a skill that now works.** Audit flags the ORANGE score but doesn't mention the top-of-file WARNING block is itself misleading to future readers. Removing L9 should be the #1 edit.
- **L87, L282, L306, L330, L381, L450** all use `openclaw cron list` / `openclaw cron add` / `openclaw config get` — all live on 2026.4.15, so the skill's own internal usage is consistent. Ironically, the L6 "Blocked" claim is contradicted by the skill's own body (which uses these commands as if they work). Audit didn't call out this self-contradiction.
- **L192** `printf '{}' > ${WORKSPACE}/memory/heartbeat-state.json` — clobbers the file if an older-process race is in progress. The `test -f` guard at L189 is supposed to prevent this, but both commands are chained via `||`, so on the happy path the file is only created once. OK — low severity.
- **L220-266 Phase 2 tells the operator to verify scripts "already exist in the workspace from the OpenClaw system"** but doesn't say where they're supposed to come from. The "If missing, the operator must create it" instruction gives a content spec but not the script itself. This is a fundamental skill-authoring gap: either ship the scripts inline, or tell the operator which skill previously should have installed them.
- **No mention that baseline §L recommends "heartbeat: cheap-checks-first, model only on alert"** — this skill fires three separate cron jobs (every 30 min, weekly, nightly). Baseline §O explicitly warns: "Don't fan out N cron jobs — collapse into heartbeat." The entire architectural approach contradicts community consensus. ORANGE score might justifiably be RED on architecture grounds alone — this is exactly the anti-pattern the community calls out.

**Verdict on this row:** UNDERSTATED on architecture. Score is plausibly RED (the N-cron-fan-out anti-pattern is a §O explicit red flag). If kept, the skill should be rewritten to use OpenClaw's native heartbeat mechanism, not multiple cron jobs.

**Recommended correction:** Upgrade action from "REWRITE + evaluate doctor overlap" to "REWRITE: collapse into a single heartbeat check per §L community pattern; delete the L9 WARNING: BLOCKED self-quarantine."

---

### Row 59 — `diagnose-cron.md`
**Score in ledger:** YELLOW
**Action in ledger:** REWRITE (shift from "manual diagnostics" to "use `openclaw cron status/runs` + when those don't suffice, fall back to these deeper checks")

**Evidence verification:**
- Claim "No Compatibility block": verified — the file opens with `# Diagnose & Fix Cron Jobs` then goes straight into Purpose/When-to-use. No version header.
- Claim "2026.4.15 added `openclaw cron status`, `openclaw cron runs` which cover much of this natively": verified in baseline §3.

**Things Claude missed — biggest miss of the sample:**
- **The skill ALREADY USES `openclaw cron status` and `openclaw cron runs`.** L128 calls `openclaw cron runs --id <jobId> --limit 10`, L131 calls `openclaw cron status`, L116-122 uses `openclaw cron add` with current flags (`--cron`, `--tz`, `--session isolated`, `--message`, `--announce --channel telegram --to`). The audit claim "much of this is now covered natively [by commands the skill doesn't use]" is backwards: the skill is among the *most up-to-date* files in the library. It was written against at least 2026.3.x, not 2026.2.26.
- The recommendation "shift from manual diagnostics to use `openclaw cron status/runs`" is based on a false premise. Those commands are already the primary diagnostic path. The skill is structured as "check version → check gateway → check webhook → LIST cron jobs → CHECK RUNS → TEST FIRE → CHECK PERSISTENCE" — that's the exact ordering a modern runbook should use.
- **L13-19 Known Issues Registry is gold.** Documents 7 specific cron bugs with affected-version ranges and fixes. This is the single most valuable content in the file and is evergreen. Audit doesn't single it out as preserve-worthy.
- **L220** incorrectly says "OpenClaw version | 2026.3.13". Baseline §1 says current stable is **2026.4.15**. That line is stale by one version — needs updating, but this is the *only* staleness issue in the whole file.
- **L213-221 Adaptation Points** hardcodes `<TELEGRAM_GROUP_ID> (Rel HQ)` and `156 (Sprint)` — internal Rel identifiers in a base-library diagnostic skill. Same leakage pattern as rows 29 and 35. Audit doesn't flag.

**Verdict on this row:** WRONG. The REWRITE recommendation is based on a misread of the file. This skill is already modern. It should be GREEN (possibly YELLOW only for the L220 version number and L214-215 Rel-specific identifiers). Action should be KEEP with a ~15-minute polish, not REWRITE.

**Recommended correction:** Downgrade score from YELLOW to GREEN (or at worst, lightest-possible YELLOW). Change Action to "KEEP — polish only (update L220 version to 2026.4.15; replace L214-215 Rel chat IDs with `<CLIENT_CHAT_ID>` / `<CLIENT_TOPIC_ID>` placeholders)." Surface the Known Issues Registry (L13-19) as a library-wide reference.

---

## Systemic patterns across the sample

**Bias toward YELLOW, not ORANGE.** The audit tends to score YELLOW when the underlying issue is architectural or time-bombed. Rows 29 (retiring Sonnet snapshot), 53 (violates §O cron fan-out anti-pattern), and arguably 17 (three separate bad-command occurrences) all deserve ORANGE. This is the *opposite* of the red flag the prompt warned against.

**Model-deprecation surface is under-sampled.** Of 10 rows, only 1 (row 59) happened to be free of Anthropic model references. But only 1 row's evidence column explicitly named a model ID that needs replacing (row 11's mention of "Claude Opus 4.6 label"). Row 29's `claude-sonnet-4-20250514` — a model that retires in under 2 months — was not surfaced at all despite being a concrete §X Priority-1 item.

**Client-identifier leakage is invisible to the audit.** Multiple rows (29 {{COMPANY_NAME}} email + Telegram chat ID, 35 {{AGENT_NAME}}/{{PRINCIPAL_NAME}} framing, 59 Rel chat IDs) carry {{FAMILY_NAME}}/Rel-specific defaults in base skills. Audit rows treat these as "light version-header rewrite" when they're actually privacy/cleanroom violations. Scrubbing should be a dedicated sweep, not a side effect.

**Cross-row dependencies aren't tracked.** Row 47 depends on a file (`_deploy-common.md`) that's in the middle of being rewritten (row 57). Row 53's script specs reference scripts that aren't installed by any other audited skill. No dependency graph → silent breakage during rewrites.

**"Write and verify against live" asymmetry.** When the skill was written recently (row 59 at 2026.3.13+) the audit under-credits it. When the skill was written older (row 53 at 2026.2.26) the audit over-indexes on the "BLOCKED" header without reading past it to see the body already uses the commands correctly. The header tells one story; the body often tells another.

---

## High-severity corrections needed (block the rewrite pass until fixed)

1. **Row 29 (`deploy-meeting-prep.md`) — Sonnet snapshot retires 2026-06-15.** L121 hardcodes `claude-sonnet-4-20250514`. This is a §X Priority-1 hard-deadline item and must be fixed before any client install using this skill crosses June 15. Upgrade ledger score to ORANGE and tag as P1.
2. **Row 59 (`diagnose-cron.md`) verdict is backwards.** The file already uses 2026.3.x+ cron commands. REWRITE recommendation should be KEEP-polish, not "shift paradigm." Fix the ledger row before any rewrite touches this file — otherwise we risk regressing a working diagnostic runbook.
3. **Row 53 (`deploy-health-monitoring.md`) is architecturally RED, not ORANGE.** The N-cron fan-out is explicitly listed as a §O community anti-pattern. Either collapse into a single heartbeat check (then YELLOW) or delete (RED). The ledger's ORANGE + "evaluate overlap" treatment papers over the fundamental design issue.
4. **Row 17 (`deploy-db-backups.md`) has 3 locations of `openclaw telegram send`, not 1.** L6 + L216-218 + L597 + L677. The "REWRITE (light)" framing will miss the L216-218 and L677 occurrences. Any rewrite must sweep all four.

## Low-severity polish

- Row 5: update L290 preservation path to include `skills/_archive/setup-macos.md`. Log the three DELETEs in the Replacements Tracking section (L63-65).
- Row 11: flag the quoted-heredoc placeholder anti-pattern at L322.
- Row 23: note that `robsannaa/openclaw-mission-control` is a 3rd-party repo needing supply-chain vetting per §O.
- Row 35: add explicit `Platforms: macOS only` to the Compatibility block; bump Node requirement from 18 to ≥22.14 per baseline §1.
- Row 41: resolve the `.env`-vs-`auth-profiles.json` inconsistency (row 12 got credit for flagging this exact issue; row 41 has the same bug and no flag).
- Row 47: swap Playwright verification for a curl probe; preserve the L86-98 "Why manual creation" passage under any replacement path.
