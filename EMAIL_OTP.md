~# Passwordless Email OTP login on the `bharat-vistaar` realm

A third way to sign in, alongside the existing password login and Google SSO
(see [GOOGLE_SSO.md](./GOOGLE_SSO.md)): **enter your email → receive a
6-digit code → enter the code → you're in.** No password involved anywhere
in this path.

This document is the full handover record for it — the design decisions,
every command that was run to build it, the two real bugs that were hit and
fixed along the way, and what's left before it's fully production-ready
(real email delivery). Read this before touching the flow in the admin
console or `realm-export.json` — the structure looks unusual for a reason,
explained in §3.

**New to this project?** Start with
[EMAIL_OTP_GETTING_STARTED.md](./EMAIL_OTP_GETTING_STARTED.md) instead —
a plain-language walkthrough of what this is and how to try it, no prior
Keycloak/Docker knowledge assumed. Come back here once you're ready for
the "why it's built this way" detail.

## Contents

1. [What this actually is](#1-what-this-actually-is)
2. [Why this needed a third-party provider](#2-why-this-needed-a-third-party-provider)
3. [Flow architecture — and the two bugs that shaped it](#3-flow-architecture--and-the-two-bugs-that-shaped-it)
4. [The theme integration](#4-the-theme-integration)
5. [Files touched, end to end](#5-files-touched-end-to-end)
6. [Rebuilding the provider JAR](#6-rebuilding-the-provider-jar)
7. [Going live with real email](#7-going-live-with-real-email)
8. [Production security checklist](#8-production-security-checklist)
9. [How to test this yourself](#9-how-to-test-this-yourself)
10. [Troubleshooting](#10-troubleshooting)
11. [Full kcadm command log](#11-full-kcadm-command-log)

---

## 1. What this actually is

- **Scope decision**: added *alongside* password login and Google SSO, not
  replacing either. All three appear as tabs on the same login page.
- **Genuinely passwordless**: the email address itself is the only
  credential. There's no password fallback hidden behind it, and no
  authenticator-app enrollment step.
- **Not a second factor**: this is deliberately different from the "email
  OTP as 2FA after your password" pattern most guides describe (including
  the upstream project's own docs — see §2). Here, the code sent to your
  email *is* the whole login.

## 2. Why this needed a third-party provider

Keycloak has **no built-in authenticator that emails a one-time code for
login** — verified directly against the official docs (release notes for
every version up to 26.7.1, the current release as of this writing) before
building anything. What Keycloak ships natively is TOTP/HOTP via an
authenticator *app* (Google Authenticator, etc.), and one-time *links* for
email verification — neither is "type your email, get a code by email, type
the code."

**Provider used**: [`mesutpiskin/keycloak-2fa-email-authenticator`](https://github.com/mesutpiskin/keycloak-2fa-email-authenticator),
Apache-2.0, the most actively maintained option in this space (tracks
current Keycloak releases, supports Keycloak SMTP/SendGrid/AWS SES/Mailgun
with fallback, configurable TTL/resend-cooldown/max-attempts, ships tests).

**Vendored, not pulled live**: the exact source (tag `v26.4.3`, commit
`c531418865fd6c75ad92f676fcb918435752366e`, Apache-2.0) is committed into
this repo at
[`providers/email-otp-authenticator/`](./providers/email-otp-authenticator/),
**unmodified** — see
[`PROVENANCE.md`](./providers/email-otp-authenticator/PROVENANCE.md) there
for the full record and how to upgrade it later. We build our own JAR from
this source (§6) rather than depending on a Maven Central artifact or a
live git fetch, so the code that runs in production is always readable
right here in this repo, and an upgrade is a reviewed commit, not a silent
version bump.

**Used in an upstream-undocumented way, on purpose**: upstream's own docs
present this authenticator *only* as a 2FA step run after a password check
— nothing in their docs covers passwordless use. We use it differently, and
this is a verified, deliberate choice, not a guess:

- `EmailAuthenticatorForm.requiresUser()` returns `true` — it only needs
  *some* prior step in the flow to have already resolved `context.getUser()`.
- `authenticate()`/`action()` only ever read `context.getUser()` and the
  auth session. Nothing in the class checks *how* the user got resolved, or
  cares whether a password was involved.

So pairing it with Keycloak's own **built-in** `Username Form` authenticator
(`auth-username-form` — identifier only, no password, plain stock Keycloak
code) as the step immediately before it produces a genuinely passwordless
flow, using the vendored authenticator exactly as written. Zero code
changes to it were needed.

**Keycloak version bump**: `docker-compose.yml`'s Keycloak image was bumped
from `26.0` to `26.6.1`, because the vendored provider's `pom.xml` compiles
against Keycloak SPI `26.6.1` and SPI compatibility across Keycloak minor
versions isn't guaranteed. If you ever bump the Keycloak image tag again,
rebuild the provider JAR against a matching `keycloak.version` first — don't
let the two drift apart silently.

## 3. Flow architecture — and the two bugs that shaped it

### The shape

The realm's `browserFlow` binding points at **`browser with email otp`** —
a full duplicate of Keycloak's built-in `browser` flow (duplicated because
built-in flows can't be edited in place — same reason `first broker login
google` exists as a copy, see GOOGLE_SSO.md). Duplicating instead of
building from scratch means the existing password + 2FA + Google + WebAuthn
behavior is preserved byte-for-byte; only one thing was added to it.

That one addition is a new **top-level sibling** flow, `Email OTP Login`,
containing exactly two `REQUIRED` steps:

```
browser with email otp                          (top-level, realm's browserFlow)
├── Cookie                          ALTERNATIVE  (unchanged, stock)
├── Kerberos                        DISABLED     (unchanged, stock)
├── Identity Provider Redirector    ALTERNATIVE  (unchanged, stock — Google broker)
├── Organization ...                ALTERNATIVE  (unchanged, stock)
├── forms                           ALTERNATIVE  (unchanged, stock)
│   ├── Username Password Form      REQUIRED
│   └── Browser - Conditional 2FA   CONDITIONAL  (TOTP app / WebAuthn / recovery codes)
└── Email OTP Login                 ALTERNATIVE  ← the only new thing
    ├── Username Form                REQUIRED    (built-in auth-username-form — identifier only)
    └── Email OTP                    REQUIRED    (email-authenticator — the vendored provider)
```

Everything under `forms` is **untouched stock Keycloak**. `Email OTP Login`
is a peer of `forms`, not a child of it. This matters — see Bug #1.

### Bug #1: `CONDITIONAL` + `ALTERNATIVE` siblings don't mix

**First attempt** put `Email OTP Login` *inside* `forms`, as a sibling of
`Username Password Form` (with both flipped to `ALTERNATIVE`, so either
path could satisfy `forms`). This silently broke every login attempt:

```
WARN [org.keycloak.authentication.DefaultAuthenticationFlow] REQUIRED and
ALTERNATIVE elements at same level! Those alternative executions will be
ignored: [auth-username-password-form, null]
```

Root cause: `forms` already contains `Browser - Conditional 2FA` as a
`CONDITIONAL` sibling (stock Keycloak, unrelated to us). Keycloak's flow
engine does not allow `CONDITIONAL` and `ALTERNATIVE` executions as direct
siblings — when it detects the mix, it ignores the alternatives entirely,
which broke *all* login, not just the new option.

**Fix**: moved `Email OTP Login` out of `forms` entirely, up to be a
sibling of `forms` itself at the top level (where the existing siblings —
Cookie, Identity Provider Redirector, Organization — are already all
`ALTERNATIVE`/`DISABLED`, no `CONDITIONAL` there to clash with). `forms`
was reverted to its exact original stock shape. This is why the tree above
looks the way it does — it's not arbitrary, it's the only placement that
doesn't collide with the existing 2FA conditional.

### Bug #2: priority ordering picked the wrong default tab

Once moved to the top level, `Email OTP Login` was created with priority
`0` — lower than `Cookie` (`10`), `Identity Provider Redirector` (`20`), and
`forms` (`26`) — making it the *first* alternative Keycloak tries. Since
Cookie/IdP-Redirector/Organization all silently no-op without a real
session, the **Email OTP identifier form became the default page shown to
every visitor**, displacing the password form — the opposite of the "add
as a third option" decision.

**Fix**: five `lower-priority` calls (see §11) moved it to the end, after
`forms`, so password stays the default tab and Email OTP is purely
additive. If you ever re-order top-level executions in the admin console,
double check which one ends up with the lowest priority — it becomes the
default.

### Why `Username Form` is the anchor point for the theme

The theme (§4) needs to send the user into this branch from a tab click.
Keycloak's *only* native mechanism for jumping to a specific alternative
mid-flow is the same one its own "Try Another Way" chooser uses: `POST
authenticationExecution=<leaf execution id>` to `${url.loginAction}`. That
id has to be a **leaf authenticator's** execution id — submitting the
*flow's own* execution id (`Email OTP Login`'s id) gets rejected with
`Requested authentication execution is not allowed`, because Keycloak
validates against `auth.authenticationSelections`, which is built from leaf
executions (see `AuthenticationSelectionResolver` in Keycloak's source —
this was read directly, not guessed, to get this right). The correct target
is `Username Form`'s execution id, which is what the theme looks up live.

## 4. The theme integration

Three theme files under
[`themes/data-ingestion-platform/login/`](../themes/data-ingestion-platform/login/)
changed or were added:

| File | What it does |
|---|---|
| `login.ftl` | Added a third "Email OTP" tab (CSS-only radio, same mechanism as the existing Google/Email tabs). Its panel is a tiny form that POSTs `authenticationExecution=<id>` to switch into the passwordless branch. |
| `login-username.ftl` | **New.** Overrides Keycloak's built-in identifier-only form (which `Username Form` renders by default) so this step stays on-brand instead of falling back to unstyled parent-theme markup — this theme deliberately doesn't load the parent's stylesheet (see `theme.properties`). |
| `email-code-form.ftl` | **New.** Overrides the OTP-entry template the vendored provider ships (as a `theme-resources`-contributed default) — same reasoning, keeps the code-entry page on-brand. Shows the masked email, resend/cancel buttons, and a resend-cooldown hint. |

The execution id used in the tab's hidden field is **never hardcoded** — it
would break the moment the flow is rebuilt. It's resolved live, in
FreeMarker, from `auth.authenticationSelections` (the same server-computed
list Keycloak's own `select-authenticator.ftl` reads from):

```ftl
<#assign emailOtpExecId = "">
<#if auth?has_content && auth.authenticationSelections?has_content>
    <#list auth.authenticationSelections as selection>
        <#if selection.authenticationExecution.authenticator == "auth-username-form">
            <#assign emailOtpExecId = selection.authExecId>
        </#if>
    </#list>
</#if>
```

If that lookup ever comes back empty (e.g. someone disables the branch),
the tab degrades to a disabled state with an explanatory message instead of
a broken link — see the `<#if !emailOtpExecId?has_content>disabled</#if>`
on the tab's radio input.

CSS additions live in `styles.css` §9 (third-tab layout) and §9B (OTP page:
`.dip-otp-desc`, `.dip-otp-masked-email`, `.dip-back-link`,
`.dip-btn-secondary`, `.dip-form-buttons-row`). No changes were needed in
`app.js` — its tab-ARIA-sync code already queries `.dip-tab-radio`/`.dip-tab-btn`
generically, so the third tab picked it up automatically.

## 5. Files touched, end to end

```
keycloak/
├── docker-compose.yml                        Keycloak image 26.0 → 26.6.1; new
│                                              provider JAR volume mount; new
│                                              smtp-setup service
├── .env.example                              NEW — was referenced by GOOGLE_SSO.md
│                                              but never actually existed; added
│                                              SMTP_* vars alongside GOOGLE_*
├── realm-export.json                         + browserFlow binding, 6 new
│                                              authenticationFlows, 2 new
│                                              authenticatorConfig entries
├── EMAIL_OTP.md                              this file
├── scripts/
│   └── configure-smtp.sh                     NEW — realm SMTP config via kcadm,
│                                              optional (skips gracefully if unset)
└── providers/
    ├── build.sh                              NEW — Dockerized Maven build
    ├── email-otp-authenticator.jar           built artifact (gitignored)
    └── email-otp-authenticator/              vendored upstream source (see §2)
        ├── PROVENANCE.md                     NEW — origin/license/commit record
        ├── LICENSE, UPSTREAM_README.md       copied verbatim from upstream
        ├── pom.xml                           unmodified
        └── src/                              unmodified

themes/data-ingestion-platform/login/
├── login.ftl                                 + Email OTP tab
├── login-username.ftl                        NEW — branded identifier-only step
├── email-code-form.ftl                       NEW — branded OTP-entry step
└── resources/css/styles.css                  + §9 (3-tab layout), §9B (OTP page)

.gitignore                                    + keycloak/providers/*.jar,
                                                 */target/, .m2-cache/
```

## 6. Rebuilding the provider JAR

No local Java or Maven is required — the build runs Maven inside a
throwaway `maven:3.9-eclipse-temurin-21` container:

```bash
cd keycloak/providers
./build.sh
```

This produces `keycloak/providers/email-otp-authenticator.jar` (gitignored
— it's a build artifact, not source) from the vendored source. Re-run it
any time `providers/email-otp-authenticator/src` or `pom.xml` changes.

Then pick up the new JAR:

```bash
cd keycloak
docker compose up -d --force-recreate keycloak
```

`start-dev` re-runs Keycloak's build/augmentation step on every boot, so
the new JAR is picked up automatically — no extra `kc.sh build` step is
needed in this dev setup (that explicit step is only required for the
production `start` command; see the vendored project's own
`providers/email-otp-authenticator/UPSTREAM_README.md` and docs site for a
production multi-stage Dockerfile pattern if you deploy this outside the
current dev docker-compose).

## 7. Going live with real email

**Current state: simulation mode.** The `Email OTP` execution's config has
`simulationMode=true` — codes are written to `docker compose logs keycloak`
(`grep "SIMULATION MODE"`), never actually emailed. This is safe for local
dev and was necessary to build/verify this whole feature without real SMTP
credentials in hand. **It must not stay this way in production** — codes
would be visible in plaintext logs to anyone with log access.

To go live:

1. Get real SMTP credentials (a transactional email provider — SES,
   SendGrid, Mailgun, or your org's mail relay — not a personal/free-tier
   SMTP account; deliverability here directly gates whether login works at
   all). Verify a sending domain with SPF/DKIM configured.
2. Copy `keycloak/.env.example` to `keycloak/.env` if you haven't already
   (it already holds the Google credentials — add the `SMTP_*` values
   alongside them; the file is gitignored, same as Google's).
3. `cd keycloak && docker compose up -d smtp-setup` — this runs
   `scripts/configure-smtp.sh`, which sets the realm's SMTP settings via
   `kcadm` (same "credentials never touch a committed file" pattern as
   Google — see `configure-google-idp.sh`). Safe to re-run.
4. In the admin console (or via `kcadm`), verify delivery: **Realm Settings
   → Email → Test connection**.
5. Once confirmed, flip simulation mode off:
   ```bash
   docker exec keycloak-keycloak-1 /opt/keycloak/bin/kcadm.sh config credentials \
     --server http://localhost:3000 --realm master --user admin --password admin
   docker exec keycloak-keycloak-1 /opt/keycloak/bin/kcadm.sh update \
     "authentication/executions/EMAIL_OTP_EXECUTION_ID/config" -r bharat-vistaar \
     -s 'config."simulationMode"=false'
   ```
   (Find the current execution id via `kcadm get "authentication/flows/Email%20OTP%20Login/executions" -r bharat-vistaar`.)
6. Re-export and recapture into `realm-export.json` the same way §11 was
   built, so the change survives a fresh `--import-realm`.

## 8. Production security checklist

These were set deliberately, not left at library defaults — see the
authenticator config in `realm-export.json` (`email-otp-settings` alias):

| Setting | Value | Why |
|---|---|---|
| Code length | 6 digits | Industry-standard tradeoff between brute-force resistance and user error rate |
| TTL | 300s (5 min) | Long enough for a real inbox check, short enough to limit a leaked/intercepted code's window |
| Resend cooldown | 60s | Prevents resend-button abuse (email bombing) |
| Max attempts | 3 | Invalidates the code after 3 wrong guesses, forcing a fresh one rather than allowing unlimited brute-forcing of a 6-digit space |
| Skip setup | `true` | Any user with an email is eligible immediately — no separate enrollment step, matching the "just works" passwordless UX this was built for |
| Show masked email | `true` | Confirms where the code went without exposing the full address on-screen |
| Simulation mode | `true` (dev only) | **Must be `false` before production** — see §7 |

Still worth confirming before a real production rollout, not yet verified
here:

- **Brute-force protection coverage**: Keycloak's realm-level brute-force
  detection (`bruteForceProtected`) is designed around username/password
  attempts. Confirm it actually covers repeated wrong `emailCode` submissions
  for this authenticator too (the code's own `maxAttempts=3` limits
  same-code guessing, but check whether repeated *fresh* code requests
  against the same account are separately rate-limited at the account
  level, not just the resend-cooldown per-session).
- **Dependency scanning**: the vendored source pulls in the AWS SES and
  SendGrid SDKs as compile dependencies (unused by our `KEYCLOAK`-provider
  configuration, but present in the dependency tree). Run this through
  whatever dependency/CVE scanning this project already uses before
  shipping.
- **Load testing** the OTP-send path once real SMTP is wired up — a spike
  of login attempts is now also a spike of outbound email.

## 9. How to test this yourself

**In a browser** (the real way): go to the login page, click the **Email
OTP** tab, click **Continue with Email Code**, enter an email that matches
an existing user, then check `docker compose logs keycloak | grep
"SIMULATION MODE"` for the code (until real SMTP is wired per §7) and enter
it.

**From scratch**, to confirm nothing depends on leftover live state:

```bash
cd keycloak
docker compose down -v
docker compose up -d
```

This was done and re-verified during development — the whole flow (tab
click → email → simulated code from logs → verified login, redirecting
with a valid authorization code) works from a cold `--import-realm`, not
just against a container that still has the original kcadm session state
in memory. Password login and Google SSO were also re-checked afterward to
confirm neither regressed.

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `REQUIRED and ALTERNATIVE elements at same level!` in logs, all login broken | A `CONDITIONAL` execution shares a level with `ALTERNATIVE` executions | See Bug #1 in §3 — don't nest new alternatives inside `forms` |
| Email OTP tab shows as the *default* page instead of password | Its priority is lower than `forms`'s | See Bug #2 in §3 — lower its priority via the admin console (drag it below "Email/Password Form") or `kcadm .../lower-priority` |
| `Requested authentication execution is not allowed` | Submitted the flow's own execution id instead of the `Username Form` leaf id | See "Why `Username Form` is the anchor point" in §3 |
| Cookie-related failures when testing with `curl`/scripts (not real browsers) | Keycloak's session cookies are `Secure`-flagged; `curl`/Python's cookiejar (correctly) won't send them over plain `http://`, even to `localhost` — real browsers specially trust `localhost` and send them anyway | Thread cookies manually in test scripts (parse `Set-Cookie`, resend as a raw `Cookie` header) — not a bug in the flow itself |
| Login-username or code-entry page looks completely unstyled | The template fell back to the parent `keycloak` theme (which this theme doesn't load CSS for) | Confirm `login-username.ftl` / `email-code-form.ftl` exist directly under `themes/data-ingestion-platform/login/` — Keycloak resolves same-named templates in the active theme before falling back to the parent or a `theme-resources`-contributed default |
| New provider JAR doesn't seem to load | `docker compose up -d` without `--force-recreate` may reuse the old container | `docker compose up -d --force-recreate keycloak` |

## 11. Full kcadm command log

This is the exact sequence run against the live dev container to build and
verify the flow (kept for anyone who wants to reproduce, extend, or
understand it via CLI rather than the admin console). All commands assume
you're in `keycloak/` with the stack running, and `kc()` is a shell
shortcut: `kc() { docker exec keycloak-keycloak-1 /opt/keycloak/bin/kcadm.sh "$@"; }`,
authenticated first via:

```bash
kc config credentials --server http://localhost:3000 --realm master --user admin --password admin
```

```bash
# 1. Duplicate the built-in "browser" flow (can't edit built-ins in place)
kc create authentication/flows/browser/copy -r bharat-vistaar \
  -s newName="browser with email otp"

# 2. Add the new top-level alternative branch
kc create "authentication/flows/browser%20with%20email%20otp/executions/flow" -r bharat-vistaar \
  -s alias="Email OTP Login" \
  -s type=basic-flow \
  -s description="Passwordless login: identify by email, then verify a one-time code sent to that email." \
  -s provider=registration-page-form

# 3. Add its two steps
kc create "authentication/flows/Email%20OTP%20Login/executions/execution" -r bharat-vistaar \
  -s provider=auth-username-form
kc create "authentication/flows/Email%20OTP%20Login/executions/execution" -r bharat-vistaar \
  -s provider=email-authenticator

# 4. Set requirements (-n skips kcadm's GET-and-merge, which fails against
#    this array-returning endpoint)
kc update "authentication/flows/browser%20with%20email%20otp/executions" -r bharat-vistaar -n \
  -s id=<Email-OTP-Login-flow-exec-id> -s requirement=ALTERNATIVE
kc update "authentication/flows/Email%20OTP%20Login/executions" -r bharat-vistaar -n \
  -s id=<Email-OTP-leaf-exec-id> -s requirement=REQUIRED

# 5. Configure the Email OTP step (see §8 for the values used)
kc create "authentication/executions/<Email-OTP-leaf-exec-id>/config" -r bharat-vistaar \
  -s alias="email-otp-settings" \
  -s 'config."length"=6' \
  -s 'config."ttl"=300' \
  -s 'config."resendCooldown"=60' \
  -s 'config."maxAttempts"=3' \
  -s 'config."showMaskedEmailOnOtpForm"=true' \
  -s 'config."skipSetup"=true' \
  -s 'config."emailProviderType"=KEYCLOAK' \
  -s 'config."simulationMode"=true'

# 6. Fix the priority so password stays the default tab (Bug #2, §3) —
#    5 calls because it started at priority 0, ahead of 5 siblings
for i in 1 2 3 4 5; do
  kc create "authentication/executions/<Email-OTP-Login-flow-exec-id>/lower-priority" -r bharat-vistaar -b '{}'
done

# 7. Bind the new flow as the realm's browser flow
kc update realms/bharat-vistaar -s browserFlow="\"browser with email otp\""
```

**Note on Bug #1's fix** (§3): the sequence above already reflects the
*corrected* placement (top-level sibling of `forms`). The first attempt —
nesting inside `forms` — was undone with:

```bash
kc delete "authentication/executions/<the-nested-Email-OTP-Login-id>" -r bharat-vistaar
kc update "authentication/flows/browser%20with%20email%20otp%20forms/executions" -r bharat-vistaar -n \
  -s id=<Username-Password-Form-id> -s requirement=REQUIRED   # revert forms to stock
```
before re-creating it at the top level as shown in step 2 above.

**Capturing into `realm-export.json`**: rather than hand-writing the JSON,
the live state was pulled with a partial export and merged in as plain
text (not a full re-serialize, which would have reformatted the entire
file):

```bash
kc create "realms/bharat-vistaar/partial-export" -r bharat-vistaar \
  -s exportClients=false -s exportGroupsAndRoles=false -o > partial-export.json
```

...then the 6 new `authenticationFlows` entries and 2 new
`authenticatorConfig` entries (alias-matched — including one easy to miss:
`browser with email otp browser-conditional-credential`, a config that got
silently duplicated along with the flow copy in step 1) were spliced into
`realm-export.json` as raw indented JSON text at the right array
boundaries, and their `id` fields were stripped to match this file's
existing convention of letting Keycloak generate fresh ids on import
(subflows are already resolved by `flowAlias` string, not by id, so this
is safe). See §9 for the from-scratch verification that this capture is
complete and correct.
