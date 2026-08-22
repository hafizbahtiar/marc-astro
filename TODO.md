# marc_astro — TODO

Static Astro site for MARC (Kelab Sukan dan Rekreasi MAIWP), deployed to
`marc.hafizbahtiar.com`. Acts partly as a marketing site, partly as the
web-landing frontend for a few flows the Go backend (`marc_go`) redirects
browsers into.

## Done

- [x] Homepage (Header, Hero, About, Features, Download, Disclaimer, Footer).
- [x] Terma dan Syarat (`/terma-dan-syarat`), Dasar Privasi (`/dasar-privasi`).
- [x] SEO/OG meta, favicon, canonical URL, Google Fonts, brand colour tokens
      matching `marc_flutter/lib/app/theme.dart`.
- [x] `/sahkan-emel` — email verification landing page. Reads `?token=`,
      calls `POST {PUBLIC_API_BASE_URL}/auth/verify-email/confirm`.
- [x] `/pembayaran/pendaftaran`, `/pembayaran/aktiviti` — ToyyibPay
      return pages. Read `?status_id=` from the redirect query string
      client-side (no fetch, no CORS needed — ToyyibPay's redirect is a
      passive browser navigation).
- [x] `/sahkan-sijil` — certificate verification page (scanned via QR on
      printed certificates). Reads `?token=`, calls
      `GET {PUBLIC_API_BASE_URL}/verify/certificates/:token`.
- [x] `/reset-kata-laluan` — password reset page (2026-08-22). Reads
      `?token=`, shows a password form, calls
      `POST {PUBLIC_API_BASE_URL}/auth/password-reset/confirm`. Unlike
      `/sahkan-emel` this does **not** auto-submit — it needs user input.
- [x] Navbar fixed to work from any page (`/#anchor`, not bare `#anchor`).
- [x] Mobile header wrap fix for narrow phones (<400px).
- [x] `public/.well-known/assetlinks.json` (2026-08-16) — Digital Asset
      Links for `com.hafizbahtiar.marc`, one `sha256_cert_fingerprints`
      entry (supplied by owner, not independently verified against Play
      Console — confirm it's the **App Signing** cert, not just a local
      upload/debug key, before relying on it). File only, **no**
      `autoVerify` intent-filter added in `marc_flutter`'s
      `AndroidManifest.xml` yet — deliberate: `/sahkan-emel` and
      `/sahkan-sijil` still deep-link nowhere in the Flutter app (no
      matching routes), so App Links would currently claim the domain
      without anywhere for it to send the user. Revisit together if/when
      in-app confirm screens for those two flows get built.

### Backend wiring required for each page above (marc_go, Railway env vars)

All follow the same optional pattern — empty = old Go-hosted fallback
behaviour, unchanged:

| Astro page | Env var to set on marc_go | Also needs |
|---|---|---|
| `/sahkan-emel` | `EMAIL_VERIFY_URL=https://marc.hafizbahtiar.com/sahkan-emel` | `CORS_ALLOWED_ORIGINS=https://marc.hafizbahtiar.com` (POST fetch) |
| `/pembayaran/pendaftaran` | `REGISTRATION_PAYMENT_RETURN_URL=https://marc.hafizbahtiar.com/pembayaran/pendaftaran` | none (passive redirect) |
| `/pembayaran/aktiviti` | `ACTIVITY_PAYMENT_RETURN_URL=https://marc.hafizbahtiar.com/pembayaran/aktiviti` | none (passive redirect) |
| `/sahkan-sijil` | `CERTIFICATE_VERIFY_URL=https://marc.hafizbahtiar.com/sahkan-sijil` | `CORS_ALLOWED_ORIGINS` already covers it (same var, list of origins) |
| `/reset-kata-laluan` | `PASSWORD_RESET_URL=https://marc.hafizbahtiar.com/reset-kata-laluan` | `CORS_ALLOWED_ORIGINS` must include the site origin (POST fetch) |

**`/reset-kata-laluan` does NOT follow the optional pattern above.** The
other four degrade to a Go-hosted fallback page when their env var is
empty; this one does not, deliberately — a password form is not something
that should appear from a fallback page nobody designed. Leave
`PASSWORD_RESET_URL` unset and `POST /auth/password-reset/request` returns
**503** to every member who taps "Lupa kata laluan?" in the app. There is
no degraded mode: the feature is either wired or off.

**Note on `/sahkan-sijil` and `CERTIFICATE_VERIFY_URL`:** this only affects
certificates *generated after* the env var is set — the QR code is baked
into the PDF at generation time (`fillPendingCertificateFiles`,
`marc_go/internal/http/handlers/activity_certificates.go`). Certificates
already printed keep pointing at the old Go-hosted JSON URL forever; that
route (`GET /verify/certificates/:token`) must stay working indefinitely,
it can never be removed.

None of these Railway env vars are set yet as of 2026-08-22. For the
**first four** flows that's cosmetic — they still work, just via the Go
backend's own plain HTML/JSON fallback pages instead of these branded
ones. For `/reset-kata-laluan` it is not: there is no fallback, so
password reset is **off** until `PASSWORD_RESET_URL` is set.

`PUBLIC_API_BASE_URL` (marc_astro's own env var, `.env`) currently points
at the **staging** API (`https://marc-go-staging.up.railway.app`) —
production Railway isn't deployed yet. Update this + redeploy Astro when
production exists.

This is a build-time inline, not a runtime read (static build, no adapter),
so a stale value ships baked into the HTML. It matters most for
`/reset-kata-laluan`: a production member's reset token exists only in the
production DB, so a page pointing at staging answers a perfectly valid
ten-second-old link with *"Pautan tidak sah atau telah luput"* — and the
request never reaches production, so nothing in its logs shows it happened.
For `/sahkan-emel` the same mistake is a retryable no-op; here it's a dead
end with actively misleading copy.

## Backlog / considered, not building

Surveyed `marc_go/internal` (2026-08-16) for any other backend flow that
sends a human-facing link or renders raw JSON/plain-HTML for a browser.
Everything else checked out as either already covered above or not a fit:

- **Stripe donations** (`DonationHandler.Checkout`) — no hosted-checkout
  success/cancel return URL exists anywhere in the backend (grepped
  `config.go`, `cmd/api/main.go`, `internal/payment/*.go` for
  `success_url`/`cancel_url` — zero hits). Donation flow is driven
  client-side via the Stripe mobile SDK, not a redirect. Nothing to attach
  an Astro page to unless/until Stripe Checkout Sessions get added
  server-side.
- **Magic link / invite link** — don't exist in the product (grepped for
  `magic.link`, `invite` — no matches). Not a gap, just absent.
  (**Password reset** was in this list until 2026-08-22 — it's now built,
  see `/reset-kata-laluan` under Done.)
- **Certificate download** (`GET /me/certificates/:id/file`) — Bearer-token
  gated, returns a signed R2 URL for the *logged-in* member's own
  certificate. Needs an active mobile session; doesn't make sense as a
  public web page.
- **Approval/rejection and donation-receipt emails** — plain informational
  HTML, no clickable link embedded, nothing for a landing page to do.

## Known polish (not blocking)

- [ ] `/reset-kata-laluan` is a one-shot form: `show(error)` hides the form
      permanently, and the fixed hint says *"Kembali ke app MARC dan minta
      pautan reset yang baharu."* That advice is wrong for the two most
      likely failures. A member on shared NAT who requested a few times
      (rate-limit bucket is per-IP, burst 5, shared between `/request` and
      `/confirm`) gets a **429** on submit — and a plain network blip hits
      the same dead end. In both cases the token is still perfectly valid;
      nothing was consumed. Fix: on non-400 branches, return to `show(form)`
      with an inline error instead of the terminal error state.
      (Final L32 review, 2026-08-22.)

## Design backlog (from earlier feedback, not started)

- [ ] Redesign `Features.astro` — currently a generic 3-column
      emoji-in-circle-badge grid, reads as an AI-website-builder template.
      Consider: real iconography instead of emoji, asymmetric/varied
      layout, leaning into the actual crest/badge visual identity now that
      the real logo and club story are known.
- [ ] Any new below-the-fold image introduced by the redesign above should
      rely on Astro `<Image>`'s default `loading="lazy"` — don't add
      `loading="eager"` unless it's genuinely above the fold on first
      paint (mirrors the Header/Hero logos, which are the only two eager
      images on the site today, both legitimately above the fold).
