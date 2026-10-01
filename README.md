# Webmention Endpoint

Receive webmentions on your [Digital Garden](https://github.com/oleeskild/digitalgarden) without editing its template.

![The webmention tag in a page's head](screenshot.png)

## The problem

[Webmentions](https://indieweb.org/Webmention) let other sites tell yours when they link to it: replies, likes, reposts, plain mentions. [webmention.io](https://webmention.io) will collect them for you, and its setup page says to paste a line like this into your site's `<head>`:

```html
<link rel="webmention" href="https://webmention.io/example.com/webmention">
```

On a hosted garden, such as one on [Forestry.md](https://forestry.md), you don't know where the `<head>` is, and may not be able to reach it. This plugin adds that line to every note and your home page for you. You fill in a setting instead.

## Pick your path

First, install the plugin: paste this repo's URL into **Install from GitHub** (in Obsidian: Settings → Digital Garden → Plugins), or copy this folder to `src/plugins/webmention-endpoint/` in your garden. Then follow one of these.

### Path A: you already have a webmention.io account

For example, you signed in to webmention.io with your main website.

1. Set **Webmention.io account domain** to that domain, e.g. `example.com`.

That's it. Mentions of your garden's pages land in that account. See [Account domain vs garden domain](#account-domain-vs-garden-domain) for why that works.

### Path B: this garden is your only site

You need to sign in to webmention.io with your garden's own domain. webmention.io signs you in with [IndieLogin](https://indielogin.com/setup), which checks that your site and a profile elsewhere point at each other.

1. Set **Sign-in profile URL** to your GitHub, GitLab or Codeberg profile, e.g. `https://github.com/you`. Publish your garden.
2. **Add your garden's URL to that profile.** This is the step people miss. The link back can go in the profile's **Website** field *or* one of its **social links**, so your Website field stays free for your main site. (Social links are confirmed to work on GitHub; GitLab and Codeberg are untested.)
3. Sign in at [webmention.io](https://webmention.io) with your garden's domain, e.g. `you.forestry.md`.
4. Set **Webmention.io account domain** to that same domain, and publish again.

This creates a webmention.io account (and API token) just for the garden. If you later sign in with another domain, that's a separate account with its own token.

The sign-in profile URL must start with `https://`. Anything else is ignored, and no sign-in link is added.

### Settings

| Setting | Default | |
|---|---|---|
| Webmention.io account domain | empty | Domain you log in to webmention.io with. Can differ from this garden's domain. |
| Sign-in profile URL (optional) | empty | Only needed for Path B: your GitHub, GitLab or Codeberg profile URL. That profile must link back to this garden. |

You can paste a URL as the account domain: `https://Example.com/` is tidied up to `example.com`. While a setting is empty or invalid, the plugin adds nothing for it, so a half-configured garden never advertises a broken endpoint. The two settings are independent.

Both can also be set with environment variables, `WEBMENTION_ENDPOINT_DOMAIN` and `WEBMENTION_ENDPOINT_REL_ME`. A value saved in the plugin settings takes priority.

## Account domain vs garden domain

Two domains are involved, and they don't have to be the same:

- The **account domain** is the one you sign in to webmention.io with. Your mentions are stored in that account.
- The **garden domain** is the site the tags appear on.

For example, the garden at `bodster.forestry.md` advertises `https://webmention.io/bodster.com/webmention`. Mentions of its pages land in the `bodster.com` webmention.io account, with target URLs on `bodster.forestry.md`. The forestry domain never needs its own webmention.io sign-in.

On Path B, the two domains are simply the same.

## Showing mentions

This plugin only receives. To show mentions under your notes, also install [Webmentions](https://github.com/oleeskild/garden-plugin-webmentions) and set it up like this:

- **Domain:** your **garden** domain (e.g. `bodster.forestry.md`), because mentions are looked up by the page they point at.
- **API token:** the token from your **account** domain's webmention.io dashboard (e.g. the `bodster.com` account).

## Checking it works

View the source of your home page or any note and look for `rel="webmention"` (and `rel="me"`, on Path B). Or from a terminal:

```sh
curl -s https://your-garden.example/ | grep webmention
```

On Windows PowerShell, use `curl.exe` (plain `curl` is a different command there) and `findstr` instead of `grep`.

To send yourself a test mention, you need a page that links to one of your garden's pages with a **full URL** (`https://your-garden.example/some-note/`). webmention.io ignores relative links like `/some-note/`. Then:

```sh
curl -i https://webmention.io/example.com/webmention -d source=https://the-linking-page/ -d target=https://your-garden.example/some-note/
```

A `201` reply only means the mention was queued. Open the URL in its `location` header to see whether it was accepted, or why it was rejected. Rejected mentions aren't stored, so they never show up on your dashboard.

## Not for you if

- **Your template already has these tags**, including ones added as custom head components. You'd get duplicates. Remove the existing ones or skip this plugin.
- **You want to sign in by email.** IndieLogin can do that with a `mailto:` link, but this plugin deliberately doesn't offer it: the link would be on every page, where address-harvesting scrapers find it. If you accept that, add `<link rel="me" href="mailto:you@example.com">` to your garden's head yourself.

## For the curious

There's nothing magic here. If your host lets you add an HTML snippet to every page's `<head>` (Forestry.md does), these two lines do the same job:

```html
<link rel="webmention" href="https://webmention.io/example.com/webmention">
<link rel="me" href="https://github.com/you">
```

This plugin is that, plus the instructions and the guardrails: it tidies the domain, refuses anything that isn't a valid domain or `https://` URL, and adds nothing until you've filled it in.

## Licence

MIT. See [LICENSE](LICENSE).
