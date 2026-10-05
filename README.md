# Case study: making a WordPress + Divi portal 3× faster on shared hosting

**TL;DR**: a content-heavy WordPress portal went from a **2.9 s** server response to **0.8 s**, its mobile Lighthouse
score went from **28 to ~50**, and LCP went from **16–20 s to ~4 s**. The theme, the hosting and the design all stayed
the same. The work took one day and was done in an agentic workflow with Claude Code.

> This is a write-up of work I did for my employer. The site is not named and no proprietary code is included.
> The snippets below are generic examples written for this case study.

---

## Context

- An Italian university-admissions portal: WordPress, the Divi theme, several custom plugins, Contact Form 7 lead forms
  sent to a CRM, and a GDPR consent manager.
- Shared Apache hosting (no LiteSpeed, no server-level cache, no compression), so moving to a new host wasn't possible.
- Hard constraints: **the design could not change**, forms and lead tracking had to keep working, and nothing could
  load before the user's cookie consent.

## Starting point

| Metric | Before |
|---|---|
| Server response (HTML) | 2.9 s |
| HTML size | 232 KB (uncompressed) |
| Page weight (home) | 3.4 MB |
| Requests | 87 |
| Lighthouse mobile score | 28 |
| LCP (mobile) | 16–20 s |

Mobile measurements used Lighthouse with **CPU throttling ×12** instead of the default ×4, because the default
setting made the site look better than it felt on mid-range Android phones.

## What I did

### 1. Page cache: one setting decided everything
I installed WP Super Cache in simple mode, so `.htaccess` stayed untouched. The trap was the option that disables
caching for "known users". It treats **any visitor with a cookie** as known, and with a consent banner almost every
visitor has a cookie, so the cache would have served almost nobody. Caching is skipped only for logged-in users.

Also:
- 22 campaign parameters (`utm_*`, `fbclid`, `gclid`, `ttclid`…) are ignored, so ad traffic hits the cache
- pages whose form carries a short-lived security token (booking form) are excluded
- I checked that 404s, sitemaps and feeds are not cached

**Result: 2.9 s → 0.8 s.** A second portal on the same hosting went from 2.6 s to 0.45 s.

### 2. Compression and browser caching
The server sent everything uncompressed. A `mod_deflate` + `mod_expires` block, placed above the WordPress rules:

```apache
<IfModule mod_deflate.c>
  AddOutputFilterByType DEFLATE text/html text/css application/javascript application/json image/svg+xml
</IfModule>
<IfModule mod_expires.c>
  ExpiresActive On
  ExpiresByType image/webp "access plus 1 year"
  ExpiresByType font/woff2 "access plus 1 year"
  ExpiresByType text/css "access plus 1 year"
  ExpiresByType application/javascript "access plus 1 year"
</IfModule>
```

A one-year expiry is safe only because every CSS and JS file carries a version query string.
**HTML: 232 KB → 39 KB.**

### 3. Load assets only where they're used
Several plugins loaded CSS and JS on every page. They now load only on pages that contain their shortcode. I also
removed an unused date picker and a font outside the brand guidelines. **Requests: 87 → 63.**

### 4. The LCP image and a misplaced viewport tag
The hero was a 342 KB JPEG with a parallax effect. I replaced it with two WebP files (216 KB for desktop, 80 KB for
mobile) and added a `preload` on the home page only, with a media query.

Then the hidden problem: the theme printed `<meta name="viewport">` **at the end of `<head>`**. Until the browser
reads that tag, a phone evaluates media queries as if the screen were 980 px wide, so it also downloaded the
desktop image. Moving the tag to the top fixed it:

```php
// Print the theme's viewport meta first in <head>, before any preload or stylesheet.
remove_action( 'wp_head', 'theme_add_viewport_meta' );
add_action( 'wp_head', 'theme_add_viewport_meta', 0 );
```

**LCP: ~8.4 s → ~4.1 s.** The image's wait went from 9.7 s to 1.6 s.

### 5. reCAPTCHA on first interaction, and a consent-manager trap
reCAPTCHA v3 loaded on every page. Now it loads the first time the user touches a form, and form submission waits
for the token.

The first version broke silently. The consent manager **blocks inline scripts that contain the string
`grecaptcha`** until the visitor accepts cookies. My inline loader never ran, and forms were sent **without a
token**, so they looked like spam. I caught it during testing by intercepting the submissions, and rolled it back
within minutes. The fix was to move the logic to an external `.js` file and keep only configuration inline. I
retested with direct submission, after a tap, from cache and on the home page.

## Results

| Metric | Before | After |
|---|---|---|
| Server response | 2.9 s | **0.8 s** |
| HTML size | 232 KB | **39 KB** |
| Page weight | 3.4 MB | **~1 MB** |
| Requests | 87 | **63** |
| Lighthouse mobile (CPU ×12) | 28 | **47–50** |
| FCP | 5.1 s | **2.5 s** |
| LCP | 16–20 s | **~4.1 s** |

What's left is mostly the consent manager (~2.4 s of main-thread blocking). It can't be removed for legal reasons.

## Lessons

1. **Read what "known user" means** in a cache plugin. With a consent banner, a cookie doesn't tell you much.
2. **Look at the order of `<head>`**, not just its size. A late viewport tag silently doubles image downloads on mobile.
3. **Consent managers rewrite your page.** Test every optimization with consent *not yet given*.
4. **Throttle harder than the default** if your audience uses mid-range phones.
5. **Performance work isn't done until forms are tested.** A fast page that sends spam-flagged leads is worse
   than a slow one.

## How I worked

I used Claude Code as an agent. It handled the measurements, the code changes and the checks, and I set the
constraints and approved every step.

Before each change I made a backup. After each change I compared before/after Lighthouse runs, checked 404s, the
sitemap and the consent behaviour, and ran an intercepted test of the forms. A design change the agent proposed
(swapping an icon for an SVG) was rejected and reverted: the design was a constraint, not a variable.

---

*Andrea Sofia, Web Developer · [LinkedIn](https://www.linkedin.com/in/andrea-sofia-dev/) ·
[GitHub](https://github.com/andrea-sofia-dev)*
