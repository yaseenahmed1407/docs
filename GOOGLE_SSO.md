# Google SSO on the `bharat-vistaar` realm

Google is configured as a Keycloak identity provider (broker), alongside the
existing username/password login — not a replacement for it. Existing users
keep logging in with their password exactly as before.

There's also a third, passwordless login option (email + one-time code) —
see [EMAIL_OTP.md](./EMAIL_OTP.md).

## 1. One-time setup

1. Create an OAuth 2.0 Client ID in [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
   ("Web application" type).
2. Add this **Authorized redirect URI** — this is Keycloak's broker
   callback, not the app's own URL:
   ```
   http://localhost:3000/realms/bharat-vistaar/broker/google/endpoint
   ```
3. Copy `keycloak/.env.example` to `keycloak/.env` and fill in the real
   values:
   ```
   GOOGLE_CLIENT_ID=...
   GOOGLE_CLIENT_SECRET=...
   ```
   `keycloak/.env` is gitignored (matched by the root `.env` pattern) — it
   is never committed, and the credentials never appear as literals in any
   file that is.
4. `cd keycloak && docker compose up -d`

That's the whole setup. `docker compose up` brings up two services:

- **`keycloak`** — imports `realm-export.json` (roles, the app client, the
  theme, and a custom `first broker login google` authentication flow —
  none of which contain secrets).
- **`google-idp-setup`** — a one-shot container that waits for `keycloak`
  to report healthy, then runs `scripts/configure-google-idp.sh`, which
  uses `kcadm` (Keycloak's admin CLI, already inside the image) to create
  the `google` identity provider and its four mappers, reading
  `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` straight from its own
  environment. It's safe to re-run (`docker compose up google-idp-setup`)
  — it deletes and recreates the `google` provider each time, so repeated
  runs never produce duplicates.

## 2. Why credentials aren't in `realm-export.json`

Keycloak's realm-import (`--import-realm`, the Quarkus/Keycloak 26
distribution used here) does **not** expand `${env.VAR}` or `$(env.VAR)`
placeholders inside the imported JSON — that substitution only existed in
the older Wildfly-based Keycloak. This was verified directly against a
running Keycloak 26 container: both placeholder syntaxes came through as
literal strings on the `clientId`/`clientSecret` fields, not the resolved
values. So the only place credentials can be injected from the environment
is at runtime, via `kcadm` talking to the already-running server — which is
exactly what `google-idp-setup` does. Nothing Google-credential-shaped ever
touches a file that gets committed.

## 3. What got configured, and why

| Setting | Value | Why |
|---|---|---|
| Identity provider | `google` (built-in `google` provider type) | Keycloak already knows how to talk to Google's OIDC endpoints |
| Trust Email | `true` | Google-verified emails are treated as verified in Keycloak, which is what allows automatic account linking (below) |
| Sync Mode | `IMPORT` | Profile fields are copied into the Keycloak user at first login, not re-synced on every login |
| First Login Flow | `first broker login google` | See below |
| Mappers | `email`, `first-name` (`given_name`), `last-name` (`family_name`), `username` (`email`) | Required mappers from the spec; username mirrors email so it's stable and predictable for new accounts |

### Automatic account linking

`first broker login` is a Keycloak **built-in** flow, and built-in flows
can't be edited in place — Keycloak's admin API rejects it
("It is illegal to add execution to a built in flow"). So this realm
defines a copy, `first broker login google`, that's identical except for
one addition inside its "User creation or linking" step: an
**"Automatically set existing user"** (`idp-auto-link`) execution, set as
the first `ALTERNATIVE` tried.

Effect: when someone signs in with Google and their Google account's
verified email matches an existing Keycloak user's email, that user is
linked immediately — no password confirmation prompt, no duplicate
account. If no match is found, the flow falls through to the stock
`Create User If Unique` → `Handle Existing Account` behavior, unchanged.

This was configured and verified live against a running Keycloak instance
(via `kcadm`) before being captured into `realm-export.json`, so the JSON
you see there is a known-working configuration, not a guess.

### Linking/unlinking from the Account Console

No extra configuration was needed for this — it's stock Keycloak behavior.
Any logged-in user can go to the Account Console
(`http://localhost:3000/realms/bharat-vistaar/account/#/linked-accounts`)
and link or unlink their Google identity from there.

## 4. Theme changes

`themes/data-ingestion-platform/login/login.ftl` now renders two tabs
above the form — **Google SSO** and **Email Login** — implemented as a
CSS-only radio-button group (see `styles.css`, section 9), not JavaScript,
so the switcher still works even if `app.js` fails to load. Email Login is
the default selected tab. The Google tab shows Keycloak's own broker login
link (`${p.loginUrl}`, found by filtering `social.providers` for
`alias == "google"`) styled as a "Continue with Google" button — nothing
about how that link is generated or validated was touched.

`app.js` adds one small enhancement on top: it mirrors the radio group's
checked state into `aria-selected` on the tab labels for screen readers.
The tabs are fully operable without it.

## 5. Testing

- **Existing password login still works**: sign in with the seeded
  `admin` / `admin` user (or any existing user) on the Email Login tab —
  unaffected by any of the above.
- **New Google user**: sign in with a Google account whose email doesn't
  match any Keycloak user → a new user is created (via `Create User If
  Unique`, same as stock Keycloak).
- **Existing user, same email**: sign in with a Google account whose
  email matches an existing Keycloak user's email → that user is signed
  in directly, and a federated identity link is created silently (check
  Account Console → Linked Accounts to confirm).
