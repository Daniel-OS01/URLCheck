# Device test of v4.0.2 (custom tabs service) and settings backup

Status: **interim**. Install, restore and link handling are tested. The custom tabs fix does not change what the Google app does, and the root cause is found. Open items are at the end.

Phones used. Every finding below says which one it comes from.

| Short name | Serial | Model | OS | Role in this test |
|---|---|---|---|---|
| poco-f5 (marble) | `d9b3bef4` | 23049PCD8G | Android 15 (SDK 35), HyperOS 3.0.4.0 | test phone: install, restore, tests 1-6, intent replay |
| poco-x8 (dash) | `HEW8EMAYWKIVRWPN` | 2602BPC18G | Android 16 (SDK 36), HyperOS 3; users 0, 11 (work profile), 999 (XSpace) | the user's daily phone: read-only checks only |

Tooling note: this PC has JDK 17 (`C:\Users\Daniel\tools\jdk17\jdk-17.0.21.1+2`) but no Android SDK, so APKs are built only by CI (PR builds publish an `evaluation` APK that installs side by side as `com.trianguloy.urlchecker.test`).

## Screenshots

All four screenshots in [device-test-v4.0.2/](device-test-v4.0.2/) were taken on **poco-f5** and show only the URLCheck dialog, the URLCheck backup screen, a Brave error page for `example.com` and the launcher wallpaper (no notifications, account names, feed content or messages). Everything that was taken on the Google app, Gmail or Settings screens, and every screenshot or UI dump of **poco-x8**, was kept out of git (poco-f5 ones in `C:\Users\Daniel\Backups\urlcheck-settings\device-test\`; the poco-x8 captures were deleted right after reading them).

## Changes made on poco-f5 (test phone) and how to revert

| # | Change | Revert |
|---|---|---|
| 1 | Play Store build 3.5 (versionCode 47, installer `com.android.vending`) uninstalled; fork build 4.0.2 (versionCode 50, sideloaded, APK SHA-256 `80BBE9E4...28FBA` verified) installed | the phone's own settings were exported first: `C:\Users\Daniel\Backups\urlcheck-settings\URLCheck_20261124232605_phone-play-3.5.ucbckp` (data + secrets, user-only ACL). To go back: uninstall the fork build, install from Play, restore that file |
| 2 | URLCheck settings restored from the **safe** backup `URLCheck_20261124230500_safe.ucbckp`, not from the phone's own export | restore the phone's own export above |
| 3 | Browser role: URLCheck (it fell back to Chrome when the Play build was uninstalled) | `adb -s d9b3bef4 shell cmd role add-role-holder android.app.role.BROWSER <package> 0` |
| 4 | Airplane mode turned off (it was on) so the Google app has network | `adb -s d9b3bef4 shell cmd connectivity airplane-mode enable` |
| 5 | URLCheck app-link selection enabled for `medium.com`, `*.wikipedia.org`, `www.reddit.com` (all declared hosts were disabled) | `adb -s d9b3bef4 shell pm set-app-links-user-selection --user 0 --package com.trianguloy.urlchecker false medium.com '*.wikipedia.org' www.reddit.com` |
| 6 | Clipboard replaced by the text `hello world` (to test "No links detected") | none needed |
| 7 | Files left in `/sdcard/Download`: `URLCheck_safe.ucbckp`, `URLCheck_20261124232605.ucbckp` | delete them |
| 8 | Google app force-stopped and opened; it was left on its settings page after a tap | none needed |

poco-f5 is reachable by USB or on the home LAN (`10.147.20.188:5555`, verify the serial before trusting the address).

## Changes made on poco-x8 (daily phone)

No setting, app, role or data was changed. Only read-only `dumpsys`, `pm get-app-links`, `cmd role get-role-holders`, `cmd package query-activities` / `resolve-activity`. Side effects to know about:

- one Back key press, which closed an "Open with" dialog the user had left open;
- the Google app was brought to the foreground twice and an attempt to open its (non-exported) settings activity was refused;
- the log buffer was cleared once (`logcat -c`);
- two UI-tree dumps and one header-only screenshot were pulled to the PC and deleted;
- the user was using the phone during the checks (the foreground app changed on its own, and URLCheck's "Verification link handling allowed" flipped from `false` to `true` between two reads), so no further input was sent.

## Tests on poco-f5

| # | Test | Expected | Actual | Result |
|---|---|---|---|---|
| 1 | Install v4.0.2 | versionName 4.0.2, fork signature | versionCode 50, versionName 4.0.2, signature differs from the Play build (so uninstall was required) | pass |
| 2 | Restore safe backup | no error, modules, patterns, automations present | toast "Backup restored" ([screenshot](device-test-v4.0.2/t2-3-after-restore-pick.png)); all modules listed; automations editor holds 7 rules (3 with `"enabled": false`); the backup holds 57 patterns including `Clean: Google AMP viewer to original` and `Warn: IP address host (IPv4 or IPv6)` (checked in the backup file, the pattern editor text is truncated in the UI tree) | pass |
| 3a | `VIEW` + `BROWSABLE` `https://example.com/?utm_source=x&fbclid=abc&id=5` | URLCheck dialog, tracking removed, hand-off to a real browser | dialog shows `https://example.com/?id=5`, hand-off target Brave, Brave opened ([dialog](device-test-v4.0.2/t3-1-view-browsable.png), [Brave](device-test-v4.0.2/t3-3-after-open-brave.png)); URLCheck did not reopen itself | pass |
| 3b | same without `BROWSABLE` | same | same dialog, cleaned URL. One earlier run right after Home showed no dialog and was not reproduced in three retries | pass (one unexplained miss) |
| 4a | `SEND` `text/plain` "see https://example.com/?utm_medium=y" | URLCheck offered, link extracted | URLCheck listed under "More" in the share sheet, dialog shows `https://example.com/` | pass |
| 4b | clipboard check | "No links detected" only when no link | link in clipboard: dialog opened; plain text: toast "No links detected" ([screenshot](device-test-v4.0.2/t4-6-clipboard-nolink.png)). The quick-settings tile was not tested separately (same code path as the toolbar button) | pass |
| 5 | URLCheck as default browser | role holder is URLCheck | `cmd role add-role-holder` worked; `resolve-activity` for `http` returns `MainDialog` | pass |
| 6 | Custom tabs from the Google app | `onBind` and `transaction N` in the `CUSTOMTABS` log, link opens in URLCheck | **no `CUSTOMTABS` line at all**, link opened in a Chrome custom tab | **fail (not caused by the service)** |
| 6b | another custom tabs app | opens via URLCheck | not testable: Telegram, Reddit and Infinity are disabled on the phone, Gmail would need mail content | not tested |
| 7 | links from other apps | lands in URLCheck | see the root cause below; a neutral source in another app is still open | partly |

## Root cause: why the Google app opens Chrome

### Captured on poco-f5 (system log, Google app search result tap)

1. `START u0 {act=VIEW cat=[BROWSABLE] dat=https://www.iana.org/... flg=0x10000400 (has extras)}` from the Google app, result code `-91`. Flag `0x400` is `FLAG_ACTIVITY_REQUIRE_NON_BROWSER`: the Google app first asks Android for an app that is **not a browser**; if none exists the start fails.
2. `START u0 {act=VIEW dat=https://www.iana.org/... pkg=com.android.chrome cmp=com.android.chrome/...IntentDispatcher (has extras)}` from the Google app, then Chrome logs `cr_ChromeConnection: New Custom Tab session created by package: com.google.android.googlequicksearchbox`. The Google app targets Chrome by package name and never bound URLCheck's service (`onBind` never ran).

### Captured on poco-x8 (the "Open with" dialog that was on screen, `dumpsys activity activities`)

`Intent { act=android.intent.action.VIEW dat=https://x.com/<account>/status/<id> flg=0x800000 xflg=0x4 pkg=com.android.chrome cmp=com.android.intentresolver/.ResolverActivity (has extras) }`, `launchedFromPackage=com.google.android.googlequicksearchbox`, `resultTo=...googlequicksearchbox/.SearchActivity`. The extras are not printed by `dumpsys`.

### Conclusion (both phones)

The Google app sends `VIEW` with an explicit `package=com.android.chrome`. A package-targeted intent is delivered only to that package, so URLCheck's intent filters are never consulted, whatever hosts they list. The "Open with: Personal/Work, Chrome only" dialog is the system resolver choosing between the personal and the work copy of Chrome (poco-x8 has a work profile); it is not a list of browsers.

Checks on poco-x8 (read-only): with `x.com` already approved for URLCheck, a plain `VIEW` + `BROWSABLE` `x.com` URL resolves to `com.trianguloy.urlchecker/.dialogs.MainDialog`; the same URL with `-p com.android.chrome` resolves only to Chrome's `IntentDispatcher`. The browser role in user 0 is URLCheck; in user 999 (XSpace) it is Chrome.

### Supported links (adding hosts), tested on poco-f5

With `medium.com`, `*.wikipedia.org` and `www.reddit.com` selected in "Open by default" (change 5 above):

| Intent | Result |
|---|---|
| `VIEW` `BROWSABLE` `https://medium.com/...` flags `NEW_TASK` | URLCheck dialog |
| same with `REQUIRE_NON_BROWSER` (what the Google app sends first) | not resolved |
| same for `en.wikipedia.org` and for a host URLCheck does not declare | not resolved |

Approving hosts changes nothing for the observed cases, and adding more hosts to the filter would not either: the Google app's decisive intent is package-targeted (Chrome), and its first attempt (`REQUIRE_NON_BROWSER`) is rejected for URLCheck even on approved hosts. Likely reason (not proven): Android keeps only the first matching filter per activity, and for `MainDialog` that is the generic http/https filter, which marks the activity as a browser. A host-only filter in a **separate** activity (alias) would be classified as a non-browser; whether that makes the Google app hand an approved host to URLCheck has not been tested.

## Open items

- Google app setting "Open web pages in the Google app" (Settings > General): not read yet on either phone. poco-x8 is in use by the user, so no input was sent to it.
- Evaluation of a host-only `activity-alias` (non-browser) so that `REQUIRE_NON_BROWSER` callers can reach URLCheck for approved hosts, using a CI evaluation APK.
- Test 7 with a neutral source (Settings legal link, a note created and deleted by the test).
- Revert the poco-f5 changes listed above when the phone is no longer needed.
