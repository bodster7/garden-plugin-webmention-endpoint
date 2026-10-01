# SPEC: garden-plugin-webmention-endpoint

- Plugin id: `webmention-endpoint` (manifest `id`, folder name and gallery `id` all identical).
  Not `dg-`: that prefix is reserved for first-party plugins (garden-plugin-author skill).
- Dev location: `bodster_garden/src/plugins/webmention-endpoint/` (own git repo, ignored by the garden)
- Garden branch: `dg-webmention-endpoint-dev` (branch name only; kept as is)

## Purpose
Get a Digital Garden receiving webmentions without touching the site template:
1. Advertise a webmention.io receiving endpoint on every page (v1.0).
2. **v1.1:** optionally publish a `rel="me"` link so a garden with no other site can
   **sign in to webmention.io** via IndieLogin (https://indielogin.com/setup).

Audience: bloggers who want mentions, not a crash course in IndieWeb plumbing.
Written for gardens where the template can't be touched (e.g. forestry.md
hosting), but useful anywhere. A forestry.md HTML snippet can do the same job
(tested 2026-10-01); this plugin is the version with instructions attached.

## Non-goals
- Fetching or displaying mentions. That's `oleeskild/garden-plugin-webmentions`;
  the two plugins are complementary and independently useful.
- Sending webmentions.
- Pingback (possible later option; not v1).
- Email (`mailto:`) or PGP sign-in links. Email is a valid IndieLogin route, but a
  `mailto:` in `<head>` on every page hands the address to scrapers; the README
  mentions it and points to a snippet for people who accept that trade-off.
- Multiple `rel="me"` profiles (one is enough to sign in).

## Key concept (must be explained in the README)
The **account domain** (the webmention.io account that stores mentions) can
differ from the **garden domain** (the site the tag appears on).

Example: bodster.forestry.md advertises `https://webmention.io/bodster.com/webmention`.
Mentions land in the bodster.com account with target URLs on bodster.forestry.md.
The forestry domain never needs its own webmention.io login (IndieAuth), which is
what makes this a workaround for hosted gardens.

## Settings
Manifest `settings` entries (v1.0, unchanged):

```json
{
  "key": "domain",
  "name": "Webmention.io account domain",
  "description": "Domain you log in to webmention.io with. Can differ from this garden's domain.",
  "type": "text",
  "default": "",
  "env": "WEBMENTION_ENDPOINT_DOMAIN"
}
```

Resolution order (loader): value stored in `src/plugins/plugins.json` → env var → `default`.
Set `env` explicitly: without it the loader falls back to an env var named `domain`.
Don't reuse `WEBMENTION_IO_DOMAIN`: the webmentions display plugin reads that, and
there it should hold the *garden* domain, not the account domain.

New in v1.1 (additive, optional):

```json
{
  "key": "relMe",
  "name": "Sign-in profile URL (optional)",
  "description": "Only needed if this garden is the domain you sign in to webmention.io with. Your GitHub, GitLab or Codeberg profile URL; that profile must link back to this garden. Leave empty if you sign in with another site.",
  "type": "text",
  "default": "",
  "env": "WEBMENTION_ENDPOINT_REL_ME"
}
```

The two settings are independent. A newcomer sets `relMe` first, signs in to
webmention.io with the garden's own domain, then sets `domain` to that same domain.

## Behaviour
1. Map a template to the `common.head` slot:
   `"slots": { "common.head": "templates/endpoint.njk" }`.
   Core renders `common.head` on notes and the home page (`note.njk`, `index.njk`);
   the 404 and random-note pages have no plugin slots, which is fine for receiving.
2. Emit exactly:
   `<link rel="webmention" href="https://webmention.io/{domain}/webmention">`
3. Normalise `pluginSettings.domain` first, in the template: trim, lowercase, strip
   `http://`/`https://`, keep only what's before the first `/`, `?` or `#` (this drops
   paths and trailing slashes). `https://Bodster.com/` → `bodster.com`.
4. Output escaping: Nunjucks autoescape is on (Eleventy 3 default, not overridden in
   `.eleventy.js`), so `{{ }}` escapes; no `| safe`, no explicit escape filter needed.
5. If the normalised value is empty **or not hostname-shaped** (`^[a-z0-9.-]+(:[0-9]+)?$`),
   emit **nothing** (no broken endpoint).
6. Files: `garden-plugin.json` + `templates/endpoint.njk` only. No hooks, styles or
   runtime JS (none are required; `dg-link-preview` is a slots-only precedent).
7. **v1.1 `relMe`**, same template, same slot, independent of `domain`:
   - Trim. Emit `<link rel="me" href="{relMe}">` only if it matches
     `^https://[^\s"'<>]+$`. Anything else (empty, `http://`, `mailto:`, bare
     username, spaces) emits nothing.
   - No host allow-list: GitHub, GitLab and Codeberg all work, and IndieLogin's
     provider list may change.
   - Autoescape still applies; no `| safe`.
   - `common.head` renders on the home page, which is where IndieLogin looks.
8. Either tag, both, or neither may be emitted; each is guarded on its own.

## Resolved unknowns (checked against the skill and `src/helpers/pluginLoader.js`, 2026-09-30)
- **Manifest schema**: required `id`, `name`, `version`, `description`, `author`.
  Optional `slots` (`{ "<slot>": "file.njk" }` or a list), `settings` (see above).
  Declared paths must be relative, `/`-separated, no `..`.
- **Reading a setting in a slot template**: `pluginSettings.domain`
  (bound per plugin by `components/pluginSlot.njk`).
- **Normalisation**: template-only, with Nunjucks `trim`, `lower`, `replace` (regex
  literal `r/^https?:\/\//`) and JS string methods such as `.split("/")[0]`.
  No JS filter/hook needed. Confirm the regex literal in test 2.
- **No hooks entrypoint**: valid; `hooks` is optional.
- **Folder name == id**: enforced; mismatch → plugin skipped with a `[plugins]` warning.
  Install from GitHub creates `src/plugins/<id>/` (confirmed by `plugins.json` records,
  e.g. `bodster7/garden-plugin-croutons` → `src/plugins/croutons/`).
- **`dg-` prefix**: reserved for first-party plugins. The loader doesn't enforce it,
  but the skill forbids it, and `npm test` treats `src/plugins/dg-*` as core.
  Hence the id `webmention-endpoint`.

## Resolved unknowns (v1.1, from Ian's clooeygooey.com sign-ins, 2026-10-01)
- **Link back from GitHub**: a GitHub profile **social link** works, not only the
  **Website** field. So one GitHub account can vouch for several sites (Website field
  for one, social links for the others). Verified for GitHub only; GitLab and Codeberg untested.
- **Second domain = separate account**: signing in to webmention.io with a new domain
  creates a separate account with its own token (clooeygooey.com, done twice).

## Test plan
1. **Local build** (bodster_garden, branch `dg-webmention-endpoint-dev`): run the dev build,
   then `curl -s http://localhost:8080/ | findstr webmention`.
   Expect two tags: the hand-added user component
   (`components/user/common/head/webmention.njk`) and the plugin's. Same `href`;
   the hand-added one ends ` />`, the plugin's `>`, so they're equivalent, not byte-identical.
   Also check the build log has no `[plugins]` warnings, and that disabling the plugin
   (`plugins.json` → `"webmention-endpoint": { "enabled": false }`) leaves only the hand-added tag.
2. **Normalisation**: try `https://Bodster.com/`, `bodster.com/`, `bodster.com/path?q=1`,
   `bad domain`, empty. Check the output each time; the last two must produce no tag.
3. **Forestry**: tag `v0.1.0`, install from GitHub on bodster.forestry.md, then
   `curl.exe -s https://bodster.forestry.md/ | findstr webmention`.
   Without a GitHub *Release* (a tag alone isn't one), installers use `main`.
4. **End-to-end**: link to a forestry page from another page (any site, including the
   forestry garden itself), send the mention (curl to the endpoint, or Telegraph), confirm it appears at
   `https://webmention.io/api/mentions.jf2?domain=bodster.forestry.md&token=<key>`.

Notes from running it:
- In Windows PowerShell, `curl` is an alias for `Invoke-WebRequest`; use `curl.exe`.
- The source page must link to the target with an **absolute** URL. webmention.io rejected
  a relative `href="/projects/project-a/"` with `no_link_found`.
- A `201`/`queued` reply only means received. Check the `location` status URL
  (no token needed) for `success` or the rejection reason; rejected mentions aren't stored.
- webmention.io accepts mentions where source and target are on the same site.
- The dev build can abort on Windows with `EBUSY` copying `dist/favicon.svg`
  (`eleventy-plugin-gen-favicons` copies it on every page render in parallel). Garden issue, not this plugin.

v1.1 additions:

5. **`relMe` validation**: `https://github.com/bodster7`, ` https://github.com/bodster7 `,
   `http://github.com/bodster7`, `mailto:x@example.com`, `bodster7`, `bad url`, empty.
   Only the first two emit a tag. Check `domain` set/unset doesn't affect it (and vice versa).
6. **Real sign-in (Path B), optional**: the sign-in route is already proven on
   clooeygooey.com; this test proves the *plugin* produces a working `rel="me"`.
   On bodster.forestry.md, set `relMe` to the GitHub profile, add
   `https://bodster.forestry.md/` as a GitHub **social link** (leave the Website field
   alone), sign in to webmention.io as `bodster.forestry.md`. Note this creates a
   separate webmention.io account; the forestry garden currently uses the bodster.com
   account, so leave `domain` as is unless you deliberately switch.

Results (2026-09-30), all passing:
1. Tag present in local build; gone with the domain unset; no `[plugins]` warnings.
2. All normalisation cases as expected (rendered with the garden's Nunjucks, autoescape on).
3. bodster.forestry.md: exactly one tag on the home page and 7 note pages.
4. Mention from `/projects/project-a/plan-a/` to `/projects/project-a/` stored in the
   bodster.com account (`wm-id` 2036737) and returned by the `domain=bodster.forestry.md` query
   and the dashboard.
5. (2026-10-01) All seven listed `relMe` inputs as expected, plus edge cases (quotes, `<`,
   spaces, `HTTPS://`, bare `https://`; `&` escaped and emitted); `domain` and `relMe`
   independent; v1.0 domain cases unchanged. Real garden build with both settings via env:
   no `[plugins]` warnings, trimmed `rel="me"` present. (bodster_garden also has a
   hand-added `indieauth.njk`, so it shows two `rel="me"` tags, like test 1's duplicate.)
6. Skipped by decision (2026-10-01); the sign-in route itself is proven on clooeygooey.com.

## Release
- `v0.1.0` once forestry test passes. Tagged 2026-09-30 (tag only, no GitHub Release).
- `v1.0.0` + gallery PR to `oleeskild/digitalgarden-plugins` after a soak period.
  Soak done 2026-10-01 (further testing on bodster.forestry.md and bodster.com); released as `v1.0.0`
  with a GitHub Release, which installers need to pick a version over `main`.
- `v1.1.0` (minor: additive optional setting) with a GitHub Release once test 5 passes
  (test 6 optional; skipped 2026-10-01).
  Submit the gallery PR at v1.1.0 rather than v1.0.0, so the listing launches with both paths.
- Repo: `garden-plugin-webmention-endpoint`, standalone, MIT licence.
- `screenshot.png`: the plugin has no visible UI, so use an illustration
  (e.g. the emitted tag in view-source), as the analytics plugin did.

## README must include
Write for the blogger, not the protocol. Order:
1. **One-line blurb**: e.g. "Receive webmentions on your Digital Garden by filling in
   one box, with no template editing." Reuse it as the manifest `description` and gallery blurb.
2. **The problem**: webmention.io tells you to paste a line of HTML into your site's
   `<head>`; on a hosted garden you don't know where that is (and may not be able to reach it).
3. **Pick your path**:
   - *Path A, already have a webmention.io account* (e.g. with your main site):
     set **account domain** to that domain. Done.
   - *Path B, this garden is your only site*: set **sign-in profile URL**; add your garden
     URL to that profile (**this is the one people miss**); sign in at webmention.io
     with your garden's domain; set **account domain** to the same domain.
     Tip: the link back can go in the profile's **Website** field *or* one of its
     **social links**, so your website field stays free for your main site.
     This creates a separate webmention.io account (and token) for the garden.
     Link https://indielogin.com/setup for the details.
4. **Account domain vs garden domain**: the explanation above, with the bodster example.
5. **Pairing with the webmentions display plugin**: token from the account domain;
   display plugin's domain = the garden's own domain.
6. **Checking it works**: view source / `curl.exe` + `findstr`; the absolute-URL and
   `location` status-URL notes from testing.
7. **Not for you if**: your template already has the tags (duplicates); or you want
   email sign-in (use a snippet and accept the scraper trade-off).
8. **For the curious**: a forestry.md HTML snippet does the same job; this plugin is
   that, plus the instructions and the guardrails.