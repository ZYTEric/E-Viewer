# Log-on fix

This change fixes log-on for both e-hentai.org and ExHentai.

## Problem

- Cookie log-on reported a generic connection error.
- The web log-on page did not load, or the challenge could not be completed.
- After web log-on, the app asked for an `igneous` cookie even when the account could use the surface site without one.

## Causes

1. `igneous` was accepted only as 16 hexadecimal characters and was lowercased. Current values are not hexadecimal, for example `vfjpw8zgrqtexu1w9`.
2. `LogOnAsync` called `ResetExCookieAsync`, which assumed that visiting ExHentai would issue a valid `igneous`. ExHentai now answers a request without `igneous` with `igneous=mystery`, and the empty response was treated as success.
3. The log-on page is on `forums.e-hentai.org`, which is protected by Cloudflare. The EdgeHTML `WebView` cannot complete the challenge.
4. `igneous` was treated as mandatory. It is optional: without it the account can still use e-hentai.org.
5. Startup evaluated `NeedLogOn` before applying the site setting, so `Host` could be wrong.

## Changes

- `ExClient/Client-User.cs`
  - Accept `[0-9A-Za-z_-]` for `igneous`, keep the original case, and reject `mystery`, `deleted` and `null`.
  - Write `ipb_member_id` and `ipb_pass_hash` to both domains.
  - Verify the credentials against `home.php` after log-on.
  - If `igneous` is present but rejected by ExHentai, remove it and keep the surface log-on.
  - `NeedLogOn` only requires the member ID and pass hash.
- `ExViewer/Views/LogOnDialog.xaml(.cs)`
  - Use the WinUI 2 `WebView2` control for the web log-on page.
  - Read cookies after log-on and reuse an existing `igneous` for the same account when the current network cannot issue one.
  - If no valid `igneous` is available, switch to the cookie form with the member ID and pass hash filled, so the user can log on to the surface site or add a valid `igneous`.
  - Keep the forum connection alive while the form is open.
  - Validate the source of WebView2 messages and handle navigation and initialization failures.
- `ExViewer/Views/RootController.cs`, `SplashControl.xaml.cs`
  - Apply the site setting before checking `NeedLogOn`.
  - If ExHentai is enabled but no valid `igneous` is available, switch to e-hentai.org and show a notice.
- `ExClient/Internal/MyHttpClient.cs`
  - Remove the automatic ExHentai cookie reset, which hid the real error.
- CI
  - Pin the runner to `windows-2022`; `windows-latest` no longer provides the UWP toolchain.

## Behavior

- With a valid `igneous`, the account uses ExHentai as before.
- Without a valid `igneous`, log-on succeeds on e-hentai.org and the app disables ExHentai with a notice. A valid `igneous` can be added later in settings.
- Web log-on is optional. Cookie log-on remains the default and does not depend on Cloudflare or WebView2.
- If the WebView2 Runtime is missing, web log-on shows a message and returns to the cookie form. The Evergreen WebView2 Runtime is required only for web log-on.

## Compatibility

- Windows 10 build 17763 or later, Windows 11.
- `igneous` values in both the legacy hexadecimal format and the current non-hexadecimal format are accepted.
- Existing cookies remain valid; no data migration is required.

## Testing

- CI builds x86, x64, arm and arm64.
- Cookie log-on with only `ipb_member_id` and `ipb_pass_hash` succeeds on the surface site.
- Cookie log-on with a valid `igneous` loads ExHentai.
- Web log-on completes through Cloudflare and either obtains a valid `igneous` or switches to the pre-filled cookie form for the surface site.
- The fallback behavior was verified on networks that can and cannot access ExHentai.

## Known limitation

If `igneous` expires while the app is running, ExHentai requests may still fail until the app is restarted or the account is logged on again. The next start detects the invalid value and switches to the surface site.
