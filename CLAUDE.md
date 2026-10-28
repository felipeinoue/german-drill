# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Deutsch Drill": a hands-free English→German phrase trainer meant to run on an iPhone. It says an English phrase, waits for the user to say the German, says the German, then moves on. Rounds of N phrases (default 5) end with "Round finished, repeat or continue?".

The whole app is one static file, `index.html` (vanilla JS, no framework, no build step, no tests, no package manager). `texts.xlsx` is the source table (column A = German, column B = English; section titles and rows missing one side are skipped) and is untracked. It is not loaded at runtime. The 253 phrases are embedded in `index.html` as `const DEFAULT = [...]`.

## Run and deploy

- Run locally: open `index.html` in desktop Chrome. Speech works on `file://`, and the mic needs a user tap on Start.
- Syntax check without a browser: extract the `<script>` body and run it through `new Function(...)` in node.
- Always commit after making changes (do not wait to be asked). Pushing to deploy is still a separate step.
- Deploy: `git push` to `origin/main` (`felipeinoue/german-drill`). GitHub Pages serves the root at https://felipeinoue.github.io/german-drill/ and updates in about a minute. `gh` is installed at `C:\Program Files\GitHub CLI\gh.exe` (not on PATH in the Claude shell) and git uses its credentials.
- Replacing the built-in phrases: regenerate the `DEFAULT` array (`[{"de","en"}, ...]`) from `texts.xlsx` with openpyxl and paste it into `index.html`. Users can also upload a table in the app (SheetJS from cdnjs), stored in their browser only.

## Architecture (all inside the single `<script>`)

Sections are marked `/* ---------- name ---------- */`: speech out, mic level and noise gate, speech in, drill loop, rendering, list / jump, table loading, wiring.

- **Drill loop** (`loop(tok)`): async state machine `en → listen → de → (next)`, with `setPhase()` driving `render()`. Cancellation uses a `runId` token plus a `cancelers` set. Every await is followed by `if (!alive()) return`. `stopAll()` bumps `runId`, cancels speech and resolves pending listens.
- **Detecting that the user spoke** (`listen()`): two detectors race. The volume-based voice detector (`vad*` over a Web Audio analyser) is the primary one, because speech recognition does not work reliably on iPhone, in Safari or Chrome (both use WebKit). Speech recognition runs alongside it when available. Any speech counts, and the app never checks or corrects what was said. There is intentionally no timeout while waiting for the user. The only timers are the TTS safety timer in `speak()`, recognizer restarts and the end-pause check.
- **Phrase timer**: a `m:ss` readout on the status row (`timerRun` / `timerStop` / `updateTimer`) shows time spent on the current phrase. It resets when the phrase index changes, and is frozen while stopped (`stopAll()` calls `timerStop()`).
- **End of speech**: the user must be quiet for `S.endPause` seconds (setting, default 2 s) before the app moves on.
- **Round-end prompt**: recognized "repeat"/"continue" act immediately. Speech with no recognized word means continue, after a 2 s grace. Repeat/Continue buttons exist as a fallback.
- **Shuffle**: toggled by the "Shuffle" pill above the transport buttons on the main screen (`btnShuffle`, not in Settings, so it stays visible). `order` is the play order (indices into `phrases`; identity when `S.shuffle` is off, a Fisher-Yates shuffle otherwise, reshuffled when a pass wraps). `pos`, `batchStart` and `lastIdx` are positions in `order`, so read phrases through `ph(i)`. The phrase list and jump-by-number use table indices and map through `order.indexOf`.
- **State**: settings in `localStorage` `de.settings` (merged over defaults in `S`), progress in `de.pos` and play order in `de.order` (saved after every phrase), uploaded table in `de.phrases` / `de.raw` / `de.swapped`. All reads and writes are try/catch wrapped.
- **iOS constraints**: audio and mic need one user tap per session (`start()` primes speech synthesis synchronously before any `await`). The mic is never listened to while the app is speaking, so the app does not hear itself. While a page holds the mic open, iOS sends all audio to the phone speaker instead of the car / Bluetooth, so with the `car` setting (opt-in, default off, because Chrome for iOS shows a "Microphone access allowed" toast every time the mic reopens) `speak()` releases the mic (`closeMic()`) and the loop reopens it (`openMic()`) before each `listen()`; the analyser and its loop stay alive and read silence while closed. Pausing, or the page being hidden (`visibilitychange` calls `pause()`), also releases the mic and suspends the audio context (`releaseAudio()`), otherwise Siri cannot use the mic while the app is paused; `start()` reopens both. A diagnostics line on the card (Settings toggle) shows mic level, voiced ms and recognizer state, and is the first place to look for iPhone issues.

## Things to know

- The file doubles as an Artifact page: it began as a Claude Artifact (`https://claude.ai/artifact/8SsJKqs3J3JVVtcLmLtxLk`, private), but the Artifact sandbox blocks the microphone, so GitHub Pages is the real deployment. `index.html` is a full document. A skeleton-less copy was used for the Artifact and is not kept in the repo.
- Voice behavior can only be verified on a real phone. Desktop Chrome supports speech recognition and behaves differently from iOS.
