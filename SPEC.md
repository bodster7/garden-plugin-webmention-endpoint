\# SPEC: garden-plugin-dg-webmention-endpoint



\- Plugin id: `dg-webmention-endpoint` (manifest `id`, folder name and gallery `id` all identical)

\- Dev location: `bodster\_garden/src/plugins/dg-webmention-endpoint/` (own git repo, ignored by the garden)

\- Garden branch: `dg-webmention-endpoint-dev`



\## Purpose

Advertise a webmention.io receiving endpoint on every page of a Digital Garden,

so the garden can \*\*receive\*\* webmentions without editing the site template.



Written for gardens where the template can't be touched (e.g. forestry.md

hosting), but useful anywhere.



\## Non-goals

\- Fetching or displaying mentions. That's `oleeskild/garden-plugin-webmentions`;

&#x20; the two plugins are complementary and independently useful.

\- Sending webmentions.

\- Pingback (possible later option; not v1).



\## Key concept (must be explained in the README)

The \*\*account domain\*\* (the webmention.io account that stores mentions) can

differ from the \*\*garden domain\*\* (the site the tag appears on).



Example: bodster.forestry.md advertises `https://webmention.io/bodster.com/webmention`.

Mentions land in the bodster.com account with target URLs on bodster.forestry.md.

The forestry domain never needs its own webmention.io login (IndieAuth), which is

what makes this a workaround for hosted gardens.



\## Setting

| key | label | type | default |

|---|---|---|---|

| `domain` | Webmention.io account domain | string | empty |



Help text: "Domain you log in to webmention.io with. Can differ from this garden's domain."



\## Behaviour

1\. Map a template to the `common.head` slot (same slot the analytics plugin uses).

2\. Emit exactly:

&#x20;  `<link rel="webmention" href="https://webmention.io/{domain}/webmention">`

3\. Normalise `domain` first: trim, lowercase, strip `http://`/`https://`,

&#x20;  strip any path and trailing slashes. `https://Bodster.com/` → `bodster.com`.

4\. HTML-escape the output.

5\. If `domain` is empty after normalisation, emit \*\*nothing\*\* (no broken endpoint).

6\. No hooks entrypoint, styles or runtime JS unless the plugin API requires them.



\## Known unknowns (Claude Code: verify against the skill before coding)

\- Exact manifest schema for settings and slots.

\- How a slot template reads a plugin setting.

\- Whether normalisation can live in the template (Nunjucks filters) or needs a

&#x20; hook/filter in JS.

\- Whether a plugin with no hooks entrypoint is valid.

\- Whether the loader requires folder name == manifest id, and what folder

&#x20; "Install from GitHub" creates (expected: `src/plugins/<id>/`).

\- Whether the `dg-` id prefix is reserved or conventional for core plugins

&#x20; (core uses e.g. `dg-search`). If so, flag it before we commit to the name.



\## Test plan

1\. \*\*Local build\*\* (bodster\_garden, branch `dg-webmention-endpoint-dev`): run the dev build,

&#x20;  then `curl -s http://localhost:8080/ | findstr webmention`.

&#x20;  Expect two tags: the hand-added slot one and the plugin's. Both should be identical.

2\. \*\*Normalisation\*\*: try `https://Bodster.com/`, `bodster.com/`, empty. Check the output

&#x20;  each time; empty must produce no tag.

3\. \*\*Forestry\*\*: tag `v0.1.0`, install from GitHub on bodster.forestry.md, then

&#x20;  `curl -s https://bodster.forestry.md/ | findstr webmention`.

4\. \*\*End-to-end\*\*: link to a forestry page from a bodster.com note, send the mention

&#x20;  (curl to the endpoint, or Telegraph), confirm it appears at

&#x20;  `https://webmention.io/api/mentions.jf2?domain=bodster.forestry.md\&token=<key>`.



\## Release

\- `v0.1.0` once forestry test passes.

\- `v1.0.0` + gallery PR to `oleeskild/digitalgarden-plugins` after a soak period.

\- Repo: `garden-plugin-dg-webmention-endpoint`, standalone, MIT licence.

\- `screenshot.png`: the plugin has no visible UI, so use an illustration

&#x20; (e.g. the emitted tag in view-source), as the analytics plugin did.



\## README must include

\- What it does / doesn't do, and pairing with the webmentions display plugin

&#x20; (token from the account domain; display plugin's domain = the garden's own domain).

\- The account-domain vs garden-domain explanation above.

\- Warning: don't install it on a site that already has the link in its template

&#x20; (duplicate tags).

