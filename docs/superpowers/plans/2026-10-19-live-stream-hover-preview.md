# Live Stream Hover Preview — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hovering a live stream's split header shows the live stream (muted by default) in place of the static thumbnail, behind an off-by-default setting, in the Razwerkz/chatterino2 fork.

**Architecture:** A per-split `StreamPreviewPopup` (a `BaseWindow` sibling of `TooltipWidget`, same window recipe) replaces the hover tooltip for live/rerun channels when enabled. It shows the existing tooltip rich text first ("stage 1"); after a ~300 ms dwell a silent `streamlink --stream-url` resolve (host process via existing StreamLink.cpp plumbing) feeds a `QMediaPlayer` + `QVideoSink`, and the first decoded frame swaps the thumbnail area for video ("stage 2"), with audio unmuted only while the video stage is visible.

**Tech Stack:** C++ (Qt 6 Widgets + Multimedia, FFmpeg backend), streamlink CLI (host), CMake, GoogleTest (`-DBUILD_TESTS=On`), KDE Wayland (CachyOS).

**Spec:** `docs/superpowers/specs/2026-10-18-live-stream-hover-preview-design.md` — the plan argues from the spec; read both.

## Global Constraints

- Qt 6 only (`MAJOR_QT_VERSION=6`); do not add Qt5 code paths.
- New settings default **OFF**: `streamPreviewEnabled` = `{"/stream-preview/enabled", false}`, `streamPreviewAudio` = `{"/stream-preview/audio", false}`.
- Audio rule: never audible unless the video stage is visible; unmute only at the stage swap; every stop path (`endPreview()`) calls `player_->stop()` first.
- Preview size = `thumbnailSizeStream` pixels: 1 → 80×45, 2 → 160×90, 3 → 360×203; Off (0) → fall back to 160×90.
- Streamer Mode enabled → preview never starts and a running preview stops immediately.
- Reruns preview like live streams (do not gate on the `rerun` flag); offline channels never preview.
- No dialogs or chat messages from any preview path; failures degrade silently to today's tooltip (debug log only).
- Stream URLs come from streamlink resolved on the host through the existing `src/util/StreamLink.cpp` plumbing (`flatpak-spawn --host` when Flatpak).
- Internal docs (`docs/superpowers/**`) stay out of the eventual PR diff; commit messages follow the repo's conventional style (`feat:`, `fix:`, `docs:`, `test:`, `chore:`).
- Window behavior copied from `TooltipWidget` (BaseWindow flags, translucency, mouse-transparency, `moveTo` bounds checking); no new window semantics.

## Review Focus

Failure modes the spec implies but that no unit test can reach; each is pinned by a manual verification step in the task named.

1. Quick pass over the header (< 300 ms dwell) — no streamlink process spawned, no video flash (Task 6).
2. Stream ends/goes offline mid-preview — video reverts to the thumbnail stage silently, no frozen frame, no dialog (Task 7).
3. Streamer Mode toggled ON mid-preview — preview and audio stop instantly, process killed (Task 7).
4. Channel switched / split closed / window closed mid-load or mid-play — no orphan `streamlink` process (`pgrep -af streamlink`) (Task 7).
5. App quit mid-load — no orphan process; note flatpak-spawn kill semantics are only exercised by a Flatpak build, which is out of scope here (Task 7 note, Task 8 checklist).

---

### Task 1: Milestone 0 — streamlink chain trial (no code)

**Files:**
- Create: `docs/superpowers/notes/2026-10-19-streamlink-trial.md`

**Interfaces:**
- Consumes: nothing (validates the user's environment before any code).
- Produces: recorded evidence that host streamlink can resolve and play Twitch live + rerun streams; a streamlink version number used by later tasks.

- [ ] **Step 1: Install streamlink on the host**

Run: `sudo pacman -S --needed streamlink`
Expected: exit 0. Then `streamlink --version` prints a version — record it.

- [ ] **Step 2: Point the Flatpak Chatterino's "Open in streamlink" at mpv with audio disabled**

In the Flatpak Chatterino (2.5.5): Settings → External Tools → Streamlink → "streamlink options" field, set exactly:
`--player mpv --player-args "--no-audio --ontop --geometry=480x270"`

- [ ] **Step 3: Live channel trial**

In the Flatpak app, right-click a split showing a live channel → "Open in streamlink".
Expected: a small mpv window appears **on the host** playing the stream, muted. Note approx. time from click to first frame.

- [ ] **Step 4: Rerun channel trial**

Same on a channel whose header shows "(rerun)".
Expected: same result as Step 3.

- [ ] **Step 5: Record and commit**

Write `docs/superpowers/notes/2026-10-19-streamlink-trial.md`: streamlink version, exact `streamlinkOpts` used, live result, rerun result, observed latency, any surprises (ads, auth prompts).
Then:
```bash
git add docs/superpowers/notes/2026-10-19-streamlink-trial.md
git commit -m "docs: streamlink chain trial results"
```

---

### Task 2: Settings and UI toggles

**Files:**
- Modify: `src/singletons/Settings.hpp` (after `thumbnailSizeStream`, line ~497)
- Modify: `src/widgets/settingspages/GeneralPage.cpp` (after the "Show thumbnails of streams" dropdown block, i.e. after its closing `});` at ~line 905, before `layout.addNavigationSpacing()`)
- Create: `tests/src/StreamPreview.cpp`
- Modify: `tests/CMakeLists.txt` (`test_SOURCES`, add the new test file)

**Interfaces:**
- Consumes: nothing.
- Produces: settings keys every later task reads — `getSettings()->streamPreviewEnabled` (bool, default false), `getSettings()->streamPreviewAudio` (bool, default false).

- [ ] **Step 1: Write the failing test**

```cpp
// tests/src/StreamPreview.cpp
#include "singletons/Settings.hpp"

#include "Test.hpp"

using namespace chatterino;

TEST(StreamPreview, SettingsDefaultOff)
{
    EXPECT_FALSE(getSettings()->streamPreviewEnabled.getValue());
    EXPECT_FALSE(getSettings()->streamPreviewAudio.getValue());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cmake -S . -B build-tests -DBUILD_TESTS=On && cmake --build build-tests -j$(nproc)` then `./build-tests/bin/chatterino-test --gtest_filter=StreamPreview.*`
Expected: FAIL — compile error, `streamPreviewEnabled` is not a member of `Settings`.

- [ ] **Step 3: Add the settings**

In `src/singletons/Settings.hpp`, directly after the `thumbnailSizeStream` line:

```cpp
BoolSetting streamPreviewEnabled = {"/stream-preview/enabled", false};
BoolSetting streamPreviewAudio = {"/stream-preview/audio", false};
```

In `src/widgets/settingspages/GeneralPage.cpp`, after the thumbnail dropdown's closing `});`:

```cpp
SettingWidget::checkbox("Live stream preview on hover", s.streamPreviewEnabled)
    ->addTo(layout);
SettingWidget::checkbox("Stream preview audio", s.streamPreviewAudio)
    ->addTo(layout);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test --gtest_filter=StreamPreview.*`
Expected: PASS (1 test).

- [ ] **Step 5: Commit**

```bash
git add src/singletons/Settings.hpp src/widgets/settingspages/GeneralPage.cpp tests/src/StreamPreview.cpp tests/CMakeLists.txt
git commit -m "feat: add off-by-default stream preview settings and toggles"
```

---

### Task 3: StreamTooltip — split the tooltip content into parts

**Files:**
- Create: `src/widgets/splits/StreamTooltip.hpp`, `src/widgets/splits/StreamTooltip.cpp`
- Modify: `src/widgets/splits/SplitHeader.cpp` (remove the `formatTooltip` free function at lines ~127–183; change the call site at ~line 999)
- Modify: `src/widgets/splits/SplitHeader.hpp` (add member `StreamTooltipParts tooltipParts_;`)
- Modify: `src/CMakeLists.txt` (`SOURCE_FILES`: add the two new files in the `widgets/splits/` block after `widgets/splits/SplitOverlay.hpp`, line ~883)
- Create: `tests/src/StreamTooltip.cpp`; Modify: `tests/CMakeLists.txt`

**Interfaces:**
- Consumes: `TwitchChannel::StreamStatus` (public struct, `src/providers/twitch/TwitchChannel.hpp` ~line 97).
- Produces: `StreamTooltipParts { QString topHtml; QString imageHtml; QString bottomHtml; QString fullHtml() const; }` and `formatStreamTooltip(const TwitchChannel::StreamStatus &status, const QString &thumbnailBase64, int thumbnailSizeSetting, bool hideViewerCountForStreamerMode)`. `fullHtml()` must equal today's `formatTooltip` output byte-for-byte; `imageHtml` is the segment the video replaces in stage 2.

- [ ] **Step 1: Write the failing tests**

```cpp
// tests/src/StreamTooltip.cpp
#include "widgets/splits/StreamTooltip.hpp"

#include "Test.hpp"

using namespace chatterino;

TEST(StreamTooltip, ComposeMatchesLegacy)
{
    TwitchChannel::StreamStatus s;
    s.title = "Hello";
    s.game = "Just Chatting";
    s.uptime = "1h 2m";
    s.viewerCount = 1234;

    auto parts = formatStreamTooltip(s, "AAAA", 2, false);
    EXPECT_EQ(parts.topHtml, "Hello<br><br>");
    EXPECT_EQ(parts.imageHtml,
              "<img src=\"data:image/jpg;base64, AAAA\"><br>");
    EXPECT_EQ(parts.bottomHtml, "Just Chatting<br>Live for 1h 2m with 1,234 viewers");
    EXPECT_EQ(parts.fullHtml(),
              "<p style=\"text-align: center;\">Hello<br><br>"
              "<img src=\"data:image/jpg;base64, AAAA\"><br>"
              "Just Chatting<br>Live for 1h 2m with 1,234 viewers</p>");
}

TEST(StreamTooltip, ThumbnailOffHasNoImage)
{
    TwitchChannel::StreamStatus s;
    s.uptime = "5m";
    auto parts = formatStreamTooltip(s, "AAAA", 0, false);
    EXPECT_TRUE(parts.imageHtml.isEmpty());
    EXPECT_FALSE(parts.fullHtml().contains("<img"));
}

TEST(StreamTooltip, ThumbnailFetchFailed)
{
    TwitchChannel::StreamStatus s;
    auto parts = formatStreamTooltip(s, QString(), 2, false);
    EXPECT_EQ(parts.imageHtml, "Couldn't fetch thumbnail<br>");
}

TEST(StreamTooltip, RerunUsesVodCasting)
{
    TwitchChannel::StreamStatus s;
    s.uptime = "2h";
    s.rerun = true;
    auto parts = formatStreamTooltip(s, QString(), 0, false);
    EXPECT_TRUE(parts.bottomHtml.contains("Vod-casting for 2h"));
}

TEST(StreamTooltip, StreamerModeHidesViewerData)
{
    TwitchChannel::StreamStatus s;
    s.uptime = "2h";
    auto parts = formatStreamTooltip(s, QString(), 0, true);
    EXPECT_EQ(parts.bottomHtml,
              "<span style=\"color: #808892;\">&lt;Streamer Mode&gt;</span>");
}
```

- [ ] **Step 2: Run to verify they fail**

Run: `cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test --gtest_filter=StreamTooltip.*`
Expected: FAIL — `StreamTooltip.hpp` not found.

- [ ] **Step 3: Implement the formatter**

Port the three lambdas from the old `formatTooltip` (SplitHeader.cpp:127–183) verbatim into `formatStreamTooltip`, threading the parameters:
- `topHtml` = title lambda output (`title.toHtmlEscaped() + "<br><br>"` or empty).
- `imageHtml` = the old `tooltip` lambda, with `thumbnailSizeSetting` as the int parameter and `thumbnailBase64` as the thumbnail string.
- `bottomHtml` = game lambda output followed by extraStreamData lambda output, with `hideViewerCountForStreamerMode` taking the place of the `getApp()->getStreamerMode()->isEnabled() && getSettings()->streamerModeHideViewerCountAndDuration` check.
- `fullHtml()` = `"<p style=\"text-align: center;\">" + topHtml + imageHtml + bottomHtml + "</p>"`.

- [ ] **Step 4: Switch SplitHeader to the formatter**

Delete `formatTooltip` from SplitHeader.cpp. In `updateChannelText()` at the live branch (~line 999):

```cpp
this->tooltipParts_ = formatStreamTooltip(
    *streamStatus, this->thumbnail_,
    getSettings()->thumbnailSizeStream.getValue(),
    getApp()->getStreamerMode()->isEnabled() &&
        getSettings()->streamerModeHideViewerCountAndDuration);
this->tooltipText_ = this->tooltipParts_.fullHtml();
```

`formatOfflineTooltip` stays where it is. Add the include and the `StreamTooltipParts tooltipParts_;` member.

- [ ] **Step 5: Run tests to verify they pass**

Run: `cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test --gtest_filter=StreamTooltip.*`
Expected: PASS (5 tests).

- [ ] **Step 6: Commit**

```bash
git add src/widgets/splits/StreamTooltip.hpp src/widgets/splits/StreamTooltip.cpp src/widgets/splits/SplitHeader.cpp src/widgets/splits/SplitHeader.hpp src/CMakeLists.txt tests/src/StreamTooltip.cpp tests/CMakeLists.txt
git commit -m "refactor: split stream tooltip content into testable parts"
```

---

### Task 4: StreamPreviewPopup (stage 1) and the hover wiring

**Files:**
- Create: `src/widgets/StreamPreviewPopup.hpp`, `src/widgets/StreamPreviewPopup.cpp`
- Modify: `src/widgets/splits/SplitHeader.hpp` (member `StreamPreviewPopup *previewPopup_`)
- Modify: `src/widgets/splits/SplitHeader.cpp` (constructor init list + `enterEvent` + `leaveEvent` + the `window->leaving` lambda at ~line 302)
- Modify: `src/CMakeLists.txt` (`SOURCE_FILES`: add the two new files adjacent to `widgets/TooltipWidget.cpp/.hpp`, lines ~670–673)

**Interfaces:**
- Consumes: `StreamTooltipParts` (Task 3); `getSettings()->streamPreviewEnabled` (Task 2); `isLive_` (existing member; true for live **and** rerun); `getApp()->getStreamerMode()->isEnabled()`.
- Produces: `class StreamPreviewPopup : public BaseWindow` with `explicit StreamPreviewPopup(BaseWidget *parent); void beginPreview(const QString &channelUrl, const StreamTooltipParts &parts); void endPreview(); bool isPreviewVisible() const;` — later tasks extend the internals; these signatures stay stable.

- [ ] **Step 1: Implement StreamPreviewPopup (stage 1 only)**

Constructor mirrors `TooltipWidget` (`src/widgets/TooltipWidget.cpp:56–72`): same BaseWindow flag set `{TopMost, DontFocus, DisableLayoutSave}`, same anonymous-namespace `tooltipParentFor` helper (copy it — returns `nullptr` on Windows, parent elsewhere), same stylesheet `"color: #fff; background: rgba(11, 11, 11, 0.8)"`, `Qt::WA_TranslucentBackground`, `Qt::WA_TransparentForMouseEvents`, `Qt::WindowStaysOnTopHint`. Layout: `QVBoxLayout` with `QLayout::SetFixedSize`, margins `(10, 5, 10, 5)`; one member `QLabel *stage1Label_` (word wrap on, `Qt::AlignHCenter`).

`beginPreview`: store `channelUrl_` and `parts_`, set `stage1Label_->setText(parts.fullHtml())`. `endPreview`: `hide()`. `isPreviewVisible`: `isVisible()`.

- [ ] **Step 2: Wire SplitHeader**

Constructor: add `previewPopup_(new StreamPreviewPopup(this))` to the init list (next to `tooltipWidget_`, line ~261).

`enterEvent` (line ~1147): before the existing tooltip block, compute

```cpp
const bool usePreview =
    getSettings()->streamPreviewEnabled.getValue() && this->isLive_ &&
    !getApp()->getStreamerMode()->isEnabled() && !this->tooltipText_.isEmpty();
```

When `usePreview`, call `previewPopup_->beginPreview("twitch.tv/" + this->split_->getChannel()->getName(), this->tooltipParts_)`, then position and show exactly like the tooltip block: `adjustSize()`; `pos = mapToGlobal(rect().bottomLeft()) + QPoint((width() - popup width) / 2, 1)`; `moveTo(pos, widgets::BoundsChecking::CursorPosition)`; `show()`. Keep the `#ifdef Q_OS_WIN` show-order dance from the tooltip block. Otherwise take the existing tooltip path unchanged.

`leaveEvent` (line ~1180): call `previewPopup_->endPreview()` before hiding the tooltip.
The `window->leaving` lambda (~line 302): also call `previewPopup_->endPreview()`.

- [ ] **Step 3: Build the app for the first time and run it**

Run:
```bash
sudo pacman -S --needed qt6-base qt6-tools boost-libs openssl qt6-imageformats qt6-svg boost libnotify rapidjson pkgconf cmake hunspell qt6-wayland
cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j$(nproc)
./build/bin/chatterino
```
Expected: app starts. (First configure downloads FetchContent deps; the `-DBUILD_TESTS=On` build dir already exists from Task 2.)

- [ ] **Step 4: Manual verification**

With the setting OFF (fresh config default): hover live, rerun, and offline channels — tooltip behavior must be identical to before (thumbnail, uptime text, offline text).
With "Live stream preview on hover" ON: hover a live channel → the popup shows exactly the tooltip content (title, thumbnail, game, viewer line); hover a rerun → same; hover an offline channel → tooltip only; move the mouse off the header → popup hides.

- [ ] **Step 5: Commit**

```bash
git add src/widgets/StreamPreviewPopup.hpp src/widgets/StreamPreviewPopup.cpp src/widgets/splits/SplitHeader.cpp src/widgets/splits/SplitHeader.hpp src/CMakeLists.txt
git commit -m "feat: add stage-1 stream preview popup on hover (gated)"
```

---

### Task 5: Streamlink resolver (spike, refactor, tests)

**Files:**
- Modify: `src/util/StreamLink.hpp`, `src/util/StreamLink.cpp`
- Create: `src/util/StreamPreviewResolver.hpp`, `src/util/StreamPreviewResolver.cpp`
- Modify: `src/CMakeLists.txt` (add both new files in the `util/` block)
- Modify: `tests/src/StreamPreview.cpp` (extend from Task 2)

**Interfaces:**
- Consumes: existing `StreamLink.cpp` internals (`getStreamlinkPath`, flatpak-spawn handling), `getSettings()->streamlinkOpts`, `StreamLinkPreferredQuality` (`Settings.hpp:81`: `Choose, Source, High, Medium, Low, AudioOnly`).
- Produces:
  - `QProcess *createStreamlinkProcess(bool silent = false, QObject *parent = nullptr);` (exported; `silent` skips the `showStreamlinkNotFoundError` dialog path)
  - `struct StreamlinkQualityArgs { QString quality; QString exclude; };`
  - `StreamlinkQualityArgs streamlinkQualityArgs(StreamLinkPreferredQuality preferred);` (exact current mapping)
  - `StreamlinkQualityArgs streamPreviewQualityArgs(StreamLinkPreferredQuality preferred);` (`Choose`/`AudioOnly` → High mapping; else `streamlinkQualityArgs(preferred)`)
  - `class StreamPreviewResolver : public QObject` with `void resolve(const QString &channelUrl, StreamLinkPreferredQuality preferred); void cancel(); static std::optional<QString> parseResolvedUrl(const QByteArray &stdout);` signals `void resolved(const QString &streamUrl); void failed();`

- [ ] **Step 1: Spike — capture real streamlink behaviour**

On the host, against a live channel (and a rerun if available), run:
```bash
time streamlink --stream-url twitch.tv/<channel> high,best --stream-sorting-excludes ">720p30"
streamlink --json twitch.tv/<channel> best
mpv --no-audio "$(streamlink --stream-url twitch.tv/<channel> high,best --stream-sorting-excludes '>720p30')"
```
Expected / decision rule: if `--stream-url` prints a single playable URL line (mpv plays it), implement the resolver against `--stream-url` and paste the captured stdout lines into the test file as fixture strings. If it does not (e.g. prints nothing, needs `--json`), switch the implementation to `--json` mode: parse with rapidjson, use the top-level `url` field, update `parseResolvedUrl` name/tests accordingly, and record the exact key shape here.

- [ ] **Step 2: Write the failing tests**

Extend `tests/src/StreamPreview.cpp`:

```cpp
TEST(StreamPreview, QualityArgsMatchLegacyMapping)
{
    using Q = StreamLinkPreferredQuality;
    EXPECT_EQ(streamlinkQualityArgs(Q::High).quality, "high,best");
    EXPECT_EQ(streamlinkQualityArgs(Q::High).exclude, ">720p30");
    EXPECT_EQ(streamlinkQualityArgs(Q::Medium).quality, "medium,best");
    EXPECT_EQ(streamlinkQualityArgs(Q::Low).quality, "low,best");
    EXPECT_EQ(streamlinkQualityArgs(Q::AudioOnly).quality, "audio,audio_only");
    EXPECT_EQ(streamlinkQualityArgs(Q::Source).quality, "best");
    EXPECT_TRUE(streamlinkQualityArgs(Q::Choose).quality.isEmpty());
    // Preview mapping: Choose and AudioOnly resolve to High
    EXPECT_EQ(streamPreviewQualityArgs(Q::Choose).quality, "high,best");
    EXPECT_EQ(streamPreviewQualityArgs(Q::AudioOnly).quality, "high,best");
    EXPECT_EQ(streamPreviewQualityArgs(Q::Low).quality, "low,best");
}

TEST(StreamPreview, ParseResolvedUrl)
{
    EXPECT_EQ(StreamPreviewResolver::parseResolvedUrl(
                  "https://foo.example/bar.m3u8\n"),
              std::optional<QString>("https://foo.example/bar.m3u8"));
    EXPECT_FALSE(StreamPreviewResolver::parseResolvedUrl("").has_value());
    EXPECT_FALSE(StreamPreviewResolver::parseResolvedUrl("error: no data\n")
                     .has_value());
}
```
(Fixture strings from the Step 1 capture where the spike chose `--json`; keep `parseResolvedUrl` as the public name either way.)

- [ ] **Step 3: Run to verify they fail**

Run: `cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test --gtest_filter=StreamPreview.*`
Expected: FAIL — `streamlinkQualityArgs` / `StreamPreviewResolver` not declared.

- [ ] **Step 4: Refactor StreamLink.cpp**

Move `createStreamlinkProcess` out of the anonymous namespace into `StreamLink.hpp` with the signature above; when `silent == true`, skip connecting the `QErrorMessage` branch (still `deleteLater()` on error/finished and still log via `qCWarning(chatterinoStreamlink)`). Keep `getStreamlinkPath()` internal. Add `streamlinkQualityArgs` with the exact mapping currently inline in `openStreamlinkForChannelOrUrl` (Choose → empty quality; High → `high,best` + `>720p30`; Medium → `medium,best` + `>540p30`; Low → `low,best` + `>360p30`; AudioOnly → `audio,audio_only`; Source → `best`), and rewrite `openStreamlinkForChannelOrUrl` to consume it with behavior unchanged (Choose still routes to `getStreamQualities` + `QualityPopup`).

- [ ] **Step 5: Implement StreamPreviewResolver**

`resolve()`: build arguments `{--stream-url}` (+ `--stream-sorting-excludes <exclude>` when non-empty) + `{channelUrl, quality}` + `QProcess::splitCommand(getSettings()->streamlinkOpts.getValue())`; start via `createStreamlinkProcess(/*silent=*/true, this)`. A single-shot `QTimer` of 10 000 ms kills the process and emits `failed()`. On `finished` with exit code 0: `parseResolvedUrl(stdout)` → `resolved(url)` else `failed()`; any other exit: `failed()`. `cancel()` sets a cancelled flag, kills the process, emits nothing (the `finished` handler must respect the flag).

`parseResolvedUrl`: scan the trimmed output line by line; return the first line starting with `http://` or `https://`; otherwise `std::nullopt` (robust to any informational lines streamlink may print around the URL).

- [ ] **Step 6: Run tests to verify they pass**

Run: `cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test --gtest_filter=StreamPreview.*`
Expected: PASS (3 tests).

- [ ] **Step 7: Commit**

```bash
git add src/util/StreamLink.hpp src/util/StreamLink.cpp src/util/StreamPreviewResolver.hpp src/util/StreamPreviewResolver.cpp src/CMakeLists.txt tests/src/StreamPreview.cpp
git commit -m "feat: silent streamlink resolver for stream preview"
```

---

### Task 6: Video stage and audio

**Files:**
- Modify: `src/widgets/StreamPreviewPopup.hpp`, `src/widgets/StreamPreviewPopup.cpp`
- Modify: `CMakeLists.txt` (add `Multimedia` to the `find_package(Qt${MAJOR_QT_VERSION} REQUIRED COMPONENTS ...)` list, lines 143–151)
- Modify: `src/CMakeLists.txt` (add `Qt${MAJOR_QT_VERSION}::Multimedia` to the `target_link_libraries(${LIBRARY_PROJECT} ...)` block at ~line 962, as PUBLIC so the test binary links it transitively)

**Interfaces:**
- Consumes: `StreamPreviewResolver` and `streamPreviewQualityArgs` (Task 5); `streamPreviewAudio` (Task 2); `parts_`, `channelUrl_`, `stage1Label_` (Task 4).
- Produces: internal pipeline members used by Task 7 — `dwellTimer_`, `resolver_`, `player_`, `audioOutput_`, `videoSink_`, `videoSurface_`, `topLabel_`, `bottomLabel_`, `bool videoStage_`, plus `void resetToStage1();` and `void ensurePipeline();`. Public API unchanged.

- [ ] **Step 1: CMake**

Add the `Multimedia` component and the link library as above (required dependency for the fork; optionality is deliberately deferred to Task 8).

- [ ] **Step 2: Implement the pipeline**

`ensurePipeline()` (called from `beginPreview`, once): create `dwellTimer_` (single-shot, 300 ms), `resolver_` (child of `this`), `player_`, `audioOutput_`, `videoSink_`, and the stage-2 widgets; connect:
- `dwellTimer_::timeout` → `resolver_->resolve(channelUrl_, getSettings()->preferredQuality.getEnum())`
- `resolver_::resolved(url)` → `player_->setSource(QUrl(url)); player_->play();` (audio stays muted)
- `resolver_::failed()` → nothing (stage 1 stays)
- `videoSink_::videoFrameChanged` → first frame: `stage1Label_->hide()`; `topLabel_->setText(parts_.topHtml); topLabel_->show();` `videoSurface_->show();` `bottomLabel_->setText(parts_.bottomHtml); bottomLabel_->show();` `videoStage_ = true;` `adjustSize();` `applyLastBoundsCheck();` then apply audio: `audioOutput_->setMuted(!getSettings()->streamPreviewAudio.getValue())`. Every frame (all frames): pass the frame's `toImage()` to `videoSurface_`.
- `streamPreviewAudio` setting signal (pajlada `SignalHolder` member `connections_`) → if `videoStage_`: `audioOutput_->setMuted(!getSettings()->streamPreviewAudio.getValue())`.

`videoSurface_`: a private `QWidget` subclass (in the .cpp) holding a `QImage`, `paintEvent` draws it scaled `KeepAspectRatio`, centered. Fixed size from `thumbnailSizeStream`: helper `QSize previewSizeFor(int setting)` → 1: `{80, 45}`, 2: `{160, 90}`, 3: `{360, 203}`, else `{160, 90}`.

`beginPreview`: `ensurePipeline()`, reset to stage 1 (`resetToStage1()`), set content, `dwellTimer_->start(300)`.
`endPreview`: `dwellTimer_->stop(); resolver_->cancel(); player_->stop(); audioOutput_->setMuted(true); resetToStage1(); hide();`.
`resetToStage1`: `videoStage_ = false`; stage-2 widgets hidden; `stage1Label_->show()`.

- [ ] **Step 3: Build and run**

Run: `cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo && cmake --build build -j$(nproc) && ./build/bin/chatterino`

- [ ] **Step 4: Manual verification (each observation is a pass/fail)**

1. Hover a live channel: popup shows thumbnail first; within ~1–5 s the thumbnail area becomes live video at thumbnail size; still image and text positions unchanged (no popup resize when video arrives).
2. Default (audio setting off): no sound.
3. Enable "Stream preview audio" while a preview is playing: sound starts; disable: silence immediately.
4. Move the mouse off the header: popup hides and **audio stops instantly** (listen for any tail).
5. Quick pass over the header (< 300 ms, slide across multiple headers): run `pgrep -af streamlink` right after — no process.
6. Set "Show thumbnails of streams" = Large (360×203): preview is large; = Small: preview is small; = Off: stage 1 is text-only, video appears above the text at 160×90.
7. Check CPU while previewing vs idle (`top`): record both in Task 8's notes (numbers go into the upstream discussion draft).

- [ ] **Step 5: Commit**

```bash
git add src/widgets/StreamPreviewPopup.hpp src/widgets/StreamPreviewPopup.cpp CMakeLists.txt src/CMakeLists.txt
git commit -m "feat: live video stage with muted-by-default audio"
```

---

### Task 7: Lifecycle hardening and edges

**Files:**
- Modify: `src/widgets/StreamPreviewPopup.hpp`, `src/widgets/StreamPreviewPopup.cpp`
- Modify: `src/widgets/splits/SplitHeader.cpp`

**Interfaces:**
- Consumes: everything from Tasks 4–6 (`resetToStage1()`, members listed in Task 6).
- Produces: the spec's anti-goals hold on every stop path — no orphans, no audio without video, silent failure, Streamer Mode suppression, stable under thrash. No new public API.

- [ ] **Step 1: Streamer Mode live stop**

In `StreamPreviewPopup`'s pipeline setup, connect `getApp()->getStreamerMode()->changed(bool enabled)` → if `enabled`, call `endPreview()`.
Verify: start a preview, turn Streamer Mode on in settings → popup and sound stop instantly; `pgrep -af streamlink` empty.

- [ ] **Step 2: Feature toggle and channel change stop paths**

Connect `getSettings()->streamPreviewEnabled` (setting signal) → if it became false, `endPreview()`.
In `SplitHeader::handleChannelChanged()` (line ~863): call `previewPopup_->endPreview()` first.
In `SplitHeader::enterEvent`, guard re-entry: if `previewPopup_->isPreviewVisible()`, call `endPreview()` before `beginPreview()` (reset, no double pipeline).

- [ ] **Step 3: Failure and error reverts**

- Player errors: connect `player_->errorOccurred` (any) and `mediaStatusChanged` for `EndOfMedia`/`InvalidMedia` → `player_->stop(); resetToStage1();` (stage 1 stays visible; debug log only — `qCDebug(chatterinoStreamlink)` or similar).
- Stream ends mid-preview: connect `twitchChannel->streamStatusChanged` handling already exists in SplitHeader for text updates; add — when `!isLive_` and the preview is visible, call `previewPopup_->endPreview()` (no live stream to show; the tooltip/status text is updated by existing code).

- [ ] **Step 4: Manual verification checklist (all must pass)**

1. No streamlink on PATH (or set External Tools → custom path to `/tmp` and enable it): hover live → stage-1 popup only, **no dialogs**; `pgrep -af streamlink` empty.
2. Bogus streamlink (`/bin/false` as binary): hover → stage 1 only, silent.
3. Live channel goes offline mid-preview (find a stream ending, or disconnect network): video reverts to stage 1, no crash, no dialog.
4. Switch the split's channel (double-click header) mid-preview: preview ends instantly, `pgrep -af streamlink` empty.
5. Close the split mid-preview; close the window mid-play; quit the app mid-load: each time `pgrep -af streamlink` empty within ~2 s.
6. Rerun channel: preview plays (tooltip says "Vod-casting").
7. Set preferred quality = AudioOnly → preview still shows video (High mapping); set = Choose → same.
8. Streamer Mode toggle mid-preview (Step 1) and feature toggle mid-preview (Step 2).
9. Rapid mouse wiggling across a header edge for ~10 s: no crash, no stuck popup, `pgrep -af streamlink` shows at most one process while dwelling/playing.
10. Note in the scratch log: flatpak-spawn kill semantics not exercised here (native build); re-verify if a Flatpak build is made.

- [ ] **Step 5: Commit**

```bash
git add src/widgets/StreamPreviewPopup.hpp src/widgets/StreamPreviewPopup.cpp src/widgets/splits/SplitHeader.cpp
git commit -m "fix: harden stream preview lifecycle (streamer mode, errors, cleanup)"
```

---

### Task 8: Upstream preparation

**Files:**
- Modify: `CHANGELOG.md` (Unreleased section)
- Create: `docs/superpowers/notes/2026-10-19-upstream-discussion-draft.md`

**Interfaces:**
- Consumes: the working feature branch and the manual-test results (Task 7) + CPU numbers (Task 6).
- Produces: a fork branch whose diff is PR-ready (no internal docs), and a discussion draft for the user to post before any PR.

- [ ] **Step 1: Changelog entry**

Under `## Unreleased` add exactly one line (match on-disk formatting):
`- Minor: Added a live stream preview when hovering a split header, disabled by default. (#XXXX)`

- [ ] **Step 2: Write the upstream discussion draft**

`docs/superpowers/notes/2026-10-19-upstream-discussion-draft.md`, sections: problem/motivation; demo (describe the capture to make: hover → thumbnail swaps to muted video → leave stops it); decisions (off by default, audio setting in Settings because the tooltip is mouse-transparent, thumbnail-first, size follows the thumbnail setting, Streamer Mode suppression, reruns included); implementation summary (`StreamPreviewPopup`, `StreamPreviewResolver`, Qt6 Multimedia requirement + explicit question whether a CMake optional gate is wanted); measured perf numbers; references to prior art (#2805/#2854 closed pending discussion); explicit ask: blessing to open a PR.

- [ ] **Step 3: Branch hygiene**

```bash
git branch internal-docs
git rm -r docs/superpowers
git commit -m "chore: remove internal docs from feature branch"
```
(Result: the eventual PR diff against upstream master contains no `docs/superpowers` changes; the docs live on the `internal-docs` branch.)

- [ ] **Step 4: Rebase and full verification**

```bash
git fetch upstream && git rebase upstream/master
cmake --build build -j$(nproc)
cmake --build build-tests -j$(nproc) && ./build-tests/bin/chatterino-test
```
Then repeat the Task 4 "feature off" identity pass (tooltips identical for live/rerun/offline with the setting off).

- [ ] **Step 5: Stop — no PR yet**

Do **not** push a PR. Hand the discussion draft to the user; any optional-dependency gate is implemented only after maintainer feedback. Commit the remaining fork changes.
```bash
git add CHANGELOG.md
git commit -m "chore: add changelog entry for stream preview"
```

---

## Task ↔ design-milestone mapping

| Design milestone (§8) | Tasks |
|---|---|
| 0 — streamlink trial | 1 |
| 1 — static popup replacing tooltip | 2, 3, 4 |
| 2 — streamlink→player spike | 5 |
| 3 — wire-up + mute | 6 |
| 4 — lifecycle hardening | 7 |
| 5 — settings + polish | 2, 6 |
| 6 — upstream discussion + PR prep | 8 |
