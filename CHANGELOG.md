# Changelog

All notable changes to `ozz1ee.keyboard-cleaner` are documented here. The format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-09-09

### Fixed

- **Power button misclassified as keyboard.** On the Apple SMC power/lid
  device the v0.2.0 parser read the `B: KEY=` line as 32-bit words in
  LSB order, but the kernel actually emits `unsigned long` chunks
  MSB-first (64 bits on aarch64/x86_64). The bug set bit 52 instead of
  bit 116 (KEY_POWER), and the classifier's `key >> KEY_KPENTER`
  guard then labelled the power button as a keyboard, which would
  have silenced the power button for the whole cleaning window. The
  parser now decodes tokens MSB-first at 64 bits per token, and the
  classifier counts set bits instead of testing individual keycode
  positions, so power buttons, lid switches, and headset volume
  rockers are correctly skipped. Reported by the maintainer of the
  Omalaunch Extension Directory on 2026-09-08.

- **Single-handler device parser gap.** The `H: Handlers=event3` form
  (one token, no `kbd`/`leds` companions) was silently dropped because
  the parser only matched `event` as a leading prefix on each
  whitespace-separated token. The line was scanned for `event\d+$` on
  each token *and* on each `=`-split sub-token so both `Handlers=kbd
  event0 leds` and `Handlers=event3` resolve to `event3`. The
  headphone jack and lid switch entries now appear in the device
  list as a result.

- **Quote-stripping on device names.** The `N: Name="..."` parser used
  `.strip('"')` after splitting on `:`, but the remainder starts with
  ` Name=` so the leading character is a space, not a quote — the
  outer quotes survived. Replaced with a regex that pulls the first
  quoted run on the line. The notification copy and the per-second
  refresh now show the clean device name.

### Changed

- **Partial-failure safety.** `grab_devices()` previously returned
  `(grabbed, skipped)`; if any classified device ended up in
  `skipped` the helper still sent the "Blocking keyboard and
  pointer. Releasing in N s — wipe safely." notification. The new
  contract adds `classified_keyboards` and `classified_pointers` to
  the return value, and `main()` releases every grabbed fd before
  surfacing an `urgency=critical` "Refusing to start — do NOT wipe"
  notification whenever a required device could not be grabbed. A
  half-blocked input device is no longer treated as a soft success.

- **README discloses `input` group persistence.** A new subsection in
  *Privilege boundaries* documents that `omarchy plugin remove` does
  not drop the user from the `input` group, explains why group
  membership is broader than this plugin, and points at the manual
  `gpasswd -d "$USER" input` command for users who want to revoke
  the access.

### Added

- **`tests/test_devices.py`** with 16 focused tests covering the
  parser (bit-116 lock-in for KEY_POWER, BTN_LEFT at bit 272 for
  trackpads and mice, popcount window for real keyboards), the
  classifier (power button, lid switch, headset volume, headphone
  jack, real keyboard, trackpad, USB mouse), and the partial-failure
  contract (4-tuple return, `classified_keyboards` exposed for
  `main()` to refuse unsafe half-grabs). Runs on stdlib only with
  `python3 tests/test_devices.py` from the plugin directory.

## [0.2.0] - 2026-09-04

### Changed

- **Live desktop notification replaces the terminal countdown.** The helper
  no longer assumes it is running inside an interactive terminal. As soon
  as the input block starts it sends an Omarchy `Notify` message via
  `/usr/share/omarchy/bin/omarchy-notification-send`; the wrapper prints
  the assigned D-Bus id, and every subsequent tick passes that id back
  with `-r` so Quickshell refreshes the same pop-up instead of stacking
  fresh toasts. The pop-up carries the `nf-md-keyboard` glyph (U+F0313),
  matching the launcher icon, and the body updates in place: "Blocking
  keyboard and pointer. Releasing in {N}s — wipe safely." → "Cleaning
  keyboard — releasing in {N}s." → "Cleaning finished. Keyboard and
  pointer restored." on completion. Notifications are best-effort: if
  the wrapper is missing, the bus is down, or the daemon hangs, the
  grab and release path is unchanged.

- **Menu closes immediately on action pick.** Each workflow command now
  runs through `setsid --fork`, which forks into a new session and exits
  in 0 ms. Omalaunch sees the immediate successful exit, closes the
  plugin window, and returns to the launcher; the helper continues
  running in its own session for the full duration. Before this change
  the menu stayed open until the helper finished because Omalaunch
  dispatches non-terminal commands synchronously.

- **README lists `setsid` instead of `xdg-terminal-exec` as a runtime
  dependency.** The previous text was a leftover from the pre-popup
  flow that spawned a terminal window; the extension no longer uses
  `xdg-terminal-exec` at all.

- **`stdout` is silenced when no TTY is attached.** The helper now
  redirects `sys.stdout` to `/dev/null` unless `isatty()` is true, so
  prints inside the running block never write to a broken pipe. The
  user-visible countdown is delivered entirely through the desktop
  notification; terminal output is debug-only.

## [0.1.0] - 2026-09-04

### Added

- Initial release. Omalaunch workflow extension that grabs every
  keyboard and pointing device via `EVIOCGRAB` (`_IOW('E', 0x10, 4)`,
  ioctl 0x40044590) for a chosen duration — 15 s, 30 s, or 1 min —
  so the keyboard can be wiped without triggering keys. The helper
  walks `/proc/bus/input/devices`, classifies each entrypoint as a
  keyboard, a pointer, or a single-button device to skip, opens the
  matching `/dev/input/event*` node, and issues the ioctl on the file
  descriptor. Grabs live on the descriptors; a SIGTERM, SIGHUP,
  KeyboardInterrupt, or process exit triggers a best-effort release
  (`EVIOCGRAB` with 0) so the keyboard always comes back.
