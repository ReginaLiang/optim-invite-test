# Optim6 invitation — manual test pages

Two static pages for exercising the UC-01 / UC-02 invitation flow by hand.
Personal test harness — not a product surface, not the client's landing page.

| Page | Role |
|---|---|
| `index.html` | Resolves the `?token=` from the mail link, shows what the invitation offers, and starts the provider round trip. Provider only — no password path |
| `callback.html` | The registered `oauth2.redirectUri`. Receives `code` + `state` and posts the ACCEPT |

Both show every request and the raw response, so a refusal is diagnosable from the page.

## The flow they implement

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

## What the pages deliberately do not do

- **No `DECLINE` when `accountState` is `NONE`.** With no account there is no identity to
  prove and the link is not a credential, so a decline would be an unauthenticated request
  deleting a record on the strength of a URL. Ignoring the mail is the decline; the reaper
  purges it at expiry.
- **No names when `accountState` is `REVOKED`.** The account already exists; names are
  rejected there.
- **No action at all when `accountState` is `ACTIVE`.** The link is navigation only — sign
  in and accept from `/v6/me/invitations`.
- **They never hold a provider token.** The client sends `code`; the server exchanges it.
