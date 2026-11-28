# Shipping one branded app per client from one Expo codebase

Research for GitHub issue #5, "How do we ship one Branded App per Client from one Expo codebase?"
Researched 2026-11-28. Primary sources only. Anything not read directly from a first-party
source is marked **UNVERIFIED** and shouldn't be relied on yet.

## TL;DR

1. **Build:** use one `app.config.js` that reads a `CLIENT` (or `APP_VARIANT`) environment variable.
   For that client it returns the `name`, icon, colours, `ios.bundleIdentifier`, `android.package`
   and backend URL. Set the variable from an `eas.json` build profile's `env`, or from Expo
   environment variables. Expo's own docs name white-labelling as a use for this pattern. [E1][E3]
2. **Apple 4.2.6:** apps built from a commercial template or app generation service "will be rejected
   unless they are submitted directly by the provider of the app's content". The template provider
   "should not submit apps on behalf of their clients". **So each client publishes under its own Apple
   Developer account.** We can work inside that account as an invited team member. The only option
   Apple names for keeping everything under our account is a single "picker" app that aggregates all
   clients. [A1]
3. **Google Play:** Google's policy pages couldn't be reached from this session. The search index
   shows a Google rule that template or wizard apps are allowed only when published from the
   **client's own individually registered developer account**. **UNVERIFIED**: confirm on
   support.google.com before relying on it. [G1]
4. **OTA:** EAS Update ties every update to a channel, a runtime version and a platform. Billing is
   per Expo account, by monthly active users (MAU: unique installs that download an update) plus
   bandwidth. Two branded apps on one device count as two MAU. `EXPO_PUBLIC_*` values are inlined at
   publish time, so each client needs its own `eas update` run. [E4][E5][E6][E7]
5. **Frappe:** run `bench config dns_multitenant on`, then `bench setup add-domain <domain> --site <site>`,
   `bench setup nginx` and `bench setup lets-encrypt <site> [--custom-domain <domain>]`. Bench refuses
   to set up SSL unless DNS multitenancy is on. [F1][F2][F3]
6. **Indian competitors:** Graphy's help-centre titles show creators open **their own** Play Console
   account (Graphy also lists an Apple account as a prerequisite). Classplus marketing says it
   "publishes for you", but whose account it uses isn't publicly documented. Both are
   **UNVERIFIED**: the page bodies couldn't be read. [C1][C2]

---

## 1. Many apps from one Expo codebase (EAS)

### Dynamic app config

- Expo docs, *Switching configuration based on the environment*: "It's common to have some different
  configuration in development, staging, and production environments, or to swap out configuration
  entirely to **white label an app**. To accomplish this, you can use **app.config.js** along with
  environment variables." [E1]
- `app.config.js` exports a function that receives `{ config }`, the static `app.json`, and can
  override `name`, `ios.bundleIdentifier` and `android.package` from `process.env.APP_VARIANT`. [E2]
- Each variant needs a unique Android Application ID and iOS Bundle Identifier. "You can also switch
  out the app icon and other properties." Any `app.json` key can be swapped the same way. If a library
  registers the app ID with an outside service (Firebase/FCM, Google Maps), that service needs a
  separate configuration for each `android.package` and `ios.bundleIdentifier`. [E3]
- When the bundle ID changes, EAS CLI asks for a new Android keystore and a new iOS provisioning
  profile, so signing credentials are per client. [E2]

### eas.json build profiles

- Set the variable per profile with `env`, for example `"env": { "APP_VARIANT": "development" }`.
  Running `eas build --profile development` "will now set `APP_VARIANT`… when evaluating
  app.config.js both locally and on the Expo Build builder." [E3]
- A profile can use `"extends"` to inherit from another, for example a `base` profile, so per-client
  profiles stay short. [E8]
- `submit` profiles can take any name. Each holds `ios.ascAppId`, `ios.appleTeamId` and similar
  fields, which allows one submit profile per client's Apple team. [E9][E10]

### Expo environment variables

- Variables can be stored on expo.dev and read by builds, updates and hosting. `eas build` picks them
  up through `"environment": "production"` on the profile, and `eas update` and `eas deploy` take
  `--environment`. [E4]
- By default only three environments exist: `development`, `preview` and `production`. "Custom
  environment names are available on Enterprise and Production plans." Environments are therefore not
  a per-client switch on lower plans; use the profile `env` or a per-client config file in the repo
  instead. [E4][E11]

### Web build per client

- `npx expo export -p web` writes a static bundle to `dist/`. `web.output` is one of `single` (SPA),
  `static` or `server`. Any static host works, for example `netlify deploy --dir dist`. [E12]
- EAS Hosting: "**Each project can have exactly one custom domain**", and custom domains are "a
  premium feature" not on the free plan. [E13] On EAS Hosting, each client domain therefore means one
  EAS project per client. The alternative is to export per client (same `CLIENT` variable) and serve
  `dist/` from a host of our choosing, such as nginx in front of that client's Frappe site.

### Suggested shape (inference, not from a source)

`clients/<id>.json` holds name, slug, bundle ID, package, icon paths, colours, backend URL,
`easProjectId`, `appleTeamId` and `ascAppId`. `app.config.js` reads `process.env.CLIENT` and loads
that file. In `eas.json`, one build profile and one submit profile per client each extend a base
profile. Treat one EAS project per client as the default, because the hosting domain limit [E13] and
per-project update URLs (`updates.url` = `https://u.expo.dev/<projectId>`) [E14] both point that way.

## 2. Apple App Store Review Guidelines (current wording)

Source: developer.apple.com/app-store/review/guidelines, page marked "Last Updated: October 16, 2026",
fetched 2026-11-28. [A1]

> **4.2.6** Apps created from a commercialized template or app generation service will be rejected
> unless they are submitted directly by the provider of the app's content. These services should not
> submit apps on behalf of their clients and should offer tools that let their clients create
> customized, innovative apps that provide unique customer experiences. Another acceptable option for
> template providers is to create a single binary to host all client content in an aggregated or
> "picker" model, for example as a restaurant finder app with separate customized entries or pages for
> each client restaurant, or as an event app with separate entries for each client event.

> **4.3 Spam (a)** Don't create multiple Bundle IDs of the same app (for example, submitting a separate
> map app for every city in the world instead of a single worldwide map that allows users to search
> any city). This practice results in unnecessary apps, which makes it hard for users to find the apps
> they want. If your app has different versions for specific locations, sports teams, universities,
> etc., consider submitting a single app and providing the variations using in-app purchase.

> **4.3 (b)** Don't submit apps that are indistinguishable from what's already widely available. […]

> **5.2.1** […] Apps should be submitted by the person or legal entity that owns or has licensed the
> intellectual property and other relevant rights.

**Answer:** yes. Under 4.2.6, each client (the "provider of the app's content") must submit its own
app, which in practice means the client's own Apple Developer Program account. If several client
apps are submitted under *our* account, they risk rejection under 4.2.6, and their separate bundle
IDs also run into 4.3(a). Our compliant options are:

- **(a)** The client enrols (as an organization) and invites our team into its account. The Account
  Holder "initially add[s] Admins to the team". Access to Certificates, Identifiers & Profiles is a
  separate permission granted in Users and Access. [A2] We then build and submit with EAS using that
  client's `appleTeamId` and App Store Connect app.
- **(b)** A single "picker" app under our own account, listing every client inside it. The guideline
  permits this, but it isn't a branded app per client.

Even under the client's account, the app must not look like a copy of other apps (4.3(b)), so give
each client real differentiation (content and branding).

## 3. Google Play equivalent

**UNVERIFIED.** support.google.com, play.google.com and android-developers.googleblog.com are all
blocked by this session's network policy.

- Google's Spam policy lives at support.google.com/googleplay/android-developer/answer/9899034 (Policy
  Center: play.google.com/about/…/spam/). [G1]
- Search results attribute this wording to Google's July 2018 policy update: apps "created by an
  automated tool, wizard service, or based on templates and submitted to Google Play by the operator
  of that service on behalf of other persons" are not allowed. Such apps are permitted only if
  "published by an individually registered developer account belonging to the user of the automated
  tool". This was seen only in secondary coverage (TechCrunch, Android Authority, 2018). The current
  page also covers "Repetitive Content", meaning multiple apps with highly similar content and user
  experience. [G1]
- **Action:** read answer/9899034 directly and quote it here before deciding. If the 2018 wording
  still stands, Play matches Apple: each client needs its own Play Console account. That fits the
  Graphy evidence in §6.

## 4. OTA updates (EAS Update) across many apps

- **Model:** a build embeds a **channel** (set in the `eas.json` build profile), a **runtime version**
  and a **platform**. `eas update` uploads to a **branch**. An update reaches a build only when the
  platform and runtime version match exactly and the build's channel is linked to that branch. A
  channel links by default to the branch of the same name, and `eas channel:edit production --branch
  version-2.0` repoints it. [E5]
- **Runtime version:** `eas update:configure` sets `"runtimeVersion": {"policy": "appVersion"}` by
  default. [E15]
- **Per-client publishing:** `eas update --environment X` inlines `EXPO_PUBLIC_*` values from that
  environment at publish time. [E16] The backend URL and brand values are therefore baked into each
  update bundle, so each client needs its own `eas update` run (one run per client project or
  channel). A loop in CI handles this. *Inference from [E16][E14].*
- **Cost:** usage is billed per Expo account on two metrics, **monthly active users (MAU)** and global
  edge bandwidth. "1 monthly active user equals 1 unique installation of an app that downloads at
  least 1 update during your billing cycle." "A single device that has two apps owned by a single
  Expo account, both of which use updates, is considered 2 monthly active users." [E6][E7]
  - Worked example from Expo's docs: the Starter plan ($19/mo) includes 3,000 MAU and 100 GiB.
    Overage is $0.005 per extra MAU, which also adds 40 MiB of bandwidth per user, plus $0.10/GiB of
    bandwidth. 20 updates of 5 MiB each to 10,000 users came to $95.31 of overage. The Production
    plan includes 50,000 MAU and 1 TiB, and the same scenario cost $0 there. [E7][E11]
  - MAU is summed across all client apps on our account. Keep update bundles small, because
    bandwidth grows with updates × size × installs. [E5][E7]
- **Builds:** each client app needs its own binaries, so build credits scale with the number of
  clients times the number of native releases. Ship JavaScript-only changes as updates. Expo suggests
  checking Fingerprint so CI rebuilds only when native code changed. [E7]

## 5. Frappe multi-site: a domain per client

From the source of `frappe/bench` (develop branch) and `frappe/frappe`. docs.frappe.io was blocked.

- **Site resolution:** Frappe picks the site from the `X-Frappe-Site-Name` header, or else from the
  request host (`frappe/app.py`: `request.headers.get("X-Frappe-Site-Name") or
  get_site_name(get_request_host(request))`). [F4] Name each site after its domain, or add the
  domain to the site.
- **Enable DNS multitenancy:** `bench config dns_multitenant on` writes `dns_multitenant: true` to
  `common_site_config.json`. [F1]
- **Custom domain:** `bench setup add-domain <domain> --site <site> [--ssl-certificate PATH
  --ssl-certificate-key PATH]` appends the domain to the site's `site_config.json` `domains` list.
  `remove-domain` and `sync-domains` also exist. [F2]
- **nginx:** `bench setup nginx` emits a `server_name` for each site. When `dns_multitenant` is on,
  it also emits one for every entry in `domains`. [F5]
- **SSL:** `bench setup lets-encrypt <site> [--custom-domain <domain>] [-n]` runs certbot, and the
  domain must already be added to the site. The code checks `if not dns_multitenant: print("You
  cannot setup SSL without DNS Multitenancy")`. It also installs a root cron job: `certbot renew -a
  nginx --post-hook "systemctl reload nginx"`. `bench setup wildcard-ssl <domain>` issues a wildcard
  certificate for a multi-tenant bench, useful for `*.ourlms.com` subdomains. [F2][F3]
- **DNS:** each client domain (or subdomain) needs an A or CNAME record pointing to the bench server
  before Let's Encrypt can validate it. *This is standard ACME behaviour, not stated in the bench
  source.*

Recommended flow: `bench new-site client.example.com` → client sets the DNS record → `bench setup
add-domain` (only if the site name differs from the domain) → `bench setup nginx` and reload →
`bench setup lets-encrypt`. Each app's `app.config.js` then points at `https://client.example.com`.

## 6. How Indian edtech white-label providers handle store listings

**UNVERIFIED.** help.graphy.com, trainings.graphy.com, classplusapp.com and apps.apple.com are all
blocked here. The points below come only from search-index titles and snippets of first-party pages.

- **Graphy:** its help centre has the articles "Android || How to create a developer account on the
  Google Play store?" and "How to Verify Your Google Play Console Developer Account for Your Graphy
  App?". Snippets of its onboarding page list a Google Play Console account ($25) and an Apple
  Developer account ($99/yr) as prerequisites. This suggests the creator owns the store accounts and
  Graphy builds and publishes into them. [C1]
- **Classplus:** marketing pages on classplusapp.com say the app is "published for you" and branded
  with the educator's name and logo. One page lists "Android app; iOS depending on plan". No page
  found says whose developer account is used. The main "Classplus" App Store listing (id1324522260)
  has the seller EPSILON EDUVENTURES PRIVATE LIMITED, a separate aggregated app. [C2]
- Neither claim is confirmed. Before relying on them, check the Play Store "developer" field on a
  known Classplus or Graphy client app.

## Open items

1. Allow support.google.com (or paste the Spam policy text) to confirm §3.
2. Allow help.graphy.com and classplusapp.com to confirm §6.
3. Check whether one EAS project can hold several bundle IDs with separate credentials. Not verified;
   the docs read here assume one project per app.

## Sources

Expo docs were read from the source in the `expo/expo` repo (`docs/pages/**`, `main` branch,
2026-11-28), because docs.expo.dev was blocked. Each docs.expo.dev page is that path without `.mdx`.

- [E1] docs.expo.dev/workflow/configuration/#switching-configuration-based-on-the-environment
- [E2] docs.expo.dev/tutorial/eas/multiple-app-variants/
- [E3] docs.expo.dev/build-reference/variants/
- [E4] docs.expo.dev/eas/environment-variables/
- [E5] docs.expo.dev/eas-update/how-it-works/
- [E6] docs.expo.dev/eas-update/introduction/ (FAQ: how MAU are counted)
- [E7] docs.expo.dev/billing/usage-based-pricing/
- [E8] docs.expo.dev/build/eas-json/
- [E9] docs.expo.dev/submit/eas-json/
- [E10] docs.expo.dev/eas/json/
- [E11] docs.expo.dev/billing/plans/ and docs.expo.dev/eas/environment-variables/manage/#custom-environments
- [E12] docs.expo.dev/guides/publishing-websites/
- [E13] docs.expo.dev/eas/hosting/custom-domain/
- [E14] docs.expo.dev/eas-update/getting-started/ and docs.expo.dev/eas-update/expo-dev-client/
- [E15] docs.expo.dev/eas-update/deployment/
- [E16] docs.expo.dev/eas/environment-variables/usage/#using-environment-variables-with-expo-update
- [A1] https://developer.apple.com/app-store/review/guidelines/ (guidelines 4.2.6, 4.3, 5.2.1; Last Updated October 16, 2026)
- [A2] https://developer.apple.com/help/account/access/roles/
- [G1] https://support.google.com/googleplay/android-developer/answer/9899034 (not readable here); 2018 wording via techcrunch.com/2018/07/27/google-follows-in-apples-footsteps-by-cleaning-up-its-play-store (secondary)
- [F1] github.com/frappe/bench, `bench/commands/config.py` (`dns_multitenant`)
- [F2] github.com/frappe/bench, `bench/commands/setup.py` (`add-domain`, `lets-encrypt`, `wildcard-ssl`) and `bench/config/site_config.py`
- [F3] github.com/frappe/bench, `bench/config/lets_encrypt.py`
- [F4] github.com/frappe/frappe, `frappe/app.py`
- [F5] github.com/frappe/bench, `bench/config/nginx.py`
- [C1] help.graphy.com/support/solutions/articles/1060000131352, us.help.graphy.com/support/solutions/articles/1060000123224, trainings.graphy.com/s/pages/onboarding-process (search-index only)
- [C2] classplusapp.com/growth/own-branded-app-for-coaching-institute/, apps.apple.com/app/id1324522260 (search-index only)
