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
  - **It is a slim bar under the wordmark**, chosen from three designs on
    2026-08-26 over a hero clock and a progress rail. The orb stays the centre
    of the app, so the clock informs and does not compete: 16px Fraunces, one
    line, tap to drop the adjust drawer. The drawer only opens while she is
    asleep, because there is nothing to move otherwise.
  - The drawer is `position:absolute` under a positioned `.nap-top`, so it rolls
    down *over* the orb instead of pushing the page. The shadow is load-bearing:
    without it the panel reads as part of the flow rather than something on top.
  - The baby's name was in the bar and came out again: it truncated the line on
    a long name and told the reader nothing they did not know.
  - The slider is clamped so she cannot have fallen asleep in the future, and
    the clamp is rounded to the slider's own five-minute step — otherwise the
    control shows one number and the value holds another.

### The tired message, removed for now

Built, then taken out again on 2026-08-26 while A2P registration is pending.
The clock stayed. To bring it back, revive from commit **5decc2c** (client) and
**af071e4** (`api/sms.js`, still local and unpushed) — do not rewrite it.

Why it went: Twilio will not offer an unregistered US 10DLC number as an SMS
sender at all, so the "From" list in the console is empty. Not Norway's doing,
US registration's.

The decisions that are settled, so nobody relitigates them:

- **One sender, and it is the calling number.** The message must arrive as the
  saved contact, "Sonja ♥️", because that is the one number a customer relates
  to. An alphanumeric sender would work today and was rejected for this reason.
- **Norwegian numbers are not purchasable in Twilio**, voice or SMS (checked).
  So per-market senders are unavoidable when Europe and the US come, and
  alphanumeric cannot reach the US at all.
- **A Messaging Service is needed** — not for the sender, but because scheduled
  sending requires one, and scheduling is what makes the text arrive with the
  page closed. Twilio wants send-at *more than* 15 minutes out, so `sms.js`
  floors at 16 and falls back to sending immediately rather than losing it.
- The sample message will need a brand and an opt-out line to clear US campaign
  review; the signature can stay "– Sonja 😴".
- Norwegian operators block SMS containing URLs unless allowlisted. Keep links
  out of it.
- `api/sms.js` inherits the shared-key problem from §Security: once configured,
  anyone holding the public key can send on the account. Fix it with the same
  auth work, before it goes live.
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

**Badges are sentence case, never caps.** The three pill badges are `.wall-tag`
and `.price-free` on the site and `.pr-free` in the app. The uppercase
letterspaced *eyebrows* and labels (`.sec-eyebrow`, `.opt-label`, `.status`,
`.hiw-num`, `.field label`, `.pr-unit`, `.price-unit`, `.nm-t`, `.vc-org`) are a
different device and stay as they are.

**Pricing rows must fit one line at 375px.** Checked by measuring, not by eye.
"Stops at" rather than "Capped at" — the service stops, the price is not
discounted, and the short word must not lie about that.

## Gotchas that cost real time

- **The call takes the microphone and does not always give it back.** After a
  Twilio call, iOS can leave the AudioContext suspended or hand back a track
  that is live but muted. Neither fires an event, so nothing recovers on its
  own and the app listens to silence forever — which is what it did until
  2026-08-26. The fix measures rather than guesses: a dead path reads about
  **-180 dB** and a real microphone never does, so `DEAD_DB = -150` is a safe
  line. Three seconds under it and `reviveMic()` resumes the context or
  rebuilds the chain from a fresh `getUserMedia`. It fails loudly into `error`
  rather than quietly. The successful rebuild cannot be tested without a real
  microphone, so verify it by making a call and watching the level ring come
  back.
- **The two buttons carry symbols, and Pause becomes Play.** While paused the
  pair reads Stop and Play, both staying put like a player rather than swapping
  places. `setBtn()` writes icon and label together — `textContent` alone would
  wipe the glyph. When this changed, the power handler still sent `paused` to
  `startMic()` from the days it said "Resume listening", so the button said Stop
  and started. Found by exercising the handlers, not by looking at them: worth
  re-running that check whenever a button's label changes state.
- **Pause is not Stop.** `pauseMic()` takes the microphone down and leaves the
  sleep session running; `stopMic()` calls `napWake()` and ends it. `startMic()`
  only starts a new session when there is not one in progress, which is what
  makes resume continue rather than reset — and also stops a page reload from
  wiping the nap.
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
- **Photos are blobs made of four `border-radius` pairs**, not SVG masks — the
  edge stays perfectly smooth, which is the point. Values near 50/50 read as a
  circle; real asymmetry (`58% 42% 33% 67% / 63% 37% 63% 37%`) is what makes a
  pebble. Four shapes cycle so they do not read as one stamp repeated.
  Originals live in the repo root, web copies in `img/` at ~70 KB each — 384 KB
  for all six, down from 17 MB.
- **`margin:0 auto` on a grid item makes it shrink to its content.** With a
  child at `width:100%` that is a circular dependency and everything collapses
  to zero. The hero blob was 0×0 on mobile until `.hero-art` got an explicit
  `width:100%`. The SVG that used to sit there had an intrinsic size from its
  viewBox and hid the bug for months.
- **`input[type="date"]` on iOS carries an intrinsic min-width** from the
  locale's date format, and `width:100%` will not shrink it below that, so it
  pushed out of the onboarding card. `min-width:0` plus
  `-webkit-appearance:none` holds it in. A desktop browser does not reproduce
  this, so check date fields on a real iPhone.

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
