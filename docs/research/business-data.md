# Business data check (read-only), 2026-11-04

These are read-only checks: only `asc` list/status calls were made. No secrets or customer data are recorded here.

## App Store Connect (`asc`, already signed in with an API key)

There are 22 apps in total. Bundle IDs use the `com.lilgroup.*` prefix, except QuickShift (`com.oscarapp.quickshift`), LilOS (`com.nuncio.lilos.mobile`) and QRit Staff (`studio.lilgroup.qrit.staff`).

**Live (READY_FOR_DISTRIBUTION)**

| App | Version | What it sells | Reviews |
|---|---|---|---|
| FillLight: Selfie Light Camera | 1.1.5, released 2026-10-28 with phased release complete | Subscription group "FillLight Pro", plus in-app purchases: Lifetime Step 1/2/3 and Pro Unlock | 1 review, 5.0★ |
| Footy President: Club Manager | 1.0 | 19 consumable/unlock in-app purchases (Go Pro, Everything Bundle, etc.) | 0 |
| QuickShift | 0.3.9 | No in-app purchases or subscriptions on Apple, so it is probably monetised through Stripe or free | 0 |
| StyleMaxx: AI Color Analysis | 1.1 | In-app purchases: Full Report First / Repeat 25 / Repeat 50 | 0 |
| Wrap: Along Your Journey | 1.0 | In-app purchase: Wrap Lifetime | 0 |

**Rejected or blocked**
- Rejected with unresolved issues: Oathwise, GymJoy, ManeBooth, GlowMax AI.
- Developer-rejected: Selah 1.0.7. Build 1.1 (18) is uploaded but not yet submitted.

**Preparing for submission**
- Has a build: LilOS (also in external TestFlight), QRit Staff, BeFocus, Repwise, GlowDNA, Kiro.
- No build uploaded yet: BeginBetter, Lunoria, FitPace, RecordCute, Dim (macOS), TwinTake.

**Not on App Store Connect**
- Heardback and QuickBook do not appear. They are probably web/Stripe products.

### What the factory can read automatically through `asc`
- App list, `asc status` (health, blockers, review state, build/TestFlight state, suggested next command).
- In-app purchase and subscription catalog.
- Customer reviews and ratings: `asc reviews list`.
- Release and review blockers: `asc review doctor`.

### Not retrieved or missing
- **Sales, downloads and revenue.** These need `asc analytics sales --vendor <VENDOR_NUMBER>` or `asc finance` / `asc insights`. I couldn't find the vendor number in the environment, so I didn't run them.
- **Analytics reports** (impressions, downloads) need an `asc analytics request`, which is a write (it creates a report request), so I skipped it.
- Subscription price points weren't pulled. They are available through `asc subscriptions` / `asc pricing` (read).

## Stripe: blocked

- Every `stripe` command, including `stripe config --list` and `stripe products list`, fails with: `chmod ~/.config/stripe/config.toml: operation not permitted`. The CLI tries to change the permissions on its config file, and the sandbox only allows writes inside the workspace.
- I didn't retry. There is no data on accounts, products/prices, subscriptions or balance. Heardback, QuickBook and QuickShift are **unmapped**.
- To fix this, do one of the following:
  - run the read commands outside the sandbox (or let the user run them);
  - set `STRIPE_API_KEY` to a **restricted read-only** key and use `curl https://api.stripe.com/v1/...`;
  - give the CLI a config path inside the workspace.

## Per product (what we know)

| Product | Platform | Pricing | Revenue / subs / downloads | Readable automatically | Missing |
|---|---|---|---|---|---|
| FillLight | iOS App Store | Subscription plus lifetime tiers (prices not pulled) | Unknown; 1 review | Status, in-app purchases, reviews | Sales report (vendor #) |
| QuickShift | iOS app (live 0.3.9) + Stripe? | None on Apple | Unknown | Status | Stripe data |
| Heardback | Not on App Store Connect; Stripe? | ? | ? | — | Stripe access |
| QuickBook | Not on App Store Connect; Stripe? | ? | ? | — | Stripe access |
| Footy President, StyleMaxx, Wrap | iOS | In-app purchases as listed above | Unknown; 0 reviews | Status, in-app purchases, reviews | Sales report |
