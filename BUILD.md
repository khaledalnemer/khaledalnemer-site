# Build & deploy

Static site. The only build step is Tailwind: it turns the classes used in
`index.html` into `styles.css`. Everything else is committed as-is and served
directly by DigitalOcean App Platform from `main`.

## Layout

The repo root holds **only files that get served** — HTML, `styles.css`,
images, the PDF, `robots.txt`, `sitemap.xml`. All build tooling lives in
`tooling/`.

That separation is deliberate. When `package.json` sat in the root,
DigitalOcean stopped deploying: App Platform inspects the root of a static
site and a `package.json` there makes it treat the repo as a Node app to
build rather than a directory to serve. Keeping the root free of manifests
leaves it detectable as plain static content again.

**Do not move `tooling/package.json` back to the root.**

## Requirements

Node 18+ (developed on Node 24, npm 11).

## Build

```bash
cd tooling
npm install     # first time only
npm run build   # writes ../styles.css
```

`npm run watch` rebuilds on save while you edit.

**Run `npm run build` and commit `styles.css` whenever you add, remove or change
a Tailwind class in `index.html`.** The stylesheet only contains the utilities
actually found in the file, so a new class is invisible until you rebuild.
`styles.css` is committed on purpose — DigitalOcean serves the repo as-is and
never runs this build.

## Why there is a build step

The site used to load `cdn.tailwindcss.com` (the Play CDN) and compile CSS in
the visitor's browser. That cost 407 KB of render-blocking JavaScript, tied the
appearance to a floating version, and — the reason it was replaced — silently
ignored class names it did not recognise. A misspelled `inset-inline-start-0`
(the real utility is `start-0`) produced no CSS and no warning; the accent bar
it was meant to position fell back to its static position and overlapped the
role title in both languages. A build makes that class of mistake catchable.

To check for classes that produce no CSS, run this in the browser console:

```js
const used = new Set();
document.querySelectorAll('*').forEach(el => {
  if (typeof el.className === 'string')
    el.className.split(/\s+/).forEach(c => c && used.add(c));
});
const defined = new Set();
const walk = rules => { for (const r of rules) {
  if (r.selectorText) (r.selectorText.match(/\.(?:\\.|[^\s.,:>+~[)])(?:\\.|[^\s.,:>+~[)])*/g) || [])
    .forEach(s => defined.add(s.slice(1).replace(/\\/g, '').trim()));
  if (r.cssRules) walk(r.cssRules);
}};
for (const s of document.styleSheets) { try { walk(s.cssRules); } catch (e) {} }
console.log([...used].filter(c => !defined.has(c)));  // expect []
```

## Cascade order matters

`styles.css` is linked **after** the inline `<style>` block in `index.html`.
The Play CDN used to inject its stylesheet at the end of `<head>`, so Tailwind
utilities won ties against the custom rules above them. Keeping that order
preserves the existing rendering exactly. If you move the `<link>` earlier,
custom rules start winning and the layout will shift.

Two rules in the mobile media query are inert because of this and always were —
`.hero-meta { align-items: flex-start }` loses to `items-center`, and
`section { padding: 2.5rem 0 }` loses to `py-12`. Change the utility in the
markup rather than the CSS if you want different behaviour there.

## After deploying

DigitalOcean rebuilds on push to `main` and invalidates the Cloudflare cache;
`/` returned `cf-cache-status: MISS` immediately after the last deploy. If an
update ever appears stale, the edge holds HTML for up to 24 h
(`s-maxage=86400`) — purge the Cloudflare cache to force it.

## Not done here

- **Security headers.** No `Content-Security-Policy`, `Strict-Transport-Security`,
  `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` or
  `Permissions-Policy` are sent. These are set in the DigitalOcean app spec or
  Cloudflare, not in this repo. Now that Tailwind is built and lucide is pinned,
  the only remaining script origin is `unpkg.com`, so a CSP is finally practical.
- **Separate `/ar/` and `/en/` URLs with `hreflang`.** Arabic is still applied
  by JavaScript on the same URL, so there is no shareable Arabic link and
  crawlers only ever see English.
- **The PDF** (`Khaled_Alnemer_Resume.pdf`) is maintained outside this repo and
  still disagrees with the HTML: it contains the placeholder
  `Albilad Bank, Location`, omits the Nov 2025 acting role, and uses a different
  Tatweer job title.
