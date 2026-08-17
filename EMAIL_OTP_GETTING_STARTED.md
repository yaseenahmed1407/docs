~# Email OTP Login — Getting Started (zero experience required)

This guide assumes you've never touched Keycloak, Docker, or this feature
before. It walks through what this thing is, how to turn it on, and how to
try it yourself — in plain language, no prior knowledge assumed.

If you want the deep technical write-up (why it's built this way, every
command used to build it, production checklist), that's
[EMAIL_OTP.md](./EMAIL_OTP.md). This guide is the on-ramp *to* that
document, not a replacement for it.

## Contents

1. [What is this, actually?](#1-what-is-this-actually)
2. [Words you'll see, explained](#2-words-youll-see-explained)
3. [What you need before starting](#3-what-you-need-before-starting)
4. [Step 1 — start everything up](#step-1--start-everything-up)
5. [Step 2 — open the login page](#step-2--open-the-login-page)
6. [Step 3 — try the "Email OTP" tab](#step-3--try-the-email-otp-tab)
7. [Step 4 — find your code and finish logging in](#step-4--find-your-code-and-finish-logging-in)
8. [Why isn't a real email showing up in my inbox?](#8-why-isnt-a-real-email-showing-up-in-my-inbox)
9. [Where the pieces live](#9-where-the-pieces-live)
10. [If something goes wrong](#10-if-something-goes-wrong)
11. [What to do next](#11-what-to-do-next)

---

## 1. What is this, actually?

Normally, to log in to a website, you type a username and a password. This
project adds a **second way to log in that doesn't need a password at
all**:

1. You type your email address.
2. The system sends a 6-digit code to that email.
3. You type the code back in.
4. You're logged in.

That's the whole thing. No password, no app to install, no account
"recovery question." Just prove you can read an email sent to that address.

This sits **alongside** the existing ways to log in (regular
username+password, and "Sign in with Google") — it's a third tab on the
login page, not a replacement for the other two.

## 2. Words you'll see, explained

| Word | What it means here |
|---|---|
| **Keycloak** | The piece of software that handles "who is this user, are they allowed in." It's not part of the main app — it's a separate service the app talks to. |
| **Realm** | Keycloak's word for "a set of users and login rules for one project." This project's realm is called `bharat-vistaar`. |
| **Docker / container** | A way of running software (like Keycloak) in a self-contained little box, so you don't have to install it directly on your computer. `docker compose up` = "start the boxes this project needs." |
| **OTP** | "One-Time Password" — a code that works once and then stops working. That's the 6-digit code emailed to you. |
| **SMTP** | The technical name for "a service that actually sends emails." Right now this project doesn't have one configured (see §8), so no *real* emails go out yet. |
| **Simulation mode** | A stand-in for SMTP during development: instead of emailing you the code, Keycloak just writes it to a log file you can read yourself. See §8. |
| **Provider / JAR** | The actual code that knows how to generate a code and check it. It's a small plugin written in Java, dropped into Keycloak. You don't need to know Java to use it — `providers/build.sh` builds it for you. |
| **Flow** | The step-by-step recipe Keycloak follows during login (e.g. "ask for username → ask for password → let them in"). This project added a *new* recipe: "ask for email → send a code → ask for the code → let them in." |

## 3. What you need before starting

- **Docker Desktop**, installed and *running* (there's a whale icon in your
  menu bar/system tray when it's up). That's genuinely the only
  installation required — no Java, no Keycloak install, nothing else.
- A terminal, opened to this project's folder.
- That's it.

## Step 1 — start everything up

In your terminal:

```bash
cd keycloak
docker compose up -d
```

What this does: starts Keycloak itself, plus two small one-time setup
helpers (one wires up "Sign in with Google", one would wire up real email —
more on that in §8). The first time you run this it may take a minute or
two while Docker downloads what it needs. You'll know it's ready when the
command finishes and returns you to the prompt.

Check it's actually running:

```bash
docker compose ps
```

You should see a `keycloak` container with a status like `Up` or `healthy`.

## Step 2 — open the login page

Open this URL in your browser:

```
http://localhost:3000/realms/bharat-vistaar/account/
```

You should land on a "Sign in" page with **three tabs**: `Google SSO`,
`Email Login`, and `Email OTP`.

## Step 3 — try the "Email OTP" tab

1. Click the **Email OTP** tab.
2. Click **Continue with Email Code**.
3. Type an email address. It has to belong to a user that already exists
   in this system (this isn't open sign-up — someone has to have created
   the account first, the same way password logins work). If you're not
   sure which email to use, ask whoever set this up for you, or use
   `admin@example.com` (the built-in test admin account) for a first try.
4. Click **Send Code**.

You'll land on a new page asking for a 6-digit code, and it'll mention
which email address it was "sent" to.

## Step 4 — find your code and finish logging in

Right now, no real email actually goes out yet (see §8 for why). Instead,
the code gets written to Keycloak's own log. To see it, run this in your
terminal:

```bash
docker compose -f keycloak/docker-compose.yml logs keycloak --since 5m | grep "SIMULATION MODE"
```

You'll see a line like:

```
***** SIMULATION MODE ***** Email code send to admin@example.com for user admin is: 482913
```

That last number (`482913` here — yours will be different) is your code.
Type it into the browser and submit. You should now be logged in.

**Tip**: if you want to watch for the code live instead of checking after
the fact, open a second terminal tab and run this *before* you click "Send
Code" in the browser:

```bash
docker compose -f keycloak/docker-compose.yml logs -f keycloak | grep --line-buffered "SIMULATION MODE"
```

Leave it running — the code will appear there the moment you submit your
email.

## 8. Why isn't a real email showing up in my inbox?

Because nobody has connected this system to an actual email-sending service
yet. Right now it's running in **simulation mode** — a deliberate,
temporary stand-in used to build and test this whole feature before real
email credentials existed. Sending real email requires:

- Signing up with an email-sending service (examples: Amazon SES, SendGrid,
  Mailgun), or using an organization's existing mail server, **and**
- Getting a username/password (or API key) for it, **and**
- Plugging those into this project.

That last part is already built and ready — there's a script
(`keycloak/scripts/configure-smtp.sh`) that just needs real credentials
dropped into a file. The exact steps are in
[EMAIL_OTP.md §7 "Going live with real email"](./EMAIL_OTP.md#7-going-live-with-real-email).
If you don't have those credentials yet, that's the thing to go get before
this can send real emails — everything else is done.

## 9. Where the pieces live

You don't need to open any of these to just *use* the feature — this is
here so you know where to look if something needs changing later.

| If you want to... | Look here |
|---|---|
| Change how the login page *looks* (colors, text, logo) | `themes/data-ingestion-platform/login/` |
| Understand *why* the login flow is built the way it is | [`EMAIL_OTP.md`](./EMAIL_OTP.md) |
| Change how long a code lasts, its length, retry limits | `EMAIL_OTP.md` §8, or the admin console under Authentication → Flows |
| Add real email sending | `keycloak/scripts/configure-smtp.sh` + `EMAIL_OTP.md` §7 |
| Rebuild the underlying plugin after a code change | `keycloak/providers/build.sh` |

## 10. If something goes wrong

| What you see | What's probably happening |
|---|---|
| `docker compose up -d` fails immediately, mentions "Cannot connect to the Docker daemon" | Docker Desktop isn't open. Open it and wait for the whale icon to stop animating, then try again. |
| The browser just times out / "can't reach this page" | Keycloak hasn't finished starting yet — wait ~30 seconds and refresh, or run `docker compose ps` to check its status. |
| "Invalid username or password" right after typing your email | That email doesn't belong to an existing user yet. Someone needs to create the account first (see §3, step 3). |
| You typed a code and it says it's wrong/expired | Codes expire after 5 minutes, and only the *most recent* code works — if you clicked "Resend," use the newest one from the logs, not an older one. |
| Nothing shows up when you `grep "SIMULATION MODE"` | Make sure you're grepping *after* clicking "Send Code" in the browser, and that you're in the right folder when running the `docker compose` command (or use the full `-f keycloak/docker-compose.yml` form shown above from anywhere in the project). |

Still stuck? [EMAIL_OTP.md §10](./EMAIL_OTP.md#10-troubleshooting) has a
more technical troubleshooting table for issues specific to how the login
flow itself was constructed.

## 11. What to do next

- Read through [EMAIL_OTP.md](./EMAIL_OTP.md) once you're comfortable with
  the basics above — it explains every design decision and has the full
  history of what was tried and fixed while building this.
- If you're the one responsible for actually launching this for real
  users, the one thing standing between "simulation mode" and "real
  emails" is getting SMTP credentials — see §8 above.
