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
- [x] Navbar fixed to work from any page (`/#anchor`, not bare `#anchor`).
- [x] Mobile header wrap fix for narrow phones (<400px).

### Backend wiring required for each page above (marc_go, Railway env vars)

All follow the same optional pattern — empty = old Go-hosted fallback
behaviour, unchanged:

| Astro page | Env var to set on marc_go | Also needs |
|---|---|---|
| `/sahkan-emel` | `EMAIL_VERIFY_URL=https://marc.hafizbahtiar.com/sahkan-emel` | `CORS_ALLOWED_ORIGINS=https://marc.hafizbahtiar.com` (POST fetch) |
| `/pembayaran/pendaftaran` | `REGISTRATION_PAYMENT_RETURN_URL=https://marc.hafizbahtiar.com/pembayaran/pendaftaran` | none (passive redirect) |
| `/pembayaran/aktiviti` | `ACTIVITY_PAYMENT_RETURN_URL=https://marc.hafizbahtiar.com/pembayaran/aktiviti` | none (passive redirect) |
| `/sahkan-sijil` | `CERTIFICATE_VERIFY_URL=https://marc.hafizbahtiar.com/sahkan-sijil` | `CORS_ALLOWED_ORIGINS` already covers it (same var, list of origins) |

**Note on `/sahkan-sijil` and `CERTIFICATE_VERIFY_URL`:** this only affects
certificates *generated after* the env var is set — the QR code is baked
into the PDF at generation time (`fillPendingCertificateFiles`,
`marc_go/internal/http/handlers/activity_certificates.go`). Certificates
already printed keep pointing at the old Go-hosted JSON URL forever; that
route (`GET /verify/certificates/:token`) must stay working indefinitely,
it can never be removed.

None of these Railway env vars are set yet as of 2026-08-16 — until they
are, all four flows still work, just via the Go backend's own plain
HTML/JSON fallback pages instead of these branded ones.

`PUBLIC_API_BASE_URL` (marc_astro's own env var, `.env`) currently points
at the **staging** API (`https://marc-go-staging.up.railway.app`) —
production Railway isn't deployed yet. Update this + redeploy Astro when
production exists.

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
- **Password reset / magic link / invite link** — doesn't exist in the
  product at all (grepped for `ResetPassword`, `forgot`, `magic.link`,
  `invite` — no matches). Not a gap, just absent.
- **Certificate download** (`GET /me/certificates/:id/file`) — Bearer-token
  gated, returns a signed R2 URL for the *logged-in* member's own
  certificate. Needs an active mobile session; doesn't make sense as a
  public web page.
- **Approval/rejection and donation-receipt emails** — plain informational
  HTML, no clickable link embedded, nothing for a landing page to do.

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
