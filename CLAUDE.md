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
- **Check-in calls** ring on a timer even when the room is quiet, so a parent
  gets proof the line still works. Off by default, then 15/30/45/60 minutes,
  in `checkEvery` (persisted) with `maybeCheckIn()` as the decision. A saved
  value that is no longer offered falls back to Off, checked against `20`.
  - It runs off the **500 ms heartbeat, not the rAF loop**: rAF stops the
    moment the page is hidden, and this is the feature that must not be the
    first to die. It needs the clock, not an audio frame.
  - It never dials while `burstStart` is set, so a check-in cannot cut off a
    cry that is on its way in. It also re-arms *before* dialling, so a failure
    or a missing recipient cannot turn into a loop.
  - Any finished call re-arms the clock. You just had a call, so the line is
    proven and the timer starts over.
  - **The cost is the reason this is off by default.** Every check-in is a
    call. Over an eight-hour night: 60 min = 8 calls = $1.60, 15 min = 32 calls
    = $6.40. The $35 ceiling stops the service after 5 nights at a quarter of
    an hour and 20 nights at the hour, so every setting reaches it inside a
    month.
  - The arithmetic **used to be printed under the picker and was removed on
    request** — Petter did not want red fine print scolding the user there.
    Leave it out. Cost belongs in the pricing sheet.
  - **Open decision**: `answerOnBridge: true` means Twilio only bills once B
    answers, so a check-in the parent lets ring costs *Twilio* nothing. Whether
    Jon Blund still charges its own $0.20 for a ring that was never answered is
    unsettled — the UI currently says "let one ring out and it costs nothing".
    Settle it when billing goes in, in `/api/voice` and this copy together.
  - On the receiving phone a check-in is indistinguishable from a real cry;
    both come from the one Twilio number. Verifying means answering and
    hearing the quiet. A second number would separate them, but that is
    server work nobody has asked for.
- **`PRICE`, `usd` and `usdWhole` sit at the top of the app script**, not in the
  pricing section, because the check-in panel prices itself during start-up.
- **Nap clock.** Birth date in onboarding step 3, awake windows by age in
  `WAKE_WINDOWS`, and a message when the window is nearly out.
  - The session is the microphone: on means she went down, off means she woke.
    The start time is an *assumption* and the slider moves it ±90 min, because
    the assumption is usually a few minutes wrong. The wake time is not
    editable — the message hangs off it, and that was the spec.
  - **Sex is deliberately not in the table.** Infant awake windows track age,
    not sex, so a sex field would promise precision that does not exist.
  - Ranges come from Cleveland Clinic, Huckleberry and Taking Cara Babies,
    which agree. We use the midpoint and write "around", never a precise
    minute. Verified bucket by bucket against the sources.
  - The message goes out `NAP_LEAD_MS` (15 min) before the window ends, once
    per nap, guarded by `sleep.msgDone`. Off by default; turning it on costs
    `PRICE.perMessage` and therefore gets a modal.
  - **It needs SMS that does not exist yet.** `api/sms.js` is written but inert
    until `TWILIO_MESSAGING_SERVICE_SID` (or `TWILIO_MESSAGING_FROM`) is set.
    Until then the app says so instead of pretending it sent something.
    **`api/sms.js` has never been run against Twilio.**
  - **Norwegian numbers are not purchasable in Twilio**, with voice or SMS
    (checked 2026-08-26). So the calling number stays US, and per-market
    senders are unavoidable when Europe and the US come.
  - **The text does not need to come from the calling number.** Piercing Do Not
    Disturb is what the *call* needs; a "she will be tired soon" text must not
    wake anyone. Two senders is correct here, not a compromise.
  - Worth weighing, and it is a design call: sent from the calling number the
    text arrives *as the saved contact*, "Sonja ♥️", which matches a message
    written in the baby's voice and signed "– Sonja 😴". From an alphanumeric
    sender it reads as from the service. The trade is deliverability: a US long
    code to a Norwegian mobile may be filtered. One test message answers it.
  - A **Messaging Service is needed either way** — not for the sender, but
    because scheduled sending requires it, and scheduling is what makes the
    text arrive with the page closed.
  - Scheduling matters more than sending: `schedule()` hands the send-at time
    to Twilio so the text arrives even with the page closed. Twilio requires
    send-at ≥15 min ahead, so `sms.js` falls back to sending immediately when
    it is nearer than that.
- **Coverage vs signal**: a *network* has coverage over an area, a *handset*
  has a signal. "A phone with coverage" is unidiomatic — native usage reserves
  "coverage" for a carrier's geographic reach. So: "a phone with a signal",
  but "internet and coverage behave differently everywhere". Checked against
  EC English, WordReference and Telstra's own network docs.
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
- **`.cta p` sets `margin:0 auto 36px`**, which outranks a bare `.cta-note`
  and silently zeroed its `margin-top`. Needs `.cta p.cta-note` to win.
- **`.hiw-copy` scrolls, `.hiw-view` clips.** The copy block is
  `overflow-y:auto`, so overflowing content is reachable, not lost — measured,
  not assumed. Steps 2 to 5 all overflow slightly at 375×667 and that is fine
  for reading. It is *not* fine for a control: anything the user must tap or
  see (the Mum/Dad row, an error) belongs above the fold, which is why the role
  row sits before the name and phone fields.

## Security

Full review, with a verified attack path, in `jonblund-voice/SECURITY-REVIEW.local.md`
(gitignored, and must stay so). Fixed 2026-08-26:

- **`esc()` in `babycall.html` is mandatory for any name going into `innerHTML`.**
  Names come from the user *and* from the phone's address book via Contact Picker,
  so a poisoned vCard could reach them. Three sites were unescaped; script did
  execute, verified with a live payload before and after. Prefer `textContent`.
- **The Twilio SDK carries an SRI hash.** Bump it with the version, never
  separately — a wrong hash means the SDK will not load and Jon Blund cannot
  ring. Recompute with `openssl dgst -sha384 -binary … | openssl base64 -A`.
- **The adapter seam is wrapped so any call lights up the screen.** The adapter
  touches no UI state by design, the remote stream is muted and the screen dims,
  so an injected `JONBLUND_TELEPHONY.call()` was completely invisible. The
  wrapper is defence in depth, not a lock.
- **`timeLimit` is 600, not 1800**, in `jonblund-voice/api/voice.js`. Cheapest cap
  on damage per call while the key is public, and it protects a customer who
  falls asleep with the line open.
- CSP is a `<meta>` and therefore only `base-uri` and `object-src`.
  `frame-ancestors` is header-only and GitHub Pages cannot set headers.

Still open, all needing the auth work: the public key mints Twilio tokens for
anyone (verified from curl, no browser), one shared `identity: 'crib'`, no rate
limit, and **the spending ceiling exists only in the client** — so it is a
display, not a limit. Check-in calls make that last one urgent.

**`JONBLUND_ALLOWED_NUMBERS` stays `*` on purpose** while two testers are trying
the app (decided 2026-08-26). Closing it mid-test risks a tester's call silently
failing on a number typed a little differently, which reads as "the app is
broken". Petter's call, and a defensible one *only because* the three things that
bound the damage are confirmed in place: geo permissions Norway-only,
auto-recharge off, `timeLimit` 600. Residual worst case is a drained Twilio
balance, not an unbounded bill.

**Revisit the moment any one of those three changes**, and before the first
paying customer. `+47*` prefix support in `voice.js` would be the middle ground
that cannot break a Norwegian tester; not built, because the decision was to
leave the variable alone.

**Do not open Twilio geo permissions beyond Norway until auth lands.** Norway-only
is what currently keeps toll fraud near worthless, and it is now load-bearing
rather than incidental.

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
