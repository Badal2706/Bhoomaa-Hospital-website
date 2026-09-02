# BHOOMAA Medicare Hospital — Launch Checklist

Planned domain: **bhoomaahospital.com** (GoDaddy) — not yet purchased at time of writing.

---

## Upload to the web root

```
index.html
robots.txt
sitemap.xml
site.webmanifest
assets/            (20 files)
```

**Do not upload:**
- `_DELETE-THESE/` — superseded files, safe to delete entirely
- `_original-photos-do-not-upload/` — full-resolution source photos, keep locally for future edits
- `LAUNCH-CHECKLIST.md` and `DEPLOY-DIGITALOCEAN.md` — these notes

Server setup steps are in **`DEPLOY-DIGITALOCEAN.md`** (steps 1–8 there cover items 1–3, 6, 7 and 11 below).

---

## Launch steps

| # | Step | Notes |
|---|------|-------|
| 1 | Purchase the domain | |
| 2 | Deploy over HTTPS | GoDaddy SSL; redirect `http://` → `https://` |
| 3 | Pick one hostname | `bhoomaahospital.com` **or** `www.` — 301-redirect the other. Two live versions split ranking signals |
| 4 | Replace placeholder URLs | If the final domain differs, find-and-replace `bhoomaahospital.com` in `index.html`, `robots.txt`, `sitemap.xml` |
| 5 | Verify canonical | Currently `https://bhoomaahospital.com/` — must match step 3 exactly |
| 6 | Verify robots.txt | Load `/robots.txt`; confirm it allows crawling and lists the sitemap |
| 7 | Verify sitemap | Load `/sitemap.xml`; update `lastmod` to the real launch date |
| 8 | Test structured data | <https://search.google.com/test/rich-results> — expect Organization, Hospital, Physician ×2 to parse |
| 9 | Test mobile usability | Real phone, not just a resized browser window |
| 10 | Test page speed | <https://pagespeed.web.dev/> — check the mobile score |
| 11 | Enable compression | gzip/Brotli takes the 145 KB HTML to roughly 28 KB |
| 12 | Create Search Console property | Verify via GoDaddy DNS TXT record |
| 13 | Submit sitemap | In Search Console, after the site is publicly reachable |
| 14 | Bing Webmaster Tools | Import from Search Console. Bing powers ChatGPT, Copilot and Alexa |
| 15 | Verify Google Business Profile | See below — this matters more than the website for map results |
| 16 | Verify NAP consistency | See below |
| 17 | Test `tel:` and WhatsApp links | On a real phone; confirm the appointment form opens WhatsApp with details filled in |
| 18 | Confirm no stray `noindex` | View source, check the robots meta tag |
| 19 | Confirm Googlebot can crawl | Search Console → URL Inspection → Test Live URL |

Do not submit anything to Google before the site is publicly accessible.

---

## Google Business Profile

More important than the website for appearing in map results.

- **Primary category: Hospital.** This is the single strongest local-pack factor, and a wrong primary category is the strongest negative factor.
- Secondary categories: General Practitioner, Emergency Care Physician, Diabetologist
- Add the website URL and the correct 9–2 / 5–7 hours, with emergency marked 24/7
- Upload interior, exterior, reception and equipment photos
- Name, address and phone must match the website **character for character**

---

## NAP — use exactly this, everywhere

```
BHOOMAA Medicare Hospital & I.C.U.
Bhoomaa Sankul, Behind Kotak Mahindra Bank
Ambika Society, Palanpur – 385001
Banaskantha, Gujarat, India
02742-255540
```

Same wording on the website, Google Business Profile, Facebook, and every directory. Inconsistent NAP is one of the most common reasons a local business underperforms in map results.

---

## Free citations worth claiming

`palanpurhospital.in` currently ranks for "hospital with ICU in Palanpur" and lists 20 hospitals — BHOOMAA is not among them, though several neighbours in the same lane are. A free listing there is likely to outperform any on-page change.

Then: Practo, JustDial, Sulekha, Lybrate, Bing Places. Use the exact NAP block above.

---

## Highest-value next step

Dedicated pages per core service — Cardiac Care, Diabetes Care, ICU & Critical Care, Emergency — are the strongest local-organic factor there is. The current single-page site is the main structural limit on how far it can rank.

This only works with genuinely different content on each page. Pages that merely swap a keyword are doorway pages and are penalised.

---

## Decisions taken, and why

**No `aggregateRating` or `review` in the structured data.** The 4.8 and the reviews display on the page, but Google's review-snippet documentation states that `LocalBusiness` and `Organization` reviews are supported "only for sites that capture reviews about other local businesses." A business marking up reviews about itself is self-serving and risks a manual action. Displaying is fine; marking up is not.

**FAQ schema kept, but expect no rich result.** Google deprecated FAQ rich results on 7 May 2026, extending it to all sites including health and government. The markup remains valid and harmless, and still helps AI assistants answer questions about the hospital correctly.

**No `meta keywords`.** Google has ignored it since 2009; it only advertises your targets to competitors.

**No `priority` or `changefreq` in the sitemap.** Google ignores both.

**Images ship as WebP with a JPEG/PNG fallback**, via `<picture>`. Modern browsers take the WebP, older ones the original. Nothing changes visually. Images were re-encoded from the untouched originals rather than from the previous web copies, so there is no double-compression loss.

**The appointment form opens WhatsApp.** No server, no monthly cost, and no patient health information stored on a web server — which avoids a data-protection problem a database-backed form would create.

---

## Still outstanding

- The consulting-room photo has a garbled desk nameplate from AI processing. Crop it out or swap in the unprocessed original before launch.
- Confirm the doctor portrait is genuinely Dr. Panchasara; it is captioned with his name.
