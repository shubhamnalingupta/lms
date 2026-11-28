# Which video host should v1 stream course videos from?

Research for issue #12. Researched 2026-11-28.

**Recommendation: Bunny Stream.** Start v1 with token-authenticated (signed, expiring) HLS played through `expo-video` on mobile and Bunny's embed on web. Add MediaCage Enterprise DRM (Widevine + FairPlay) for each client that pays for it. Reasoning is in [Recommendation](#recommendation).

## How this was researched (read first)

The sandbox's network blocked direct fetches of every vendor domain: bunny.net, developers.cloudflare.com, mux.com, vdocipher.com. Every fact below comes from web-search results limited to the vendor's own domain (first-party pages), so I read search extracts, not the live pages myself. Each claim cites the first-party URL the extract came from. Before you sign anything, **re-check every price on the live page.** Labels used:

- **(verified-extract)**: a figure taken from a first-party page's search extract.
- **(unverified)**: no first-party confirmation found, or the sources conflict.
- **(assumption)**: my own input to the worked example.

## Summary table

| | Bunny Stream | Cloudflare Stream | VdoCipher | Mux | Unlisted YouTube |
|---|---|---|---|---|---|
| Signed / expiring URLs | Yes: SHA256 token with expiry [B3] | Yes: signed tokens, max 24 h, geo/IP rules [C3] | Yes: single-use OTP per playback, made on your server [V5] | Yes: signed JWT playback tokens [M4] | No. Anyone with the link can watch |
| Widevine / FairPlay DRM | Yes, MediaCage Enterprise [B4] | **No** for playback (community/staff answers; no DRM in docs) [C4] (unverified that this is still current) | Yes, Widevine + FairPlay. FairPlay needs your own Apple application [V6] | Yes, add-on [M4] | No |
| Screen-capture blocking | Enterprise only. Not on Widevine L3 devices [B4] | No | Yes with DRM in its SDKs. iOS needs FairPlay and the Value plan or higher. The vendor itself says recording is "still possible", so read this as mitigation [V6][V7] | Only what hardware DRM gives you (unverified) | No |
| Dynamic (per-user) watermark | Not found in docs (unverified) | Static image watermark only [C5] | Yes: email/ID/IP, moving. In the RN SDK since 1.20.6 [V7][V8] | Static image watermark only [M6] | No |
| expo-video playback | **Official Bunny "Expo Video guide"**, including Enterprise DRM through the `drm` source option [B5] | Plain HLS works. No DRM to set up | **Not expo-video.** Needs its own native module `vdocipher-rn-bridge`. No Expo config plugin found [V8] (unverified on Expo) | Mux RN guide uses expo-video with signed URLs. No first-party DRM sample for RN [M5] | Only through a WebView/iframe player |
| Web | Iframe embed (`iframe.mediadelivery.net`), FairPlay HTML5 guide [B4] | Iframe/HLS | Its JS player | Mux Player web | Iframe |
| India delivery | PoPs in Mumbai, New Delhi, Chennai, Bangalore [B6] | Data centres in Mumbai, Chennai, New Delhi [C6] | India-based company. No first-party CDN detail found (unverified) | Multi-CDN, nearest PoP. No India detail found (unverified) [M6] | Google's network |
| Per-client separation | A video library per client (own API key), sub-users, "Bunny for Platforms" / Partner Portal API for resellers [B7] | One Cloudflare account per client (unverified for any reseller program) | Several accounts (one API key each) on Express or higher, or folders/tags in one account [V9] | Organizations (billed separately) and environments (isolated assets/keys) [M6] | One channel per client |
| Frappe LMS embed | **Native**: paste the `iframe.mediadelivery.net` URL [F1] | **Native**: paste the `/watch` URL [F1] | **Not native**. Needs a per-play OTP, so it can't be a static URL | Not in Frappe's list. Static HLS only works without signed URLs [F1] | **Native** [F1] |

## Pricing inputs (first-party extracts)

- **Bunny Stream** [B1]: storage $0.01/GB for the 1st and 2nd region, $0.005/GB for the 3rd region on. Delivery on the Standard network, Asia & Oceania: **$0.030/GB**. Delivery on the Volume network, one worldwide rate: **$0.005/GB** for the first 500 TB. Standard encoding is free. Minimum charge is $1/month (third-party review, unverified). MediaCage Enterprise DRM [B2] costs **$99/month per video library**, plus $0.005 per licence for the first 20k each month and $0.004 per licence from 20,001 to 100k. One playback issues ≥2 licences (video + audio). (verified-extract)
- **Cloudflare Stream** [C1]: **$5 per 1,000 minutes stored** and **$1 per 1,000 minutes delivered**. Same price at any bitrate. Encoding is free. (verified-extract)
- **Mux** [M1][M2]: the first **100,000 delivered minutes each month are free**. Plus-quality delivery costs $0.0008/min at ≤720p and $0.001/min at 1080p, for the first 500k-minute tier. Storage at ≤720p costs $0.0024 per minute per month. Encoding at ≤720p costs $0.025/min for the first 5k minutes. Basic-quality encoding is free. DRM [M3] costs **$100/month plus $0.003 per licence**. (verified-extract; rates for later tiers not found)
- **VdoCipher** [V1]: prepaid plans valid for one year. Starter $149 (1 TB). Value $399 (2.5 TB, 200 GB storage). Express $699 (5 TB, 400 GB). Pro $1,599 (15 TB, 1 TB). Premium $4,999 (50 TB, 2 TB). Enterprise is custom. Bandwidth overage runs from $0.11/GB on Starter to $0.07/GB on Premium. Storage overage is $0.75/GB. I found USD prices only. (verified-extract)
- **YouTube**: free.

## Worked example: 1,000 learners × 20 h/month

Assumptions (all **(assumption)**):
- 20,000 viewer-hours = **1,200,000 delivered minutes** a month.
- Bunny and VdoCipher bill per GB, so bitrate matters. I assume an average 720p adaptive stream of about 1.5 Mbit/s, which is about **0.675 GB/hour**, so about **13.5 TB/month**. At 1080p (about 3 Mbit/s), double the GB-based costs.
- Library: 200 hours of content = 12,000 minutes stored, about 300 GB with all renditions.
- About 40,000 playbacks a month (30-minute sessions), so 40k–80k DRM licences.

| Host | Delivery | Storage | DRM | **≈ Monthly total** |
|---|---|---|---|---|
| Bunny, Standard network (Asia rate) | 13,500 GB × $0.03 = $405 | 300 GB × $0.01 = $3 | +$279–$439 if Enterprise is on ($99 + $180–$340 licences) | **~$408** (no DRM) / **~$690–$850** (DRM) |
| Bunny, Volume network | 13,500 GB × $0.005 = $68 | $3 | same | **~$71** / ~$350–$510. India coverage on the Volume network is unverified |
| Cloudflare Stream | 1,200 × $1 = $1,200 | 12 × $5 = $60 | not available | **~$1,260** |
| Mux (Plus, ≤720p) | (1.2M − 100k free) × $0.0008 ≤ $880 | 12,000 × $0.0024 = $29 | +$220–$340 | **≤ ~$910** / **≤ ~$1,250**. Plus a one-off encode of ≤$300 (unless Basic) |
| VdoCipher (Premium list price) | 162 TB/yr: $4,999 + 112 TB × $0.07 = $12,839/yr | included up to 2 TB | included | **~$1,070** (list price. An Enterprise quote is likely lower, unverified) |
| Unlisted YouTube | $0 | $0 | none | **$0** |

The Mux total is an upper bound because I don't know how the free minutes count against the first tier, or the rates beyond 500k.

## Per-option notes

- **Unlisted YouTube: rule out.** The link is the only protection. YouTube's API policies forbid overlays on the player, changes to the player, and hiding YouTube branding [Y1]. That conflicts with a white-label app and with any watermark overlay. There's also no HLS URL you can legitimately hand to `expo-video`.
- **Cloudflare Stream.** Solid signed URLs, India data centres, and Frappe embeds it natively. But it has no DRM, its watermark is a static image, and at this usage it costs about 3× Bunny Standard.
- **Mux.** Best developer experience, with orgs/environments that fit resale well. But it's per-minute expensive at this scale, its RN DRM path isn't documented ("expo-video doesn't have a first-class story" for DRM, Mux blog [M5]), and Frappe doesn't embed it natively.
- **VdoCipher.** Strongest anti-piracy set: DRM, dynamic watermark, and capture blocking in its SDK. But it needs a native bridge rather than `expo-video` (Expo prebuild compatibility is unverified), it needs a backend-issued OTP per play so Frappe can't embed it natively, its price is high at this volume, and you prepay a year up front.
- **Bunny Stream.** Cheapest India delivery, PoPs in four Indian cities, Frappe embeds it natively, and there's a first-party `expo-video` guide that includes Widevine/FairPlay. Separate libraries per client give isolation. Gaps: no dynamic watermark found, Basic DRM only works in Bunny's own embed (whether it works with expo-video conflicts between pages, so unverified) [B5], and Enterprise DRM is $99/month *per library*, so per client. It also needs your own Apple FairPlay certificate [B4] and activation through sales.

## Recommendation

**Bunny Stream for v1.**

1. **Fits the stack.** Frappe accepts Bunny URLs natively [F1], and Bunny documents `expo-video` playback, DRM included [B5]. No custom native module is needed in the Expo app.
2. **Cheapest for an India-heavy audience.** About $408/month on the Standard network for the worked example, against about $1,260 for Cloudflare and up to about $910 for Mux. Pricing is pay-as-you-go, with no annual prepay.
3. **Protection can grow later.** v1 ships signed, expiring token URLs plus referrer lock. Enterprise DRM (Widevine/FairPlay, screen-grab protection) is a switch per library, so it can be a paid tier per client ($99/month plus licences) rather than a platform-wide cost.
4. **Resale fits.** One library per client, sub-users, and a Partner Portal / Bunny for Platforms path if resale grows [B7].

**Close the gaps in the app, not at the host.** Add a per-user text watermark overlay (email/ID) in the RN player screen and on web. On Android, use `FLAG_SECURE` through `expo-screen-capture`; whether it protects on iOS is unverified. **Switch to VdoCipher** only if a client contractually needs vendor-enforced dynamic watermarks and capture blocking.

**Check before committing:** the live Bunny pricing page, especially whether Stream can use the Volume network and its India coverage; whether expo-video DRM works on web (the expo-video docs list DRM types for Android and iOS only [E1]); and a FairPlay test on a real device.

## Sources (all first-party unless marked)

- [B1] https://bunny.net/docs/stream/pricing · https://bunny.net/pricing/stream/
- [B2] https://docs.bunny.net/stream/pricing (MediaCage Enterprise fees)
- [B3] https://docs.bunny.net/stream/token-authentication
- [B4] https://docs.bunny.net/stream/drm · https://docs.bunny.net/stream/mediacage-enterprise · https://docs.bunny.net/stream/players/fairplay
- [B5] https://bunny.net/docs/stream/expo-video · https://bunny.net/docs/stream/mobile-sdk · https://bunny.net/docs/stream/mediacage-basic
- [B6] https://bunny.net/network/ · https://bunny.net/cdn/content-delivery-asia/
- [B7] https://bunny.net/bunny-for-platforms/ · https://bunny.net/docs/api-reference · https://bunny.net/blog/introducing-sub-accounts-invite-your-team-to-bunnycdn/
- [C1] https://www.cloudflare.com/products/stream/ · https://www.cloudflare.com/plans/
- [C3] https://developers.cloudflare.com/stream/viewing-videos/securing-your-stream/
- [C4] https://community.cloudflare.com/t/cloudflare-stream-drm-support/835626 (community forum, not docs)
- [C5] https://developers.cloudflare.com/stream/edit-videos/applying-watermarks/
- [C6] https://www.cloudflare.com/network/
- [M1] https://www.mux.com/docs/pricing/overview · https://www.mux.com/docs/pricing.txt
- [M2] https://www.mux.com/pricing (older marketing page)
- [M3] https://www.mux.com/docs/guides/protect-videos-with-drm
- [M4] same as M3; https://www.mux.com/articles/securing-video-playback-with-signed-urls
- [M5] https://www.mux.com/docs/frameworks/react-native-video-playback · https://www.mux.com/blog/react-native-needs-a-new-video-player
- [M6] https://www.mux.com/docs/core/mux-fundamentals · https://www.mux.com/docs/guides/add-watermarks-to-your-videos
- [V1] https://www.vdocipher.com/site/pricing/
- [V5] https://www.vdocipher.com/docs/server/ · https://www.vdocipher.com/page/api/
- [V6] https://www.vdocipher.com/page/drm/ · https://www.vdocipher.com/blog/screen-capture-block-video/
- [V7] https://www.vdocipher.com/dynamic-watermarking/ · https://www.vdocipher.com/page/features/
- [V8] https://www.vdocipher.com/docs/mobile/react-native/getting-started/ · https://www.vdocipher.com/docs/mobile/react-native/changelog/ · https://www.npmjs.com/package/vdocipher-rn-bridge
- [V9] https://www.vdocipher.com/blog/course-website-builder/
- [Y1] https://developers.google.com/youtube/terms/developer-policies · https://developers.google.com/youtube/terms/required-minimum-functionality
- [F1] https://docs.frappe.io/learning/course-creation/adding-a-lesson/add-video-content
- [E1] https://docs.expo.dev/versions/latest/sdk/video/
