# SPEC: garden-plugin-webmention-endpoint

- Plugin id: `webmention-endpoint` (manifest `id`, folder name and gallery `id` all identical).
  Not `dg-`: that prefix is reserved for first-party plugins (garden-plugin-author skill).
- Dev location: `bodster_garden/src/plugins/webmention-endpoint/` (own git repo, ignored by the garden)
- Garden branch: `dg-webmention-endpoint-dev` (branch name only; kept as is)

## Purpose
Advertise a webmention.io receiving endpoint on every page of a Digital Garden,
so the garden can **receive** webmentions without editing the site template.

Written for gardens where the template can't be touched (e.g. forestry.md
hosting), but useful anywhere.

## Non-goals
- Fetching or displaying mentions. That's `oleeskild/garden-plugin-webmentions`;
  the two plugins are complementary and independently useful.
- Sending webmentions.
- Pingback (possible later option; not v1).

## Key concept (must be explained in the README)
The **account domain** (the webmention.io account that stores mentions) can
differ from the **garden domain** (the site the tag appears on).

Example: bodster.forestry.md advertises `https://webmention.io/bodster.com/webmention`.
Mentions land in the bodster.com account with target URLs on bodster.forestry.md.
The forestry domain never needs its own webmention.io login (IndieAuth), which is
what makes this a workaround for hosted gardens.

## Setting
Manifest `settings` entry:

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

Results (2026-09-30), all passing:
1. Tag present in local build; gone with the domain unset; no `[plugins]` warnings.
2. All normalisation cases as expected (rendered with the garden's Nunjucks, autoescape on).
3. bodster.forestry.md: exactly one tag on the home page and 7 note pages.
4. Mention from `/projects/project-a/plan-a/` to `/projects/project-a/` stored in the
   bodster.com account (`wm-id` 2036737) and returned by the `domain=bodster.forestry.md` query
   and the dashboard.

## Release
- `v0.1.0` once forestry test passes. Tagged 2026-09-30 (tag only, no GitHub Release).
- `v1.0.0` + gallery PR to `oleeskild/digitalgarden-plugins` after a soak period.
  Soak done 2026-10-01 (further testing on bodster.forestry.md and bodster.com); released as `v1.0.0`
  with a GitHub Release, which installers need to pick a version over `main`.
- Repo: `garden-plugin-webmention-endpoint`, standalone, MIT licence.
- `screenshot.png`: the plugin has no visible UI, so use an illustration
  (e.g. the emitted tag in view-source), as the analytics plugin did.

## README must include
- What it does / doesn't do, and pairing with the webmentions display plugin
  (token from the account domain; display plugin's domain = the garden's own domain).
- The account-domain vs garden-domain explanation above.
- Warning: don't install it on a site that already has the link in its template
  (duplicate tags).