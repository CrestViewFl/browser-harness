# Instagram — Scrape Profile Reels + Per-Reel Detail

URLs:

- Profile root: `https://www.instagram.com/<handle>/`
- Profile reels tab: `https://www.instagram.com/<handle>/reels/`
- Single reel: `https://www.instagram.com/reel/<shortcode>/`
- Single post: `https://www.instagram.com/p/<shortcode>/`

Always navigate via `new_tab(url)`, never `goto(url)`. Goto runs in whatever the user's active tab is and clobbers their work.

## Prerequisites

- Logged into Instagram in the Chrome profile browser-harness is attached to. If not logged in, the profile root will redirect to `/accounts/login/?next=/<handle>/` and the tile selectors below return zero rows.
- Pacing: 3-7s human delay between profile loads. IG starts soft-throttling around 200 profile loads / hour from a single session, and the 429 surface is silent (you get an empty grid, not an error).

## What the rendered page actually exposes

Instagram's React app no longer ships `window._sharedData`. Data is loaded dynamically. The reliable readout paths are:

| Field | Where |
|---|---|
| Follower count | `<meta property="og:description">` content, regex `([\d.,KMB]+)\s+Followers` |
| Display name | `document.title` ("Display Name (@handle) on Instagram") |
| Post grid | `<a>` elements where `a.href` matches `/<handle>/(p|reel)/<shortcode>/` (see selector note below) |
| Tile caption fallback | `<img>` `alt` attribute inside the tile `<a>` (often holds the first ~100 chars of the caption verbatim) |
| Caption (detail page) | og:description after the colon (most reliable when logged-in); JSON-LD `caption` when present |
| Posted at | `time[datetime]` attribute on the detail page, or JSON-LD `uploadDate` |
| Like count | og:description text `"X likes, Y comments - handle on DATE: \"caption\""` (most reliable when logged in); JSON-LD `interactionStatistic` → `LikeAction` → `userInteractionCount` if JSON-LD shipped |
| Comment count | Same og:description text; or JSON-LD `CommentAction` |
| Video src | `document.querySelector('video').src` or `currentSrc` (returns a `blob:` URL, not the CDN URL — use yt-dlp for download) |
| Duration | `document.querySelector('video').duration` |

**Important:** when logged in, Instagram serves a different chrome than logged-out. JSON-LD `script[type="application/ld+json"]` is typically ABSENT on logged-in reel detail pages. The og:description text block has the like + comment counts AND the caption, formatted as:

```
29 likes, 9 comments - leifabel11 on June 9, 2026: "He always calls shotgun.". 
```

Parse it with two simple regexes:
- `([\d,]+)\s+likes?` for like_count
- `([\d,]+)\s+comments?` for comment_count
- `:\s*"(.+?)"\.?\s*$` for caption (multiline, capture group)

Do NOT rely on JSON-LD alone — most of the test sample (logged in, June 2026) had no JSON-LD on reel pages.

## The selector trap that will eat your day

`a[href^="/p/"]` and `a[href^="/reel/"]` return **zero** matches on a real profile grid. IG renders the post tile anchors with `href="#"` literally in the HTML and computes the navigation programmatically. The resolved `a.href` property holds the real URL — but it is in the form `/<handle>/p/<shortcode>/` or `/<handle>/reel/<shortcode>/`, not `/p/<shortcode>/`.

The reliable selector is "every anchor whose resolved `.href` matches the path regex," done in JS:

```javascript
(() => {
  const re = /\/([a-zA-Z0-9_.]+)\/(p|reel)\/([A-Za-z0-9_-]+)/;
  const anchors = Array.from(document.querySelectorAll('a'));
  const seen = new Set();
  const tiles = [];
  for (const a of anchors) {
    const m = a.href.match(re);
    if (!m) continue;
    const shortcode = m[3];
    if (seen.has(shortcode)) continue;
    seen.add(shortcode);
    const img = a.querySelector('img');
    tiles.push({
      href: a.href,
      shortcode,
      type: m[2] === 'reel' ? 'reel' : 'post',
      alt: img?.alt || null
    });
  }
  const og = (sel) => document.querySelector(sel)?.content || null;
  return {
    handle: location.pathname.split('/').filter(Boolean)[0],
    title: document.title,
    og_description: og('meta[property="og:description"]'),
    tiles
  };
})()
```

After `new_tab` + `wait_for_load()`, sleep ~2.5s for the React app to mount, then scroll 3-4 viewports (`window.scrollBy(0, window.innerHeight * 1.5)`) to load more grid tiles. Without the scroll you get the initial 12. On `@leifabel11` this returns 67 tiles after 2 scrolls — sufficient for any 30-90 day window.

**The IIFE wrapping is also load-bearing.** `browser-harness js()` calls `Runtime.evaluate(returnByValue=True)`. A bare arrow function `() => {...}` evaluates to a function object — you get the function back, not its return value. Always wrap as `(() => {...})()` so the function is invoked.

## Per-reel detail — what to grab

Navigate to the reel URL, sleep ~2.5s, then read JSON-LD + meta + `<video>`. The reel page mounts the `<video>` element before the JSON-LD in most cases, so wait for `document.querySelector('video')?.readyState >= 2` before screenshotting.

## Reel screenshots — the lazy-load trap

A fixed `sleep(2)` after navigating to a reel will miss the first frame about half the time. Do this instead:

1. Wait until `document.querySelector('video')?.readyState >= 2` (HAVE_CURRENT_DATA). Poll every 500ms up to 8s.
2. To seek, set `video.currentTime = t` and listen for the `seeked` event with `{once: true}`. Resolve a Promise inside `js(...)` so Python knows when to call `screenshot()`.
3. After the seek event fires, wait one `requestAnimationFrame` cycle (double-rAF is safer) before capturing — the compositor needs a frame to repaint.

## Stable selectors and patterns

- The `<article>` grid is the only stable container. Class names are CSS-module hashes that rotate. Selecting on `a[href^="/p/"]` is more durable.
- `og:description` is stable across IG profile redesigns. Use it for follower count.
- JSON-LD on reels is the only reliable like/comment-count source from the DOM. Counts inside the React rendered chrome are aria-labeled with text like "Liked by X and 12,345 others" which truncates.

## Rate-limit and throttling signs

- Empty tile grid on a profile that visibly has posts → you've been soft-throttled. Back off 30 minutes.
- HTTP 429 on the underlying `/api/v1/feed/user/.../` calls (visible in `cdp("Network.requestWillBeSent")` if you wire that listener) → hard throttle. Back off 2-4 hours.
- A login redirect when you were logged in → session expired. Stop and prompt the user to log back in in Chrome.

## Selectors that DO NOT work

- `a[href^="/p/"], a[href^="/reel/"]` for post tiles — see the IG URL pattern note above. Tiles use `/<handle>/p/<shortcode>/` not `/p/<shortcode>/`. Use the regex-on-`.href` approach instead.
- `[role="button"][aria-label="Like"]` for like count — IG replaced aria-label with a localized string ("Me gusta" in Spanish builds) AND removed the count.
- `_ac7v` and other obfuscated CSS-module classes — rotate per build.
- `window._sharedData` — gone since 2024.
- `script[type="application/ld+json"]` alone for like/comment counts — frequently absent on logged-in reel pages. Always have an og:description text fallback.

## Video download

`yt-dlp --cookies-from-browser chrome <reel-url>` works for any reel the logged-in user can view. The `--cookies-from-browser` flag reads the live Chrome cookie store; no need to export cookies to disk.

For audio: extract with `ffmpeg -i in.mp4 -vn -ac 1 -ar 16000 -codec:a libmp3lame -q:a 4 out.mp3`. 16kHz mono is what Whisper resamples to anyway.

## Things you'll lose without admitting it

- View counts on reels: IG hides them on some accounts (creator vs personal) and they're not in JSON-LD. The `<video>` element has no playCount property.
- Save count: not in any DOM-exposed source. Only available through IG's own creator analytics.
- Comment text: requires expanding the comments tray (lazy-rendered, multiple scroll passes). For a research tool we only need the count, not the text — skip.

## Reels tab: the grid is virtualized (learned 2026-09-10, @mirandamorey_, 257 posts)

`/<handle>/reels/` keeps only ~40-65 tile anchors in the DOM at any time and drops the ones that
scrolled far above the viewport. A single snapshot after scrolling therefore caps out around 45 tiles
even when the account has 100+ reels. Collect a **union across scroll steps**: re-run the tile regex
after every `window.scrollBy(0, innerHeight * 1.2)` and merge by shortcode; stop after 6 scrolls with
no new shortcode. 90 unique reels took 8 scroll steps at 1.6-2.6 s pacing. Insertion order of the
union equals grid order.

- The first tiles on the reels tab can be **pinned** reels, which are older than everything below them.
  Do not assume grid order is chronological; decode dates from the shortcode instead.
- **Shortcode to date**: shortcode is base64url (`A-Za-z0-9-_`) of the media id;
  `ts_ms = (media_id >> 23) + 1314220021721` gives the post time (accurate to the day). Use it to
  sort "the N newest" without opening any reel.
- Tile `innerText` on the reels tab is the view count (e.g. "8,000"); `img.alt` is often null there,
  unlike the posts grid.
- Logged-in detection: `document.title` starts with "(N) Instagram" (notification count) when a
  session exists; a logged-out load of `/reels/` redirects to `/accounts/login/`.

## Downloading without cookies

`yt-dlp` (via the shared `watch` skill) downloads public reels with no cookies at roughly 8 s per reel
including frame extraction. Expect a few per account to fail with
`This content isn't available to everyone: It can't be seen by certain audiences.` These are
age-restricted posts (alcohol partnerships on this account). They need a logged-in session; back-fill
from the next-oldest reels instead of borrowing cookies unless the owner approves.
