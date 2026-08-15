# Post-v1 fixes — what changed, why, and where the knobs are

Written 2026-08-14, after the post-v1 audit landed as five batches on `main`. Read this when you
want to know why something is the way it is, or where to change it. Companion docs:
[MISSION.md](MISSION.md) (what Voix is), [BRANCHING.md](BRANCHING.md) (how work lands).

## How we got here

After v1 shipped (Phases 0–6 plus field fixes, commits `e1de4b5..0d81d60`), two audits ran: one
over the new practice/coaching code, one over inherited FluidVoice leftovers. Findings were
prioritized P0 (correctness) / P1 (first-five-minutes UX) / P2 (robustness) and fixed in five
batches, one PR each.

| Batch | Landed as | Content |
|---|---|---|
| A | [#6](https://github.com/stalinzbb/voix/pull/6) → `be7bdff` | Session-history data-loss fix, onboarding AX gate, Option+R default, permission-dialog strings, repo hygiene |
| B | [#11](https://github.com/stalinzbb/voix/pull/11) → `3181520` | Recorder lifecycle: collision guard, quit-safe persistence, cancel, cap, no-speech error |
| C | [#12](https://github.com/stalinzbb/voix/pull/12) → `76f88ef` | Contour charts on one time grid, voiced-only pitch buckets, honest history rows |
| D | [#13](https://github.com/stalinzbb/voix/pull/13) → `ec579c2` | Past feedback reachable, markdown block rendering |
| E | [#14](https://github.com/stalinzbb/voix/pull/14) → `8f51084` | Reachable leftovers disarmed, pace-norm dedup, store I/O tests, backup coverage |

PRs #7–#10 are the same content closed unmerged — see [The merge incident](#the-merge-incident).

## Batch A — data loss and first-run landmines

**Session-history wipe.** `PracticeSessionStore.load` decoded the whole array in one shot; any
single undecodable entry (schema drift, corruption) threw the entire history away. The
non-optional `quality` field added with the signal-quality gate already broke every pre-gate file.
Two-layer fix, both pinned by tests in `PracticeSessionStoreTests`:

- `PracticeSessionStore.decodeSessions(from:)` decodes entry-by-entry via a private `LossySession`
  wrapper — one bad entry drops that entry only.
- `DeliveryMetrics.init(from:)` (extension in `SpeechAnalysisService.swift`, kept in an extension
  so the memberwise init survives) decodes a missing `quality` as `.unknown` — pre-gate sessions
  keep their measurements, flagged as never-assessed.

**Policy going forward:** any new field added to `DeliveryMetrics` or `PracticeSession` must be
optional or get a decode default in that same custom init, or old session files lose data.

**Onboarding blocked on Accessibility** (`WelcomeView.isPermissionsReady`) — practice records via
the in-app button and never types into other apps, so AX now doesn't gate onboarding; the AX rows
are labelled "(Optional)" and the hotkey playground step no longer blocks Continue. AX is only
needed for the optional dictation hotkeys.

**Option+R rewrite hotkey** defaulted on (`SettingsStore.rewriteModeShortcutEnabled`) — a fresh
install owned a global AX text-rewrite chord in every app. Now defaults off; users can enable it.

**macOS permission dialogs** — the four usage strings in `Info.plist` said "FluidVoice"; rewritten.

**Repo hygiene** — deleted `.github/workflows/stale.yml` (auto-closed stacked PRs after 7 days,
directly sabotaging BRANCHING.md's strategy) and `.github/FUNDING.yml` (routed sponsorship
upstream). In-app support links (`SettingsView`, `AnalyticsPrivacyView`) now point at this fork's
GitHub issues instead of upstream's email.

## The test-infrastructure discovery (part of #6)

The whole suite began failing with *"The test runner hung before establishing connection"* — zero
tests run. Sampling the hung host showed the main thread parked in `SecItemCopyMatching`, twice
(once from `SettingsStore.init`'s key migration, once from `WelcomeView.body` via
`DictationAIPostProcessingGate.isProviderConfigured()`).

Cause: `KeychainService` still uses upstream's service name `com.fluidvoice.provider-api-keys`,
and this machine has a real key stored under it (the installed FluidVoice app coexists here). The
test build is intentionally unsigned, so it's never in the keychain item's ACL — macOS raises an
authorization prompt and the first keychain read blocks the main thread forever. It passed before
only because the item didn't exist; a missing item returns without prompting.

Fix: under XCTest (detected via `NSClassFromString("XCTestCase")`), every SecItem path in
`KeychainService` is bypassed and keys live in an in-memory dictionary. Guarded at the primitives
(`loadStoredKeys` / `saveStoredKeys` / `legacyProviderEntries` / `removeLegacyEntries`) so every
caller — present and future — is covered at once.

**If the suite ever hangs like this again:** run it, `pgrep` the `Voix Debug` host, `sample` it —
a keychain prompt from some new call path is the first suspect.

**Still open:** the shared service name itself is a product bug — Voix and an installed FluidVoice
read/write the same keys and can clobber each other. Fix = rename service + one-time migration
(copy old → new, don't delete old). Tracked as a spawned task.

## Batch B — recorder lifecycle

All in `PracticeSessionService` unless noted:

- `startPractice()` refuses when `asr.isRunning` (dictation hotkey held → practice used to steal
  that audio mid-utterance) and when `phase.isBusy` (no starting over an in-flight pipeline).
- Sessions are persisted **before** the coaching LLM call, with placeholder
  `coachingError: "Coaching did not complete."`, then swapped for the finished version via
  `PracticeSessionStore.replace(_:)`. A quit/crash/cancel during coaching keeps the recording's
  transcript and analysis.
- `cancelCoaching()` + a Cancel button in `PracticeView` during the coaching phase.
- Coaching makes **one** LLM attempt (`config.maxRetries = 1`); the old default of 3 × 180 s meant
  a dead endpoint locked the UI for ~9 minutes.
- 20-minute recording cap (`maximumRecordingSeconds`) auto-stops into a normal saved session —
  also keeps final transcription under Parakeet's ~24-minute model limit.
- A red dot on the sidebar Practice row (`ContentView`) while recording — the only indicator once
  the user navigates away; dictation hotkeys deliberately no-op while practice owns the recorder.
- Empty transcript now throws `CoachingError.noSpeechDetected` instead of masquerading as an
  AI-provider failure; the "configure a provider" tip in the feedback card only renders when no
  provider is actually configured.

## Batch C — charts tell the truth

- **One time grid.** `DeliveryMetrics` publishes a single `contourIntervalSeconds`, but energy
  frames (20 ms) and pitch windows (50 ms) have different native rates. The pitch track is now
  resampled onto the energy contour's grid (`SpeechAnalysisService.resamplePitch`), so the interval
  is exact for both at every duration. Before: recordings under ~12 s drew the pitch line at the
  wrong time scale, misaligned with its own pause bands.
- **Voiced-only buckets.** Resampling averages voiced windows only; a bucket with none stays 0.
  Plain averaging mixed unvoiced zeros in (150 Hz + 0 → 75 Hz, never spoken) and made speakers
  chart as more monotone than they are. Both behaviors pinned in `SpeechAnalysisServiceTests`.
- **History rows** (`PracticeView.rowMetricsSummary`) show "delivery not measured — unreliable
  recording" for gate-rejected sessions instead of "0 WPM · 0 pauses".

## Batch D — feedback reachable and readable

- Past-session rows are buttons; tapping loads the stored session back into the detail view via
  `PracticeSessionService.showSession(_:)` (guarded while recording/busy). Before, coaching text
  became unreachable the moment "New session" was pressed.
- `PracticeView.markdownBlocks` renders the coach's headings/bullets line-by-line with inline
  styling via `AttributedString`. `Text(LocalizedStringKey:)` dropped all block structure. If the
  coach ever emits tables or nested lists, swap this for a real markdown view.

## Batch E — leftovers disarmed, decisions made

- Menu bar: "Check for Updates…" (only outcome: an error dialog — Voix doesn't self-update) and
  "Rollback to Previous Version…" (picker listed upstream FluidVoice releases) removed from
  `MenuBarManager`. Handlers stay dormant.
- Change logs sidebar link removed (`ContentView`) — `ChangelogView` fetches altic-dev's releases.
  Restore with the fork's repo when Voix publishes releases (`ChangelogView.owner`).
- Command Mode settings row deleted (`SettingsView`) — same reasoning as its sidebar link;
  `CommandModeService`/`TerminalService` stay dormant, off by default.
- **Transcript fidelity decision** (`ASRService.stop`, practice path): custom-dictionary
  corrections stay (they fix ASR mishearings of names/jargon); spoken-punctuation rewriting is
  skipped (turning a spoken "period" into "." rewrites what was said and shifts the word count
  under the pace metrics). Fillers are kept (Batch-1 seam) — the analyzer counts them.
- Pace norm deduplicated: `PracticeSessionService.paceRange` / `paceRangeText` is the single
  source interpolated into both the coach prompt and `PracticeView.paceNote`. The two hand-written
  copies drifted once (fixed in `0d81d60`); now they can't.
- `PracticeSessionStore` takes an injectable directory; real file-I/O tests cover persistence
  across instances, 200-cap pruning (oldest dropped), corrupt-file recovery, and `replace()`
  reaching disk.
- Practice sessions are included in `BackupService` documents (optional `practiceSessions` field;
  old backups decode fine and leave sessions untouched on restore).
- MISSION.md's dormancy rule refined: **dormant is fine unless it's reachable or phones home.**

## The merge incident

Merging the original stacked PRs (#6–#10) in a rapid loop hit the exact trap BRANCHING.md
documents: when #6's branch was deleted, GitHub closed #7 and #9 before retargeting (closed
stacked PRs can't be reopened), and #8/#10 merged into their *parent branches* instead of `main`.
No content was lost — every batch's commit existed locally. Recovery per the doc: replacement PRs
based directly on `main` (#11–#14), with D and E rebased onto `main` first because GitHub reported
phantom conflicts between squashed history and original commits (the trees were byte-identical;
verified with an empty `git diff`). Orphaned branches deleted afterward.

**Lesson, now practiced:** merge stacked PRs one at a time, waiting for GitHub's retarget between
each — or base recovery/simple chains on `main` directly.

## Where to change things later

| Want to change | Go to |
|---|---|
| Coaching prompt wording/sections | `PracticeSessionService.coachPrompt` |
| Pace norm (100–130 WPM) | `PracticeSessionService.paceRange` — prompt and card label both follow |
| Coaching timeout / token budget / retries | `PracticeSessionService.requestCoaching` (`timeoutSeconds` 180, `maxTokens` 16 000, `maxRetries` 1) |
| Recording cap (20 min) | `PracticeSessionService.maximumRecordingSeconds` |
| Pause definition (≥ 0.5 s) | `SpeechAnalysisService.minPauseSeconds` |
| Silence/VAD sensitivity | `SpeechAnalysisService.absoluteSilenceFloorDb` (−50), `silenceMarginDb` (10) |
| Pitch detection range/voicing | `SpeechAnalysisService.minPitchHz` (60), `maxPitchHz` (500), `voicingThreshold` (0.3), `subharmonicTolerance` (0.9) |
| Quality-gate strictness | `SpeechAnalysisService.minSignalToNoiseDb` (12), `minSpeechLevelDb` (−45), `maxClippedSampleRatio` (0.005) |
| Chart resolution | `SpeechAnalysisService.maxContourPoints` (240) |
| Filler word list | Settings UI → `SettingsStore.fillerWords` (user-editable; analyzer counts against the same list dictation strips with) |
| Session cap (200) / storage | `PracticeSessionStore.maximumSessions`; file lives at `~/Library/Application Support/Voix/PracticeSessions.json` |
| What the coach receives | `DeliveryMetrics.summaryText()` — withholds numbers when `quality.isReliable` is false |
| Metrics schema | Add fields with a decode default in `DeliveryMetrics.init(from:)` or old sessions lose data |
| Test keychain behavior | `KeychainService.isRunningUnderXCTest` + the four guarded primitives |
| Practice transcript processing | `ASRService.stop`, the `forPracticeSession` branches (fillers kept, dictionary kept, spoken punctuation skipped, snapshot always retained) |

## Known gaps and deferred work

- **Keychain service shared with FluidVoice** — rename + migrate; tracked as a spawned task.
- **No automated test for the `forPracticeSession` ASR flag** (fillers survive, snapshot retained)
  — needs a live-model E2E like `DictationE2ETests`; manually verified.
- Everything in MISSION.md's deferral table still stands: word-aligned metrics (v2), trend charts
  (v2), per-session audio retention/playback (v1.1), ML VAD/YIN, editable coaching prompt.

## Running the tests

```bash
xcodebuild test -project Fluid.xcodeproj -scheme Fluid \
  -destination 'platform=macOS,arch=arm64' \
  -skip-testing:FluidDictationIntegrationTests/DictationE2ETests/testDictationEndToEnd_whisperTiny_transcribesFixture \
  CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO
```

180 tests as of `8f51084`. The Whisper-tiny E2E is skipped for the same nondeterminism reason CI
skips it; run it locally without the skip flag when touching the Whisper path.
