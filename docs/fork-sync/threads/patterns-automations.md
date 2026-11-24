# Custom Patterns & Automations Update (v4.0.2)

## Purpose and Scope

Audit, fix, and extend the user's URLCheck v4.0.2 **custom patterns** (`files/patterns`) and **automations** (`files/automations`), guided by the official wiki and verified against the `Daniel-OS01/URLCheck` v4.0.2 source code and JDK 17 `java.util.regex`.

- **Baseline backup:** `URLCheck_20261124233625_phone.ucbckp` (v4.0.2, SHA-256 `2BFE58CC…B2BA`, 57 patterns, 7 automations).
- **Deliverable backup (local only, never committed):** `URLCheck_20261124234418_patterns-automations.ucbckp` (v4.0.2, 30,716 bytes, SHA-256 `8354C4F76A3A8D28A730006A6A558FA86DB0C471A072D644180F7381FECAD402`, 77 patterns, 13 automations).
- **Byte-identical preservation:** Only `files/patterns` (plain compact JSON) and `files/automations` (Android `org.json` compact JSON with `\/` escaping) were modified. `version` (`4.0.2`), `readme.txt`, `preferences`, `secrets`, `files/hosts`, and `files/clearUrlCatalog` are byte-identical to the baseline, with identical ZIP entry order, DEFLATE compression, comment, and UTF-8 flag bits (`0x808`).
- **Official wiki references:**
  - [Custom patterns wiki](https://github.com/TrianguloY/URLCheck/wiki/Custom-patterns)
  - [Automations wiki](https://github.com/TrianguloY/URLCheck/wiki/Automations)

---

## v4.0.2 Code Reference & Wiki Gaps

Verified directly against `PatternModule.java`, `PatternCatalog.java`, `RegexFix.java`, `AutomationRules.java`, `OpenModule.java`, `ShareUtility.java`, `StatusModule.java`, `UnshortenModule.java`, `VirusTotalModule.java`, `DrawerModule.java`, `WebhookModule.java`, `DebugModule.java`, and `ClearUrlModule.java`.

### Supported Pattern Fields (`PatternModule.java`)

| Field | Type | Default | Semantics |
|---|---|---|---|
| `enabled` | boolean | `true` | Skips pattern if `false` |
| `encode` | boolean | `false` | URL-encodes the input URL (UTF-8) before matching `regex` |
| `regex` | string \| list | `^.*$` | Java regex tested with `Matcher.find()` (partial match). If list, first matching entry is used |
| `excludeRegex` | string \| list | none | Java regex tested with `Matcher.find()`. If any matches, pattern is skipped |
| `replacement` | string \| list | none | Passed to `RegexFix.replaceAll(...)`. If list, a random element is chosen |
| `decode` | boolean | `false` | Runs `URLDecoder.decode(..., UTF-8)` on the replaced URL |
| `automatic` | boolean | `false` | Applies replacement automatically during `onModifyUrl` |
| `button` | string | `"Replace"` | Custom action button label (v4.0.x) |
| `applied` | string | `"Applied - <name>"` | Custom message shown once applied (v4.0.x) |
| `description` | string | ignored | Pass-through user note (ignored by app, preserved in JSON) |

### Supported Automation Fields & Actions (`AutomationRules.java`)

| Field | Type | Default | Semantics |
|---|---|---|---|
| `enabled` | boolean | `true` | Skips rule if `false` |
| `referrer` | string \| list | any | Exact package name match against `AndroidUtils.getReferrer(cntx)`. **Gotcha:** when `referrer == null` (clipboard, direct intent), the filter is skipped and the rule matches |
| `regex` | string \| list | any | Java regex tested with **`String.matches()` (full-string match)** |
| `excludeRegex` | string \| list | none | Java regex tested with **`String.matches()` (full-string match)** |
| `action` | string \| list | required | One or more action keys executed in order |
| `args` | object | `{}` | Action-specific parameters |
| `stop` | boolean | `false` | Stops evaluating further automation rules for the current URL |

Registered automation actions in v4.0.2 (only active when the owning module is enabled):

| Action | Module | Args | Notes |
|---|---|---|---|
| `open` | `OpenModule` | `package`, `component` (optional) | Without args opens first app in learned order (respects `open_rejected`). With args opens target directly |
| `share` | `OpenModule` | `package`, `component` (optional) | Refuses to share to URLCheck itself to prevent loops |
| `copy` | `OpenModule` | — | Copies current URL to clipboard |
| `ctabs` | `OpenModule` | — | Enables Custom Tabs toggle |
| `incognito` | `OpenModule` | — | Enables Incognito toggle (Firefox) |
| `close` | `OpenModule` | — | Closes URLCheck dialog |
| `checkStatus` | `StatusModule` | — | Triggers HTTP status / redirect check |
| `unshort` | `UnshortenModule` | — | Triggers unshorten check |
| `scan` | `VirusTotalModule` | — | Triggers VirusTotal scan (module is OFF by default; fails with error toast if module is disabled) |
| `drawer` | `DrawerModule` | — | Expands the drawer |
| `webhook` | `WebhookModule` | — | Sends URL to configured webhook (module is OFF by default) |
| `toast` | `DebugModule` | `text` (string) | Shows short toast with `args.text` |

Note: `ClearUrlModule` has `clear` commented out in v4.0.2 (ClearURLs auto-clean is controlled via `clearurl_auto` preference instead).

### Upstream Wiki Gaps Found
1. `Custom-patterns.md` uses `"enabled": "false"` (string instead of boolean) in the Fandom → BreezeWiki example and leaves dots unescaped in several examples (`tiktok.com`, `instagram.com`, `spotify.com`).
2. `Automations.md` does not list the supported fields, the 12 action keys, the `args` schema, full-match (`String.matches`) vs partial-match (`Matcher.find`) distinction, or the `referrer == null` quirk.
3. In `app/src/main/res/values/strings.xml`, `auto_share` does not yet mention the `package` and `component` args supported by `OpenModule.java` / `ShareUtility.java`.

---

## Patterns Inventory (57 → 77)

All 57 baseline patterns are preserved (0 removed, 20 added, 19 fixed/improved, 38 unchanged).

| # | Pattern Name | Status | Enabled | Mode | Reason / Notes |
|---|---|---|---|---|---|
| 1 | `Clean: upgrade http to https` | **Changed** | Yes | Auto | Added `(?i)` (matches `HTTP://` / `LOCALHOST`) and excluded link-local `169.254.x.x` & `0.0.0.0` |
| 2 | `Clean: lowercase HTTPS scheme` | **Added** | Yes | Auto | Normalizes `HTTPS://` / `Https://` to `https://` |
| 3 | `Clean: Google AMP viewer to original` | Unchanged | Yes | Auto | Verified in Java 17 |
| 4 | `Manual: AMP cache link to original` | Unchanged | Yes | Manual | Verified in Java 17 |
| 5 | `Clean: unwrap Google redirect (/url?q=)` | **Added** | Yes | Auto | Unwraps `google.*/url?q=` and `?url=` outbound redirects (`decode: true`) |
| 6 | `Clean: unwrap Facebook link shim (l.php)` | **Added** | Yes | Auto | Unwraps `l.facebook.com/l.php?u=` and `lm.facebook.com` (`decode: true`) |
| 7 | `Clean: unwrap Instagram link shim (l.instagram.com)` | **Added** | Yes | Auto | Unwraps `l.instagram.com/?u=` (`decode: true`) |
| 8 | `Clean: unwrap Reddit outbound link (out.reddit.com)` | **Added** | Yes | Auto | Unwraps `out.reddit.com/...?url=` (`decode: true`) |
| 9 | `Clean: unwrap YouTube outbound redirect` | **Added** | Yes | Auto | Unwraps `youtube.com/redirect?...&q=` (`decode: true`) |
| 10 | `Clean: unwrap Outlook/Teams Safe Links` | **Added** | Yes | Auto | Unwraps `*.safelinks.protection.outlook.com/?url=` (`decode: true`) |
| 11 | `Clean: unwrap Slack redirect` | **Added** | Yes | Auto | Unwraps `slack-redir.net/link?url=` (`decode: true`) |
| 12 | `Clean: fix ?& separator` | Unchanged | Yes | Auto | Keeps separator tidy after tracking removal |
| 13 | `Clean: drop trailing ? or &` | Unchanged | Yes | Auto | Removes dangling `?` / `&` |
| 14 | `Warn: suspicious TLD` | Unchanged | Yes | Warn | Verified host-only matching |
| 15 | `Warn: punycode host` | Unchanged | Yes | Warn | Verified `xn--` host-only matching |
| 16 | `Warn: user@host trick` | Unchanged | Yes | Warn | Verified credentials/userinfo warning |
| 17 | `Warn: IP address host (IPv4 or IPv6)` | Unchanged | Yes | Warn | Verified IPv4 and `[IPv6]` |
| 18 | `Warn: non-ASCII host` | Unchanged | Yes | Warn | Verified host-only (allows Hebrew paths) |
| 19 | `Warn: non-ASCII anywhere (old rule)` | Unchanged | No | Warn | Kept disabled |
| 20 | `Warn: uppercase letter in domain` | **Added** | Yes | Warn | Catches `googIe.com`-style uppercase homographs in host |
| 21 | `Warn: dangerous scheme (data/javascript/blob/file)` | **Added** | Yes | Warn | Warns on `javascript:`, `data:`, `vbscript:`, `blob:`, `file:` |
| 22 | `Warn: unusual port` | **Added** | Yes | Warn | Warns on non-80/443 ports; excludes localhost & LAN/private IPs via `excludeRegex` |
| 23 | `Warn: URL shortener` | **Added** | Yes | Warn | Visual badge for 26 known shorteners while Unshorten runs |
| 24 | `Warn: lookalike brand domain` | **Added** | Yes | Warn | High-confidence phishing lookalikes (`brand.com.evil`, `brand-verify.*`), zero false positives on official domains |
| 25 | `Manual: strip tracking (one tap)` | **Changed** | Yes | Manual | Added `(?i)` so uppercase params (`UTM_SOURCE`, `FBCLID`) are also stripped |
| 26 | `Manual: mobile to desktop (m. / mobile.)` | **Added** | Yes | Manual | One-tap button (`Desktop site`) stripping `m.` / `mobile.` including `en.m.wikipedia.org` |
| 27 | `Redirect: Reddit to Redlib` | **Changed** | Yes | Manual | Added `excludeRegex` for `out.reddit.com` so outbound redirects aren't misrouted to Redlib |
| 28 | `Redirect: redd.it short link to Redlib` | Unchanged | Yes | Manual | Preserved |
| 29 | `Redirect: i.redd.it image to Redlib` | Unchanged | Yes | Manual | Preserved |
| 30 | `Redirect: preview.redd.it image to Redlib` | Unchanged | Yes | Manual | Preserved |
| 31 | `Redirect: Reddit to Reveddit` | Unchanged | No | Manual | Kept disabled |
| 32 | `Redirect: X/Twitter to Nitter` | Unchanged | Yes | Manual | Preserved |
| 33 | `Redirect: X/Twitter to FxTwitter` | **Added** | Yes | Manual | Wiki example; one-tap `FxTwitter` button for `/status/` links |
| 34 | `Redirect: YouTube to Invidious` | Unchanged | Yes | Manual | Preserved |
| 35 | `Redirect: youtu.be to Invidious` | Unchanged | Yes | Manual | Preserved |
| 36 | `Redirect: Imgur to rimgo` | Unchanged | Yes | Manual | Preserved |
| 37 | `Redirect: Medium to Scribe` | Unchanged | Yes | Manual | Preserved |
| 38 | `Redirect: Medium subdomain to Scribe` | Unchanged | Yes | Manual | Preserved |
| 39 | `Redirect: Wikipedia to Wikimore` | Unchanged | Yes | Manual | Preserved |
| 40 | `Redirect: Google Translate to SimplyTranslate or Lingva` | Unchanged | Yes | Manual | Preserved |
| 41 | `Redirect: Google Maps place to OpenStreetMap` | **Changed** | Yes | Manual | Added `(?i)` for case-insensitive scheme/host |
| 42 | `Redirect: Fandom to BreezeWiki` | **Added** | Yes | Manual | Wiki example with `excludeRegex` for `www.fandom.com` and `BreezeWiki` button |
| 43 | `Redirect: Instagram to Kittygram` | Unchanged | No | Manual | Kept disabled |
| 44 | `Redirect: StackOverflow to AnonymousOverflow` | Unchanged | No | Manual | Kept disabled |
| 45 | `Redirect: Quora to Quetre` | Unchanged | No | Manual | Kept disabled |
| 46 | `Redirect: TikTok to VxTikTok` | **Added** | No | Manual | Wiki example (fixed escaped dot), disabled by default |
| 47 | `Redirect: Instagram to InstaFix` | **Added** | No | Manual | Wiki example (fixed escaped dot, excludes `/accounts/`), disabled by default |
| 48 | `Tool: add https://` | Unchanged | Yes | Manual | Preserved |
| 49 | `Tool: add http://` | Unchanged | No | Manual | Kept disabled |
| 50 | `Tool: Instagram login wall to post` | Unchanged | Yes | Manual | Preserved |
| 51 | `Tool: Google Maps place to geo: URI` | **Changed** | No | Manual | Added `(?i)` |
| 52 | `Tool: TinyURL preview` | Unchanged | Yes | Manual | Preserved |
| 53 | `Tool: bit.ly preview` | **Changed** | Yes | Manual | Added `(?i)` and optional trailing slash `\/?$` |
| 54 | `Tool: is.gd preview` | **Changed** | No | Manual | Added `(?i)` |
| 55 | `Tool: tiny.cc preview` | **Changed** | No | Manual | Added `(?i)` |
| 56 | `Tool: archive.ph snapshots` | **Changed** | No | Manual | Added `(?i)` |
| 57 | `Tool: Wayback save` | **Changed** | No | Manual | Added `(?i)` |
| 58 | `Tool: Wayback view` | **Changed** | No | Manual | Added `(?i)` |
| 59 | `Tool: Play Store to APKMirror` | **Changed** | Yes | Manual | Added `(?i)` |
| 60 | `Tool: Play Store to APKPure` | **Changed** | Yes | Manual | Added `(?i)` |
| 61 | `Tool: Play Store to Aurora Store` | **Changed** | No | Manual | Added `(?i)` |
| 62 | `Tool: Amazon short product link` | **Changed** | Yes | Manual | Added `(?i)`, `excludeRegex` (no button if already short `/dp/<ASIN>`), `button`/`applied` labels |
| 63 | `Tool: eBay short item link` | **Changed** | Yes | Manual | Added `(?i)`, stripped item slug, added `excludeRegex` (no button if already short `/itm/<id>`), `button`/`applied` labels |
| 64 | `Tool: AliExpress short item link` | **Changed** | Yes | Manual | Added `excludeRegex` (no button if already clean), `button`/`applied` labels |
| 65 | `Tool: Song.link (music streaming)` | **Added** | Yes | Manual | Wiki & built-in example for Spotify/Apple/YouTube Music/Bandcamp/etc., with `excludeRegex` |
| 66 | `Tool: check with urlscan.io` | **Added** | No | Manual | Wiki example, disabled by default to avoid appearing on every URL |
| 67 | `Tool: GitHub repo in GitHub1s` | **Changed** | No | Manual | Added `(?i)` |
| 68 | `Tool: X to twitter.com` | Unchanged | No | Manual | Kept disabled |
| 69 | `Tool: try Cobalt` | **Changed** | No | Manual | Added `(?i)` |
| 70 | `Tool: try loader.to` | **Changed** | No | Manual | Added `(?i)` |
| 71 | `Tool: decode %2F` | **Changed** | No | Manual | Added `(?i)` (also matches `%2f`) |
| 72 | `Tool: decode %3A` | **Changed** | No | Manual | Added `(?i)` (also matches `%3a`) |
| 73 | `Tool: decode %40` | **Changed** | No | Manual | Added `(?i)` |
| 74 | `Tool: search with Brave` | Unchanged | No | Manual | Kept disabled |
| 75 | `Tool: search with DuckDuckGo` | Unchanged | No | Manual | Kept disabled |
| 76 | `Tool: search with Startpage` | Unchanged | No | Manual | Kept disabled |
| 77 | `Tool: search with searx.be` | Unchanged | No | Manual | Kept disabled |

---

## Automations Inventory (7 → 13)

All rules that change where links open (`open`, `ctabs`, `share`, `copy`) are **disabled by default**.

| # | Automation Name | Status | Enabled | Actions | Reason / Notes |
|---|---|---|---|---|---|
| 1 | `Security: suspicious TLD, check and drawer` | **Changed** (renamed from `…and VirusTotal`) | Yes | `checkStatus`, `drawer` | Removed `scan` because `VirusTotalModule` is disabled and triggered an error toast (`auto_error_toast: true`) on every match |
| 2 | `Security: punycode, non-ASCII host or user@host, check and drawer` | Unchanged | Yes | `checkStatus`, `drawer` | Preserved verbatim |
| 3 | `Security: IP address host or unusual port, check and drawer` | **Added** | Yes | `checkStatus`, `drawer` | Runs status check & opens drawer on public IP hosts or non-80/443 ports (excludes localhost & private LAN hosts) |
| 4 | `Security: suspicious TLD, VirusTotal scan (enable VirusTotal first)` | **Added** | No | `scan` | Preserves VirusTotal scanning cleanly as a separate opt-in rule |
| 5 | `Unshorten: known shorteners` | **Changed** | Yes | `unshort` | Expanded from 10 to 26 shorteners; added `excludeRegex` so `bit.ly/...+`, `is.gd/...-`, `tiny.cc/...~` preview links aren't auto-unshortened |
| 6 | `Info: link from an AI app was checked` | Unchanged | Yes | `toast` | Preserved verbatim |
| 7 | `Open: YouTube in ReVanced` | Unchanged | No | `open` | Kept disabled |
| 8 | `Open: Reddit in the Reddit app` | Unchanged | No | `open` | Kept disabled |
| 9 | `Open: auto-continue clean web links in Brave (disabled by default)` | **Added** | No | `open` (`package: com.brave.browser`, `stop: true`) | Opt-in only; excludes suspicious TLDs, punycode, non-ASCII, `user@host`, IP hosts, shorteners, native-app domains, and embedded redirects |
| 10 | `Open: enable Custom Tabs for Google apps (disabled by default)` | **Added** | No | `ctabs` | Opt-in Custom Tabs toggle for Google Search & Google News referrers |
| 11 | `Share: send to Telegram (disabled example)` | **Added** | No | `share` (`package: org.telegram.messenger`) | Demonstrates v4.0.2 `share` `package` arg; targets `example.invalid` until edited |
| 12 | `Copy: auto-copy clean link (disabled example)` | **Added** | No | `copy` | Demonstrates `copy` action; targets `example.invalid` until edited |
| 13 | `Send everything to webhook` | Unchanged | No | `webhook` | Kept disabled |

---

## Validation & Test Results

- **Java 17 Regex Compilation (`java.util.regex.Pattern.compile`):** **112 / 112 passed** (0 failures across all `regex` and `excludeRegex` entries in both `files/patterns` and `files/automations`).
- **Java 17 Pattern Behavioral Suite (`Matcher.find()` + `replaceAll()` + `URLDecoder.decode`):** **71 / 71 passed**, covering positive & negative cases for HTTP upgrade, uppercase HTTPS, Google/Facebook/Instagram/Reddit/YouTube/Outlook/Slack redirect unwrapping, AMP viewer/cache, all 11 `Warn:` rules, mobile→desktop (including `en.m.wikipedia.org`), Fandom→BreezeWiki, FxTwitter, Amazon/eBay/AliExpress shorteners (including already-short negative tests), `bit.ly` preview, and Song.link.
- **Java 17 Automation Behavioral Suite (`String.matches()`):** **24 / 24 passed**, covering suspicious TLDs, punycode/user@host, public IP/unusual port vs localhost/LAN exclusions, expanded shorteners vs `+`/`-`/`~` preview exclusions, and Brave auto-continue exclusions.
- **Backup Validator (`docs/fork-sync/tools/validate_backup.py`):** **ALL CHECKS PASSED** (CRC, entry order, `version = '4.0.2'`, DEFLATE, zip comment, UTF-8 `0x808` flags via Java 17 `ZipOutputStream`, preferences/secrets strict JSON & key check, all catalog schemas).
- **On-device smoke test:** Skipped. `d9b3bef4` (marble) was disconnected and `HEW8EMAYWKIVRWPN` was actively used by the device-test thread and was not touched.

---

## How to Enable Disabled Rules

In URLCheck on the phone:
1. Open **URLCheck → Settings → Automations** (or **Modules → Pattern checker → Edit**).
2. Find the rule you want and change `"enabled": false` to `"enabled": true` (or remove the `"enabled"` line).
3. For `Share: send to Telegram` or `Copy: auto-copy clean link`, also change `example\.invalid` in `"regex"` to the domain(s) you want.
4. For `Security: suspicious TLD, VirusTotal scan`, first enable the **VirusTotal** module and configure an API key under **Modules → VirusTotal**.

---

## Open Questions for the Owner

1. **Auto-continue in Brave:** Kept disabled by default so links always show the URLCheck dialog. Enable `Open: auto-continue clean web links in Brave` only if you want clean non-app web links to skip the dialog.
2. **Custom Tabs from Google apps:** Decide whether to enable `Open: enable Custom Tabs for Google apps` (`ctabs` action).
3. **Optional patterns currently disabled:** `Redirect: TikTok to VxTikTok`, `Redirect: Instagram to InstaFix`, and `Tool: check with urlscan.io` are included disabled — enable any you use.
4. **VirusTotal scanning:** If you add a VirusTotal API key and enable the module, enable `Security: suspicious TLD, VirusTotal scan`.
