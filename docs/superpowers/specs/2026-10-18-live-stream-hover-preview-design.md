# Live Stream Hover Preview — Design

- **Status:** Draft for review
- **Date:** 2026-10-18
- **Repo:** `Razwerkz/chatterino2` (fork), branch `feature/live-hover-preview`, based on upstream `master` @ `7d8dba67`
- **Eventual goal:** upstream PR to `Chatterino/chatterino2` — after a discussion thread first (see §10)
- **Context:** personal use on KDE Wayland (CachyOS); daily-driver install is Chatterino 2.5.5 via Flatpak, which stays untouched. Development happens in a native build from this fork.

---

## 1. Intent

While chatting, chat frequently reacts to something happening on stream. Today the user must switch to a browser or otherwise leave Chatterino to see what happened. The desired experience: **hover the stream title in a split header and see the live stream, muted, in place of the current static thumbnail** — a few seconds of hovering answers "what is chat reacting to?", without leaving chat, and without audio ever surprising the user.

Everything in this document exists to serve that moment. Anything that adds complexity beyond it is out of scope for v1 (§6).

## 2. User-facing behavior (the spec)

1. Hover a live channel's split header → the existing hover popup appears **immediately** with today's content (stream title, category, uptime, thumbnail if enabled).
2. After the user **dwells ~300 ms**, and only if the feature is enabled and streamlink is resolvable, a live preview starts loading in the background.
3. When the **first video frame** is ready, the popup swaps the thumbnail area for the video — the title/category/uptime text stays. The popup grows to the fixed preview size (§5.2) and re-runs its on-screen bounds check. No second window is ever opened.
4. Until video is ready — and permanently, if the stream never loads (offline, error, streamlink missing, resolution timeout) — the popup looks exactly like today's tooltip. It never shows a black rectangle or an error dialog.
5. The preview is **muted by default**. A setting ("Stream preview audio", default off) can be enabled; flipping it applies **live** to a currently visible preview.
6. Cursor leaves the header → the popup hides and the preview stops (player stopped, streamlink process killed). No background downloading.
7. The popup is mouse-transparent: clicks, scrolls, drags, and clicks on the header all pass through exactly as today.
8. **With the feature off (default), behavior is indistinguishable from current behavior.**
9. Interaction with the existing "Show thumbnails of streams" setting: independent. If thumbnails are off, stage 1 of the popup is the text-only version of today's tooltip, and the video appears above the text as usual when ready.

## 3. Decisions already made

| Decision | Decided by | Note |
|---|---|---|
| Not a plugin — patched fork | Verified evidence, §4 | Plugin API physically cannot do any part of this |
| Fork + eventual upstream PR | User | Discussion thread before PR |
| New settings toggle, **off by default** | User | PR-friendliness |
| Thumbnail first, video replaces it when loaded | User | Solves the 1–3 s player-startup reality |
| Audio control lives in **Settings**, not in the preview | User | Consequence of tooltip being mouse-transparent |
| Audio default: muted | User | "muted by default" from the original ask |
| Audio setting applies live; no volume slider in v1 | Agent call, no objection | Volume later if missed |
| Video replaces the thumbnail area only; title/category/uptime text stays | Agent call (builder-check) | Matches "instead of a still thumbnail" |
| Preview video size fixed 480×270 in v1 | Agent call (builder-check) | Independent of thumbnail-size setting; setting later if wanted |
| `AudioOnly` quality preference → preview uses High mapping | Agent call (builder-check) | Preview must never resolve audio-only |
| Player starts only after ~300 ms dwell | Agent call, no objection | Quick mouse passes stay today-identical |
| Graceful silent degradation (no streamlink → tooltip as today) | Agent call, no objection | No error popups |
| Route B (external mpv window) as documented fallback only | Agent call, no objection | Used if Qt Multimedia pipeline disappoints |
| Design doc kept out of the eventual PR diff | Agent call | See §10 |

An **interactive preview** (click to pin, then buttons work) is a possible later feature; explicitly not v1. The one thing knowingly traded away from the original ask: a mute/unmute button inside the preview (impossible in a mouse-transparent hover tooltip).

## 4. Verified constraints (do not re-litigate)

- **Plugins cannot implement this, at any version.** The Lua plugin API (2.5.5 and master) exposes only: registered commands, one event (`EventType::CompletionRequested`), chat read/send, HTTP/WebSocket, images inside chat messages, read-only window/split inspection, context menus. There are **no hover/mouse events, no widget/window creation, no media/video type, and no process spawning** (`io.popen` is a throwing stub; `os` library not loaded). Evidence: `src/controllers/plugins/`, `src/controllers/plugins/api/`, `docs/wip-plugins.md`.
- **The existing hover tooltip cannot host video or interaction.** `TooltipWidget` is `WA_TransparentForMouseEvents`, hides on `leaveEvent`, and its entries support only image + text. Evidence: `src/widgets/TooltipWidget.cpp:66`, `src/widgets/splits/SplitHeader.cpp:1147–1184`.
- **Wayland-compatible window recipe exists and must be reused.** The current tooltip already works on the user's Wayland session. The preview popup must copy `TooltipWidget`'s recipe (BaseWindow flags `TopMost`/`DontFocus`/…, stays-on-top, translucent, mouse-transparent, `moveTo` positioning) rather than inventing new window semantics (`src/widgets/TooltipWidget.cpp:56–67`).
- **Playback stack available:** Qt6 Multimedia (+Widgets) and FFmpeg are present in the Flatpak runtime (`org.kde.Platform` 6.11). QtWebEngine and libmpv are **not** present → those routes are rejected; Qt Multimedia is the practical stack.
- **Streamlink integration already exists:** `src/util/StreamLink.cpp` — process creation including `flatpak-spawn --host` under Flatpak (`:66`), custom path + extra options settings, preferred-quality mapping (`openStreamlinkForChannel`).
- **Upstream posture:** the previous "attached player" PRs (#2805, #2854) were closed with "reach out / discuss first". PR strategy must include an early upstream discussion (§10).
- **Thumbnail mechanics:** base64 JPEG inside rich-text tooltip HTML; fetched on stream-status updates; 5-minute max age (`SplitHeader.cpp:961–999`, `formatTooltip` `:127`).

## 5. Technical approach (starting point — builder refines)

### 5.1 Trigger and lifecycle

- Hook: `SplitHeader::enterEvent` / `leaveEvent` (`SplitHeader.cpp:1147`, `:1180`), where the tooltip is shown/hidden today.
- When the feature is **on** and the channel is **live** and streamlink is resolvable: show the preview popup instead of `TooltipWidget`, reusing its position math. Otherwise: unchanged code path.
- Single active preview app-wide (only one cursor). Cancelable start: dwell (`QTimer` ~300 ms) → resolve → play. Leaving or moving off before the dwell timer fires cancels everything (no process ever spawned).
- **Hard stops that must kill streamlink + player:** leave event; split/channel change while shown; feature toggled off; window closed; app quit. Orphaned processes are an anti-goal (§7).
- Reruns must be excluded explicitly: Twitch reports reruns with `live == true`, so require `!rerun` from the stream status (`TwitchChannel::StreamStatus`).
- Repeated hovers on the same header: v1 accepts a full restart (~seconds). Keep-alive/caching is a later optimization if annoying in daily use.

### 5.2 Preview popup widget

- New widget, e.g. `src/widgets/StreamPreviewPopup.{hpp,cpp}`, sibling of `TooltipWidget` (not a modification of it — `TooltipWidget` is shared by emotes/links and stays untouched).
- Copies the TooltipWidget window recipe (§4). Contains:
  - **Stage 1:** the exact rich text of today's tooltip (reuse `tooltipText_` content, incl. base64 thumbnail) in a label styled like `TooltipEntryWidget`.
  - **Stage 2:** the same text minus the thumbnail image, with the video surface where the image was — the video replaces the image area only; the text stays visible.
- Video size: fixed 16:9, **480×270** in v1 — deliberately independent of the thumbnail-size setting (80/160/360 px wide would make the preview unusably small); a size setting is possible later. When video appears, the popup grows and re-runs its bounds check (`applyLastBoundsCheck`); same popup, same anchor preference.
- One popup per split header, same ownership pattern as the tooltip.

### 5.3 Stream sourcing

- **Primary:** run `streamlink --json <channel-url> <quality>` via the existing process plumbing (`createStreamlinkProcess`, `src/util/StreamLink.cpp:53`) → parse the JSON → feed the resulting HLS URL to `QMediaPlayer`.
  - Quality: reuse the `preferredQuality` mapping from `openStreamlinkForChannel`. Edge: if the user's setting is "Choose" (interactive picker), the preview uses the "High" mapping (`high,best`) — a preview cannot prompt.
  - Edge: if the user's `preferredQuality` is `AudioOnly`, the preview resolves with the High mapping instead — a preview must never produce audio-only playback behind what looks like a static popup.
  - `streamlinkOpts` (user's additional options) should be applied for consistency; note they are inert for playback in `--json` mode except quality-affecting flags.
  - Process creation must use a **silent** path: the existing helper pops `QErrorMessage` dialogs on failure (`showStreamlinkNotFoundError`); the preview must bypass those to honor §2.4.
  - Resolution timeout (e.g. ~10 s): kill and stay on thumbnail.
- **Spike required before committing:** exact `--json` output shape; HLS playback via Qt Multimedia (FFmpeg backend) incl. startup time and CPU; behavior when the live edge disconnects. If HLS-through-QMediaPlayer disappoints: fallback candidate is `streamlink --stdout` piped via `QMediaPlayer::setSourceDevice` (live, no seeking; different trade-offs).
- Both approaches must run the streamlink process so it can be cleanly killed on stop (including via `flatpak-spawn` semantics to be verified for process-tree kill).

### 5.4 Frame rendering

- **Recommended:** `QVideoSink` → custom paint into the popup widget. Keeps the popup a non-native, translucent-friendly surface; no native child window.
- Naive alternative: `QVideoWidget` (native child surface) — known risk on Wayland / with translucent top-level windows.
- Builder may spike both; choose by measured behavior on the user's machine (KDE Wayland).

### 5.5 Audio

- `QAudioOutput` attached to the player; muted by default.
- Driven by the "Stream preview audio" setting; live-applied via the settings signal (flip setting while previewing → immediate mute/unmute).

### 5.6 Settings

- Location: General page near "Show thumbnails of streams" (`GeneralPage.cpp`; 2.5.5 anchor ~`:831`). Keys under `/appearance/…`, following `Settings.hpp:496` style.
- `Enable live stream preview on hover` — bool, default **off**
- `Stream preview audio` — bool, default **off** (muted)
- No other settings in v1.

### 5.7 Degradation & error handling

- Streamlink missing / resolution failure / timeout → stay on thumbnail; debug-level log only.
- Offline channels → never start (existing offline tooltip path).
- Player error mid-preview → revert to thumbnail stage, stop player; no dialogs.

### 5.8 Build integration

- Add `Multimedia` (+ `MultimediaWidgets` only if `QVideoWidget` is chosen) to `find_package(Qt… COMPONENTS …)` (`CMakeLists.txt:143`) and link `Qt${MAJOR_QT_VERSION}::Multimedia` in `src/CMakeLists.txt` (link block ~`:962–992`).
- Upstream-friendliness: consider a CMake option compiling the feature out when Multimedia is unavailable, so the PR doesn't hard-require a new dependency everywhere. (Name/mechanism left to the builder.)
- Native dev build on Arch needs `qt6-multimedia`; follow `CONTRIBUTING.md`. Flatpak rebuild is not needed for development (native build runs fine alongside the Flatpak install).

## 6. Non-goals (v1)

- No interactive controls inside the preview (no buttons, no volume, no click actions). Pin-to-interact is a possible later feature.
- No VODs, clips, or reruns — live channels only.
- No previews in other tooltips/usercards/link previews.
- No volume slider; no per-channel preview configuration.
- No plugin-API work of any kind.
- No embedded player docked into the chat pane (old "attached player" scope).
- No preloading / background connections: previews start on hover only.

## 7. Anti-goals (would make this a failure even if it "works")

- Audio ever playing when the audio setting is off.
- Orphaned streamlink processes, or any downloading after the cursor leaves.
- Noticeable lag or CPU use for quick mouse passes; any regression when the feature is off.
- Interfering with header click/drag/scroll behavior (mouse-transparency must hold).
- A PR that ignores upstream's "discuss first" request or hard-requires Qt Multimedia for all builds.

## 8. Left to the builder (explicitly)

- Class layout/naming/ownership; how much of `tooltipText_`/tooltip internals to share.
- `--json` vs `--stdout` outcome and player-sourcing details; timeout/retry policy.
- Rendering pick (`QVideoSink` paint vs `QVideoWidget`) by evidence.
- Settings labels/placement polish.
- Threading/lifecycle details; what is unit-testable; CMake option shape.
- Milestone ordering (design suggests: 0) streamlink + external-player manual trial; 1) static popup replacing tooltip; 2) streamlink→player spike; 3) wire-up + mute; 4) lifecycle hardening; 5) settings + polish; 6) upstream discussion + PR prep).

## 9. Test plan outline

- **Off-state identity:** feature off → tooltips identical to today across channel types.
- **Live hover:** instant thumbnail; video swaps after dwell; muted by default; live audio toggle works.
- **Negatives:** streamlink missing; offline channel; quick pass-through hover (no process); hover away mid-load (no process left: `pgrep streamlink`); split/channel switch mid-preview; app quit.
- **Wayland sanity:** popup position/stacking, no focus stealing, clicks pass through.
- **Edges:** reruns never preview (explicit `!rerun` check); `AudioOnly` preference resolves video (High mapping).
- **Perf:** CPU while previewing vs idle; unchanged app startup.
- **Builds:** native Arch build; Flatpak rebuild optional.

## 10. Path to upstream PR

1. Working feature in the fork, behind the off-by-default setting.
2. **Upstream discussion first** (short post, drafted for the user's review): intent, demo capture/gif, dependencies story, off-by-default, perf notes; acknowledge #2805/#2854.
3. PR per `CONTRIBUTING.md` (changelog entry, formatting, targeted tests).
4. This design doc stays out of the PR: remove it from the branch tip before the PR (or park it on a docs branch). It lives in the fork only.

## 11. References

- Plugin capability audit: `src/controllers/plugins/` (master), `docs/wip-plugins.md`
- Tooltip mechanics: `src/widgets/TooltipWidget.{hpp,cpp}`, `src/widgets/splits/SplitHeader.cpp`
- Streamlink integration: `src/util/StreamLink.cpp`, `src/widgets/settingspages/ExternalToolsPage.cpp`
- Upstream prior art: Chatterino2 PRs #2805, #2854 (closed, "discuss first"); issue #2809 (Flatpak `flatpak-spawn`, fixed via PR #3178)
