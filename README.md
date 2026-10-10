# Optim6 invitation — manual test pages

Two static pages for exercising the UC-01 / UC-02 invitation flow by hand.
Personal test harness — not a product surface, not the client's landing page.

| Page | Role |
|---|---|
| `index.html` | Resolves the `?token=` from the mail link, shows what the invitation offers, and offers **both** proof paths as separate panels: a password form, and the provider round trip |
| `callback.html` | The registered `oauth2.redirectUri`. Receives `code` + `state` and posts the ACCEPT |
| `signin.html` | Ordinary sign-in outside the portal, for getting a token into Postman. Two separate cards: a **password** form that returns a session in one call, and the provider round trip |

Both show every request and the raw response, so a refusal is diagnosable from the page.

## The two paths

An account holds exactly **one** login method, fixed at creation and never changeable (ID-4), so at
most one of these can succeed for any given link. `index.html` renders them as two labelled panels,
never as one form with a provider button bolted on — mixing them on screen is how a tester ends up
sending a password body to a provider account and reading `403 / 1413` as a bug.

| `accountState` | Password panel | Provider panel |
|---|---|---|
| `NONE` — no account yet | `{ action:"ACCEPT", firstName, lastName, password }` → creates the account, login method `LOCAL` **for life**, and returns a session. **No decline** | `{ action, state, code }` via `callback.html`. No names in the body. `DECLINE` allowed since 6.25.0 |
| `REVOKED` — account exists, zero grants | `{ action:"ACCEPT"｜"DECLINE", password }` → the account's **existing** password is the proof (IV-11 ②). No names | `{ action, state, code }`, and it must be the account's existing `(config, subject)` |
| `ACTIVE` | — | — (link is navigation only; `409 / 1438`) |

On `REVOKED` the resolve response's `loginMethod` says which one applies; the page shows that panel
and names the other as refused rather than hiding it.

**There is no username field anywhere.** The address is fixed by the token and is rendered read-only:
the body the contract defines carries `password`, never a username.

### Why there is a password box at all

Earlier revisions of this page deliberately had none, on the grounds that a page reached from a
mailed link should not train anyone to type a password into it. That reasoning is about the
**client's** landing page and still holds for it. This is a manual test harness, and UC-02's password
path (OC-11268) and UC-04's `REVOKED` password proof are otherwise only exercisable with a
hand-rolled `curl`. The panels are labelled so nobody mistakes this for the product surface.

## The provider flow

```
mail  ──►  index.html?token=<token>
             │  GET  /unauth/platform/v6/invitations/{token}      → accountState, email, isp, roles
             │  GET  /unauth/platform/v6/oauth2/providers          → the provider picker
             ▼
        [Continue with Google] / [Continue with Microsoft]
             │  POST /unauth/platform/v6/oauth2/authorizations
             │       { providerId, purpose: "INVITATION", invitationToken }
             │       → { authorizationUrl, state, expiresIn }
             ▼
        provider  ──►  callback.html?code=…&state=…
             │  POST /unauth/platform/v6/invitations/{token}/action
             │       { action: "ACCEPT", state, code, firstName?, lastName? }
             ▼
        SessionResponse
```

**The provider fork lives here, not in the mail.** The authorize URL is built by the
server and carries a state that lives `oauth2.authorizationStateTtlSeconds` (600s by
default). A link baked into an invitation mail would be dead long before anyone opened
it, and the enabled-provider list is per-deployment. So the mail carries one link; this
page offers the choice.

## `signin.html` — the two sign-in methods

Both land on `POST …/unauth/platform/v6/sessions`, discriminated by `method`. They are separate
cards for the same reason `index.html`'s panels are: ID-4 fixes an account's one login method at
creation, so at most one can work for any given address.

| Method | Body | Result |
|---|---|---|
| `PASSWORD` | `{ method:"PASSWORD", username, password, role?, isp? }` | The response **is** the session — one call, no round trip. The v6 replacement for `POST /v2/login` and `POST /unauth/platform/v5/login`, both frozen and deprecated |
| `PROVIDER_CODE` | `{ method:"PROVIDER_CODE", state, code }`, posted by `callback.html` | Same session shape, after the provider round trip |

`role` / `isp` are optional and validated as a **pair** (SW-2) — an account granted `admin` in ISP B
and `engineer` in ISP A asking for `admin` in ISP A is refused, even though each value appears in one
of its grants. Blank is absent. `isp` is a compartment id; `*` is GLOBAL.

**A password presented for a provider account is `401` / `1470`**, exactly like a wrong password, and
deliberately not told apart: distinguishing them would make the endpoint an oracle for which accounts
exist and how they sign in. The four refusals the page annotates:

| | |
|---|---|
| `401` · `1470` `INVALID_CREDENTIALS` | Unknown address **or** wrong password — one code for both |
| `401` · `1471` `ACCOUNT_LOCKED` | Answered **only once the password verified**, so the lock is not an oracle for account existence. Administrative reset; no `lockedUntil` exists to report |
| `403` · `1411` `NO_ACCESS` | Authenticated, zero activated grants (ID-6). No token, no user row |
| `403` · `1415` `INVITATION_PENDING` | Same, except an invitation is waiting — use the link from the mail, which `index.html` handles |

The password is redacted from the on-page log before it is printed; the response is not.

## Setup

Three things must agree on the callback URL:

1. **This page's URL** — e.g. `https://<user>.github.io/<repo>/callback.html`
2. **`oauth2.redirectUri`** on the target environment — the platform's single registered
   redirect URI. The server builds every authorize URL from it and echoes it on the code
   exchange, which OIDC requires to match.
3. **The provider app config** — the same URL registered as an authorized redirect URI in
   the Google Cloud console and/or the Microsoft Entra app registration.

Then open the link from the invitation mail. `index.html` takes the token from `?token=` and
nothing else, and the API origin is baked into the page — neither is typed in, because a page
that lets the visitor retype the origin can be pointed at somebody else's API, and a token box
invites pasting a link somebody else received. Point the page at another environment by editing
`API` at the top of its script.

The in-flight round trip is kept in `sessionStorage` and is **per-tab**, so start and finish in
the same tab.

## Two things that will bite first

**CORS.** These pages are a different origin from the API, so every call is cross-origin.
The browser will block the response unless the API returns
`Access-Control-Allow-Origin` for this page's origin. Both pages call this out explicitly
in the log when a request fails at the network layer, because it looks identical to the
API being down.

**`portal.baseUrl`.** The invitation mail refuses to send when it is unset, rather than
mailing a link that goes nowhere. If no mail arrives, check that first.

## Password policy

The `NONE` password box mirrors `ApplicationConstants.PASSWORD_PATTERN` via the contract — 10 to 64
characters with at least one lowercase letter, one uppercase letter, one digit and one special
character. The page checks it **only** to spare a round trip; the server is the authority and
answers `400` / `1474` `PASSWORD_POLICY_VIOLATION`, which the page shows unedited.

The `REVOKED` password box carries **no** policy check and no length bounds, deliberately: it
verifies a stored credential rather than setting a new one, and bounding it would reject a password
predating the current policy with a policy error rather than an authentication one.

## What the pages deliberately do not do

- **No password `DECLINE` when `accountState` is `NONE`.** With no account there is no password to
  verify, so ignoring the mail is the decline and the IV-8 reaper purges the invitation at expiry.
  Declining actively needs the provider round trip (IV-11 ③, 6.25.0).
- **No names when `accountState` is `REVOKED`.** The account already exists; names are
  rejected there — on either path.
- **No action at all when `accountState` is `ACTIVE`.** The link is navigation only — sign
  in and accept from `/v6/me/invitations`.
- **They never hold a provider token.** The client sends `code`; the server exchanges it.

## Step-up (UC-12 / SW-5)

`stepup.html` starts the PROVIDER step-up round trip for a session that was refused `403 / 1408` with `stepUpMechanism: PROVIDER`: paste the calling session's accessToken and the target, it calls `INITIATE_STEP_UP`, stashes `purpose: STEP_UP` and sends you to the provider. `callback.html` then shows the code and state and completes the elevation with `SWITCH_CONTEXT + stepUp {code, state}` on that session — never through the sign-in route, which would consume the single-use state (1472 there, 1421 on the completion).
