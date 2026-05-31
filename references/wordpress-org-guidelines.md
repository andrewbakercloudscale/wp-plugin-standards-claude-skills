# WordPress.org Detailed Plugin Guidelines — human-review compliance

The automated half of review (Plugin Check / PHPCS) is covered by `pcp-checklist.md`
and the security references. **This file covers the other half: the 18 Detailed Plugin
Guidelines a *human* reviewer enforces** — the policy/business rules that no linter
catches and that cause most repeat rejections of otherwise clean code.

Source: https://developer.wordpress.org/plugins/wordpress-org/detailed-plugin-guidelines/

Review every plugin against all 18 below. Guidelines 1, 4, 5, 6, 7, 8, 10, 11, 12, 13
are the ones that are **not** detectable by PCP — check them by reading the code,
the readme, and the admin UI behaviour, not by running a tool.

---

## 1 — GPL-compatible licensing (all code AND assets)

- Plugin `License:` must be `GPLv2 or later` or another GPL-compatible licence.
- **Everything bundled must be GPL-compatible — not just PHP libraries.** Images,
  icons, fonts, JS/CSS libraries, and data files all count.
- **Audit:** for every file under `assets/`, `images/`, `fonts/`, `lib/`, `vendor/`,
  confirm a GPL-compatible licence. List bundled third-party works and their licences
  in `readme.txt` under `== Credits ==` or a clearly-labelled section.
- Common rejection: a stock icon set, font, or JS widget under a non-GPL/CC-BY-ND/
  "free for personal use" licence. Replace with a GPL-compatible equivalent.

## 2 — Developer responsibility

- The author is responsible for **every** bundled file and for complying with the
  **terms of service of every external API** the plugin calls.
- **Check:** if the plugin integrates a third-party API (payment gateway, AI provider,
  cloud storage), confirm that automated/programmatic use is permitted by that service's
  ToS and that any required attribution is present.

## 3 — Stable version must be available on the directory

- The version on the WordPress.org SVN (`trunk` / tagged `Stable tag`) must be the
  functioning version users get. Don't ship a stub to the directory while distributing
  the "real" plugin elsewhere.
- **Check:** `Stable tag` in `readme.txt` points to a real, complete version; the
  directory copy is the full plugin, not a loader that fetches code elsewhere (see #8).

## 4 — Code must be human-readable (no obfuscation, no minified-only) 🔴 commonly missed

- **No obfuscated code.** No `eval()` of encoded strings, no `base64`/`gz`/`str_rot13`
  decode-then-execute, no packed/mangled source.
- **Minified JS/CSS is only allowed if the unminified source is also included** in the
  plugin (or its public source repo, linked in the readme). Shipping only `app.min.js`
  with no `app.js` is a rejection.
- **Audit:** `grep -rn "eval(\|base64_decode\|gzinflate\|gzuncompress\|str_rot13\|\\\\x[0-9a-f]" --include=*.php`.
  For every `*.min.js` / `*.min.css`, confirm a non-minified counterpart ships or the
  readme links to the buildable source.

## 5 — No trialware / locked or time-limited functionality 🔴 commonly missed

- The plugin's advertised functionality **must work fully** in the directory version.
  You may **not**: disable features until payment, expire features after a trial period,
  gate features behind a licence key, or ship a "sandbox/demo only until you upgrade".
- Upselling a *separate* pro add-on is fine **if** the free plugin is fully functional on
  its own and the upsell follows #10 and #11.
- **Audit:** search for licence/trial gating — `grep -rin "trial\|license_key\|licence_key\|is_pro\|->pro\|expire\|activation_key\|unlock" --include=*.php`.
  Read each hit: does it *disable real functionality* of the free plugin? If so, it is
  trialware → must be removed or the feature made fully functional.

## 6 — SaaS is allowed, but the plugin must do real local work 🔴 commonly missed

- A plugin may rely on an external service for substantial functionality (e.g. an AI
  API, a media CDN). It may **not** be merely: a licence-key validator, an account
  signup/storefront, or an empty shell whose only job is to call your server.
- **Check:** does the plugin provide meaningful functionality in WordPress itself, or is
  it just a thin wrapper / iframe around a hosted service? If the latter, it will be
  rejected. Document the service in `== External services ==` (see #7).

## 7 — No tracking or "phoning home" without explicit opt-in consent 🔴 commonly missed

- The plugin must **not** contact any external server with user/site data unless the
  user has **explicitly opted in** (an unchecked-by-default checkbox or a clear consent
  prompt). Silent analytics, telemetry, "anonymous usage stats on by default", checking
  in on activation, or registering the site with your server are all rejections.
- This applies to **any** outbound call carrying site/user data: analytics beacons,
  error reporting, newsletter signup, "report this install".
- **Required:** document every external service in `readme.txt` under `== External
  services ==` — what service, what data is sent, when, and links to its Terms and
  Privacy Policy. The readme's privacy statement must be **accurate** (no "makes no
  external requests" while a beacon fires).
- **Audit:** find every `wp_remote_*` / outbound call and confirm (a) it is gated behind
  explicit opt-in (default off), and (b) it is documented. Calls to your *own* analytics
  endpoint on activation/page-load with no opt-in are the classic rejection here.

## 8 — No executing/loading external code; restricted asset sources 🔴 partly missed

- **No installing or loading plugins/themes from anywhere other than WordPress.org.**
- **No serving plugin assets from a third-party CDN.** All JS/CSS/images ship in the
  package and load locally. The **only** widely-accepted remote exception is **web
  fonts** (e.g. Google Fonts) — and even those are increasingly discouraged for privacy.
- **No embedding your admin pages via an `<iframe>` to an external service.** Admin
  screens must be rendered locally.
- **No remote `.php`/code includes**, no loading executable code from your server at
  runtime (ties to the "deploy executable code" hard-reject in `SKILL.md`).
- **Audit:** `grep -rn "<iframe\|cdn\.\|jsdelivr\|unpkg\|cloudflare\|googleapis\|<script[^>]*src=.https" --include=*.php --include=*.html`.
  Anything loading code/markup/assets from a remote host (other than fonts/documented
  API calls) is a violation.

## 9 — Legal & ethical conduct

- No black-hat SEO, fake reviews, sockpuppet ratings, plagiarised code, or harassment.
- **Check:** code is original or properly licensed/attributed; the readme makes no
  manipulative claims.

## 10 — "Powered by" links and credits must be opt-in and default-hidden 🔴 commonly missed

- The plugin must **not** add front-end links, "Powered by", attributions, or credit
  footers **by default**. Any such output must be **opt-in** (default off) via a clear
  setting — never buried in terms, never required for the plugin to function.
- **Audit:** search rendered front-end output for outbound links to your own
  site/author — `grep -rn "Powered by\|<a href=.https" --include=*.php public/ templates/`.
  Confirm any credit/link is gated behind a setting that defaults to disabled.

## 11 — No admin-dashboard hijacking 🔴 commonly missed

- Admin notices must be **dismissible** and **contextual** (shown on the plugin's own
  screens, not site-wide on every admin page). No persistent, non-dismissible nags.
- Upgrade/upsell prompts must be limited and not disruptive. Dashboard widgets must be
  dismissible. **No advertising with referral/affiliate tracking in the dashboard.**
- **Audit:** `grep -rn "admin_notices\|add_meta_box\|admin_print_notices" --include=*.php`.
  For each notice: is it limited to the plugin's screens, is it dismissible (with the
  dismissal persisted), and is it free of upsell/affiliate tracking? A global
  `admin_notices` nag on every page is the classic rejection.

## 12 — No readme spam 🔴 partly missed

- `Tags:` — **5 maximum**, all genuinely relevant. No competitor names, no trademarked
  terms you don't own, no keyword stuffing.
- No affiliate links in the readme. No "blackhat" SEO, no keyword-stuffed descriptions.
  The readme is written for humans, not search bots.
- **Audit:** count `Tags:`; scan the readme for affiliate URLs (`?ref=`, `/aff/`,
  `affiliate`), competitor plugin names, and repeated keyword stuffing.

## 13 — Use the libraries bundled with WordPress 🔴 commonly missed

- Do **not** bundle your own copy of a library WordPress already ships: **jQuery**,
  jQuery UI, **Backbone**, Underscore, **SimplePie**, **PHPMailer**, Requests, Masonry,
  MediaElement, etc. Use the core-registered handle instead (`wp_enqueue_script('jquery')`).
- Bundling your own creates security/compatibility risk and is rejected.
- **Audit:** look under `assets/`, `js/`, `lib/`, `vendor/` for `jquery*.js`,
  `phpmailer`, `class-phpmailer`, `SimplePie`, `underscore`, `backbone`. If found,
  remove and depend on the WordPress-registered handle.

## 14 — Don't commit excessively to SVN

- SVN is a release repo: commit deployment-ready versions, not every WIP change. Operational
  — not detectable in a single-version review, but avoid version churn.

## 15 — Increment the version every release

- Covered by the version-sync checks in `SKILL.md` / `reuse.md`: header `Version`,
  `VERSION` constant, and `readme.txt` `Stable tag` all match and increase each release.

## 16 — Plugin must be complete and functional at submission

- No name-squatting / reserving a slug for the future. The submitted plugin must be
  fully working (ties to #5 — no "coming soon" gated features).

## 17 — Respect trademarks and project names

- Covered in `SKILL.md`: a slug/name may not *begin* with a trademarked term you don't
  own. Use "Feature for Brand" form, or assert ownership in the review reply.

## 18 — Directory maintenance rights

- Informational: WordPress.org may update guidelines, disable plugins, or reassign
  inactive ones. No code action.

---

## Reviewer-reply checklist (when responding to a rejection)

- Address **every** cited item, and proactively fix the **same class** elsewhere in the
  code (reviewers explicitly note they may not list every instance).
- Be clear, concise, direct; include a concrete example of the fix.
- Never claim "fixed" or "tested" unless it is true and you tested on a clean install
  with `WP_DEBUG = true`.
- If you believe something is a false positive, say so politely with an example — do not
  silently leave it.
