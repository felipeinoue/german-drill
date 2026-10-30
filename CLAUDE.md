# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Deutsch Drill": a hands-free English→German phrase trainer meant to run on an iPhone. It says an English phrase, waits for the user to say the German, says the German, then moves on. Rounds of N phrases (default 5) end with "Round finished, repeat or continue?".

The whole app is one static file, `index.html` (vanilla JS, no framework, no build step, no tests, no package manager). `texts.xlsx` is the source table (column A = German, column B = English; section titles and rows missing one side are skipped) and is untracked. It is not loaded at runtime. The 253 phrases are embedded in `index.html` as `const DEFAULT = [...]`.

## Run and deploy

- Run locally: open `index.html` in desktop Chrome. Speech works on `file://`, and the mic needs a user tap on Start.
- Syntax check without a browser: extract the `<script>` body and run it through `new Function(...)` in node.
- Always commit and push to `origin/main` after making changes (do not wait to be asked). The user tests on the iPhone through the GitHub Pages URL.
- After pushing, wait until the Pages build for that commit is live, then tell the user. Poll `gh api repos/felipeinoue/german-drill/pages/builds/latest --jq '.status + " " + .commit'` until it prints `built <HEAD sha>` (status `errored` means the build failed, so report that). Use a Monitor until-loop, about every 10 s, capped at 5 min.
- Deploy: `git push` to `origin/main` (`felipeinoue/german-drill`). GitHub Pages serves the root at https://felipeinoue.github.io/german-drill/ and updates in about a minute. `gh` is installed at `C:\Program Files\GitHub CLI\gh.exe` (not on PATH in the Claude shell) and git uses its credentials.
- Replacing the built-in phrases: regenerate the `DEFAULT` array (`[{"de","en"}, ...]`) from `texts.xlsx` with openpyxl and paste it into `index.html`. Users can also upload a table in the app (SheetJS from cdnjs), stored in their browser only.

## Architecture (all inside the single `<script>`)

Sections are marked `/* ---------- name ---------- */`: speech out, mic level and noise gate, speech in, drill loop, rendering, list / jump, table loading, wiring.

- **Drill loop** (`loop(tok)`): async state machine `en → listen → de → (next)`, with `setPhase()` driving `render()`. Cancellation uses a `runId` token plus a `cancelers` set. Every await is followed by `if (!alive()) return`. `stopAll()` bumps `runId`, cancels speech and resolves pending listens.
- **Detecting that the user spoke** (`listen()`): two detectors race. The volume-based voice detector (`vad*` over a Web Audio analyser) is the primary one, because speech recognition does not work reliably on iPhone, in Safari or Chrome (both use WebKit). Speech recognition runs alongside it when available; its results only count if the mic peak reached the threshold during that turn (`gateOpen()`, always on, no setting). Any speech counts, and the app never checks or corrects what was said. There is intentionally no timeout while waiting for the user. The only timers are the TTS safety timer in `speak()`, recognizer restarts and the end-pause check.
- **Session timer**: a clock button in the header (`#clock`, `sessionFrom`, `resetClock()`) counts `m:ss` / `h:mm:ss`. In Drive it starts at the first play (`start()` calls `startClock()`); in Focus it starts on entering Focus (`setMode()`); it keeps counting through pauses, and switching to Drive while not playing stops it at 0:00. It resets on tap, when the page becomes visible again (`visibilitychange`: screen unlock, app switch) and on `pageshow` from the back/forward cache; after a reset it restarts at once in Focus or while playing, otherwise it waits at 0:00 for the next play. It is never saved, so a reload starts at 0.
- **End of speech**: the user must be quiet for `S.endPause` seconds (setting, default 2 s) before the app moves on.
- **Round-end prompt**: recognized "repeat"/"continue" act immediately. Speech with no recognized word means continue, after a 2 s grace. Repeat/Continue buttons exist as a fallback.
- **Two modes** (`S.mode`, switched by the Drive | Focus control under the header, `setMode()`): *Drive* is the hands-free voice loop above, for use while doing something else (driving). *Focus* is for full-attention study: a scrolling list (`#flist`, `renderFocus()`) of all phrases in `order`, English shown and the German covered like a thumb over a table column. Tap a row to uncover or cover it; rows re-cover when scrolled out of view (IntersectionObserver); "↑ 15" scrolls back 15 phrases, "Cover all" re-covers everything. Silent unless the user taps the speaker button beside a German line (`.fsay`, speaks it with `speak()`); no grading, no timers, by the user's choice. The top visible row (table index) is saved in `de.focus`. Entering Focus pauses the drill (`pause()`), and the phrase list / jump-by-number scrolls the Focus list instead of restarting the drill (`goPhrase()`).
- **Shuffle**: toggled by the "Shuffle" pill above the transport buttons on the main screen (`btnShuffle`, not in Settings, so it stays visible). `order` is the play order (indices into `phrases`; identity when `S.shuffle` is off, a Fisher-Yates shuffle otherwise, reshuffled when a pass wraps). `pos`, `batchStart` and `lastIdx` are positions in `order`, so read phrases through `ph(i)`. The phrase list and jump-by-number use table indices and map through `order.indexOf`.
- **State**: settings in `localStorage` `de.settings` (merged over defaults in `S`; the "Reset settings to defaults" button restores `SETTING_DEFAULTS`: round size, pause, speed, threshold, diagnostics, but not shuffle, mode, table or progress), progress in `de.pos` and play order in `de.order` (saved after every phrase), uploaded table in `de.phrases` / `de.raw` / `de.swapped`. All reads and writes are try/catch wrapped.
- **iOS constraints**: audio and mic need one user tap per session (`start()` primes speech synthesis synchronously before any `await`). The mic is never listened to while the app is speaking, so the app does not hear itself. The voice plays on the phone speaker (web pages cannot reach CarPlay / Bluetooth on iOS; tried and dropped, there is no `car` setting). The mic stays open for the whole session and is never toggled per turn; `openMic()` before each `listen()` does nothing while it is open. Pausing, or the page being hidden (`visibilitychange` calls `pause()`), also releases the mic and suspends the audio context (`releaseAudio()`), otherwise Siri cannot use the mic while the app is paused; `start()` reopens both. A diagnostics line on the card (Settings toggle) shows mic level, voiced ms and recognizer state, and is the first place to look for iPhone issues.

- **Build marker / updates**: `const BUILD` in `index.html` is shown at the bottom of Settings and in the diagnostics line. It is the commit's date and time (`yymmdd.hhmmss.ffff`, local time, from `date '+%y%m%d.%H%M%S.%4N'`); set it in every commit. `checkUpdate()` (on load and whenever the page becomes visible) fetches the page with `cache: 'no-store'` and, if the server's `BUILD` differs and the drill is not running, reloads under `?v=<build>` to get past the phone's cached copy.

## Things to know

- The file doubles as an Artifact page: it began as a Claude Artifact (`https://claude.ai/artifact/8SsJKqs3J3JVVtcLmLtxLk`, private), but the Artifact sandbox blocks the microphone, so GitHub Pages is the real deployment. `index.html` is a full document. A skeleton-less copy was used for the Artifact and is not kept in the repo.
- Voice behavior can only be verified on a real phone. Desktop Chrome supports speech recognition and behaves differently from iOS.
