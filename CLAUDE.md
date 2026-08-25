# Jon Blund

A baby call that rings a real phone. Device A lies by the crib and listens in
the browser; when it hears crying it places an actual phone call to phone B
over Twilio. No app, no hardware.

**Call it a baby call, never a baby monitor** — in copy and in comments. A is
any device with internet and a microphone (an iPad is tested and good at it);
B must be a phone with coverage, because that is the leg Twilio dials.

**Petter is not a developer.** Give click paths, not assumptions. Norwegian in
conversation, English in the product. Never push without being asked — he says
"push" when he wants it out.

## The three moving parts

| Where | What |
|---|---|
| `jonblund` (this repo) | `index.html` marketing site · `babycall.html` the app. GitHub Pages → **www.jonblund.site** |
| `jonblund-voice` (separate private repo) | Vercel serverless: `/api/token`, `/api/voice`, `/api/vcard`, `/api/incoming` |
| Twilio | number **+1 240 232 6663** (US, voice only), TwiML app `AP8b24…`, geo permissions Norway only, auto-recharge OFF |

Local: `/Users/petter/Desktop/Babycall/{jonblund,jonblund-voice}`.
`jonblund-voice/NOTES.local.md` holds the env vars — gitignored, never commit it.

## How a call happens

1. Browser detects crying → `JONBLUND_TELEPHONY.call()`
2. Twilio Voice SDK opens a WebRTC leg, hits the TwiML app
3. TwiML app → `/api/voice`, which returns `<Dial callerId=…>` for the number
4. Twilio bridges the two legs. B hangs up → `disconnect` → 20s cooldown → listening

## Decisions worth not relitigating

- **Detection**: loudness over an adaptive noise floor AND tonality in 250–2500 Hz,
  sustained 2s of *actual sound* (not wall clock — gap tolerance must not pad it).
  One fixed threshold; the Sensitivity and "call after N" controls were removed
  because they asked a parent to guess at decibels.
- **`answerOnBridge: true`** so billing starts when B answers, not when it rings.
- **All Twilio SDK sounds off, and the remote stream muted** — phone A sits beside
  a sleeping baby and must never make noise. This is not optional.
- **The contact card** (`/api/vcard`) is what lets the call through Do Not Disturb.
  It is a feature, not a nicety.
- **Pricing**: $2.99/month + $0.20/call, first 2 minutes of a call included
  then $0.39/min, first month free, spending *stops* at $35 — the service
  pauses, it is not a discounted cap. Lives in one `PRICE` constant in
  `babycall.html`, and separately in the marketing copy.
- **The year figure** ($181.88: the fee for 12 months plus two cries a night
  for 365 nights) is modelled, not measured. It has a deliberate seam —
  `monthlyBill()` in both files — to be replaced by the real average monthly
  bill once there are enough customers to average over.
- **Favicon** is Petter's own `jb` in Fraunces, `j` in ink and `b` in accent,
  on the cloud gradient. `favicon.png` is his file as delivered, kept as the
  source. The served sizes are cropped in from it: his artwork puts the mark
  at only 43% of the canvas width, which turns to mush at 16px, so the empty
  margin is trimmed to a 514px square (mark at ~78% of the height) before
  resizing to `favicon-16.png`, `favicon-32.png` and `apple-touch-icon.png`.
  Redo that crop step if the source is ever replaced; do not resize his file
  directly. No SVG: Fraunces' hairlines cannot be traced faithfully by hand.

## Gotchas that cost real time

- **Measure before tuning thresholds.** Three rounds were burned guessing at
  detection numbers. Use `OfflineAudioContext` to score synthetic signals, or
  read the on-screen diagnostics (tap the status label).
- **The Twilio SDK CDN at `sdk.twilio.com/js/voice/releases/…` 403s.** Use
  jsDelivr: `@twilio/voice-sdk@2.18.3`.
- **The adapter must return its hang-up handle immediately**, before awaiting
  `accept` — otherwise "Hang up" is wired to nothing while it rings.
- **Vercel env vars need a redeploy** to take effect.
- **Flex `min-height:auto`** silently crushed modal text to zero height. Definite
  heights beat negotiated ones.
- Contact Picker API is Chrome-on-Android only. Build the button at runtime so
  iPhone never sees a dead control.
- **The onboarding steps do not scroll** (`.hiw-view` is `overflow:hidden` with
  a definite height). Step 1's copy has ~18px of slack on a 375×667 phone, so
  measure the paragraph there before lengthening it, or the last line vanishes.

## Not safe for real customers yet

- `JONBLUND_KEY` sits in public client code and is shared by everyone
- `JONBLUND_ALLOWED_NUMBERS` is `*` — any Norwegian or US number is dialable
- Vercel Hobby forbids commercial use; Twilio's outbound calls-per-second
  default is far below what real traffic needs

Both of the first two are solved by the payment work: per-user auth (Supabase),
a per-user number allowlist, and the spending gate in `/api/voice`.

## Next

Supabase auth + Stripe Checkout with a 7-day trial, added as a final onboarding
step. Full recipes:

- Twilio setup — https://claude.ai/code/artifact/e16b97ad-b901-4ea1-8557-f218930cb370
- Login and payment — https://claude.ai/code/artifact/6025f066-41d0-4121-b766-f0f19fc99940
