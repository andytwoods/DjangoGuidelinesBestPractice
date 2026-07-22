# Project Guidelines

You are an expert in Python, Django, and scalable web apps. Write secure, maintainable, performant code.

---

## 1. Project Scaffold & Layout

Bootstrapped from [cookiecutter-django](https://github.com/cookiecutter/cookiecutter-django). Do not hand-roll structure it already provides.

**What is already in place — do not recreate:**
- Split settings: `config/settings/base.py`, `local.py`, `production.py`, `test.py`
- Custom `User` model at `<project_name>/users/models.py`; `AUTH_USER_MODEL = "users.User"` already set
- Whitenoise, Redis, django-allauth (with email verification), django-environ, argon2 passwords
- `pyproject.toml` with `uv` for dependency management (no `requirements/` folder)

**Layout rules:**
- Django apps directory: `<project_name>/` (`APPS_DIR = BASE_DIR / "<project_name>"`)
- All apps live under `<project_name>/`: e.g. `<project_name>/plunges/`, `<project_name>/tasks/`
- Register new apps in `LOCAL_APPS` in `config/settings/base.py` as `"<project_name>.<appname>"`
- Add dependencies with `uv add <package>`

---

## 2. Python Conventions

- PEP 8, 120-char line limit, double quotes, `isort`, f-strings
- Use built-in Django facilities before reaching for third-party libraries
- Prioritise security; use the ORM over raw SQL
- Use signals sparingly

### Exception handling
- **Never** use bare `except:` or `except Exception:` — catch the most specific exception you can genuinely handle
- If meaningful recovery is not possible, let exceptions propagate so failures are loud
- Use `logging` instead of `print()` for errors

---

## 3. User Model

- Login is **email-only** — no `username` field
- The model has a `name` CharField; no `first_name` or `last_name`
- Always reference via `get_user_model()` — never import the model directly
- New user-related profile models belong in `<project_name>/users/models.py`

---

## 4. Authentication

- Use **django-allauth** for all authentication. No custom auth backends.
- Social login providers: **Google** and **GitHub**
- Email/password is kept as a fallback only; allauth is configured to require email verification
- **User registration is disabled in production** (`ACCOUNT_ALLOW_REGISTRATION = False`) pending University Ethics approval. Do not re-enable without explicit instruction.

### User impersonation
- Use **django-hijack**; configure the hijack button in Django admin (user list and detail views)
- Only superusers and staff with explicit permission may hijack
- Show a visible warning banner for the duration of any hijacked session

### Consent middleware
- `ConsentRequiredMiddleware` blocks all authenticated views until the user accepts the current consent version
- To exempt a URL, add its `url_name` to the exempt list in the middleware (e.g. `users:deletion_complete`)
- For HTMX non-GET requests from non-consented users, respond with `HX-Redirect` to force a full-page reload so the consent modal appears

### Passkey (WebAuthn) setup

**Packages** (install with `uv add`):
- `django-allauth[mfa,webauthn]` — the `webauthn` extra pulls in `fido2`
- `webauthn>=2.7.1` — direct dependency required by allauth's WebAuthn implementation

**`INSTALLED_APPS`** (`base.py`):
```python
"allauth.account",
"allauth.mfa",
"allauth.mfa.webauthn",
```

**Settings by environment:**
```python
# base.py (all environments)
MFA_SUPPORTED_TYPES = ["webauthn", "recovery_codes"]
MFA_PASSKEY_LOGIN_ENABLED = True

# production.py only
MFA_PASSKEY_SIGNUP_ENABLED = True
ACCOUNT_EMAIL_VERIFICATION_BY_CODE_ENABLED = True

# local.py only — never in production
MFA_WEBAUTHN_ALLOW_INSECURE_ORIGIN = True  # allows passkey testing over HTTP
```

### Passkey UX preferences
- Passkey is the **primary** sign-in method. Email/password is secondary.
- The login template (`account/login.html`) overrides allauth's default and must:
  - Show a prominent "Sign in with passkey (fingerprint / device PIN)" button **first**
  - Hide the email/password form in a native `<details>`/`<summary>` collapsed by default
- **Don't hand-write these pages.** Adding `"allauth_bulma.pages"` to `INSTALLED_APPS` (see §8) ships `account/login.html` and `account/signup.html` already in this shape, with the conditional-UI script below already wired. Supply project copy by extending them and filling a block, rather than copying the template:

```django
{# <project_name>/templates/account/login.html #}
{% extends "allauth_bulma/login.html" %}
{% block login_intro %}<p class="subtitle is-6">Your strapline here.</p>{% endblock %}
```

- Use **WebAuthn Conditional UI** (`mediation: "conditional"`) so the browser surfaces stored passkeys in the email-field autofill without a button click. The package ships this as `allauth_bulma/passkey_conditional_ui.html`; the rules below are what it implements, and what to follow if you ever write it by hand:
  - On page load, check `PublicKeyCredential.isConditionalMediationAvailable()`
  - If available, set `autocomplete="username webauthn"` on the email input and start a background conditional credential request against allauth's `mfa_login_webauthn` endpoint
  - Use an `AbortController`; abort it in a **capture-phase** listener on the passkey button (allauth's handler runs in the bubble phase — this prevents "A request is already pending")
  - Swallow `AbortError` and `NotAllowedError` silently; log anything else
- The server cannot know if a visitor has a passkey before they identify — conditional UI is browser/OS-side. Do not attempt server-side passkey detection on the login page.

---

## 5. Guard email-sending endpoints against form-abuse / email bombing

**The risk.** Any unauthenticated endpoint that triggers an outbound email on submission – account signup with email verification, password reset, "resend confirmation," waitlist/contact forms – can be abused as a spam relay. An attacker scripts the form with arbitrary recipient addresses; your server emails strangers on their behalf, burning SMTP credits and damaging sender reputation (blocklisting). The victims are third parties, so it's easy to miss until the bill or a deliverability warning arrives.

> Real incident: a public allauth signup form with no CAPTCHA and an over-generous default rate limit was scripted by a bot with random third-party addresses, firing ~1,900 unsolicited "Confirm your email" messages, exhausting the Brevo SMTP credits and threatening sender reputation.

Apply **all** of these layered defenses – no single one is sufficient.

### 5.1 CAPTCHA on the form itself

The only thing that reliably stops distributed/slow bots (per-IP rate limits don't). Prefer a self-hosted CAPTCHA like **django-simple-captcha** when you can't or don't want to depend on a third party: it generates the challenge image locally, needs no API keys and no external calls, and serves the image from `'self'` so it works under a strict CSP (`img-src 'self'`).

**Wire it into the form – don't just install it:**

```python
# forms.py
from captcha.fields import CaptchaField
from allauth.account.forms import SignupForm

class UserSignupForm(SignupForm):
    captcha = CaptchaField()
```

Ensure the app is fully wired: `"captcha"` in `INSTALLED_APPS`, `path("captcha/", include("captcha.urls"))` in the URLconf, migrations applied, and Pillow installed. In tests, set `CAPTCHA_TEST_MODE = True` so the literal answer `"PASSED"` validates (submit `captcha_0=key`, `captcha_1="PASSED"`).

If you must use reCAPTCHA/hCaptcha/Turnstile instead, the rule is the same: attach the field to the form **and** add the provider's domains to your CSP `script-src`/`frame-src`. Installing the package alone does nothing.

### 5.2 Tighten framework rate limits – never trust defaults

With django-allauth, override `ACCOUNT_RATE_LIMITS`; the default `signup: "20/m/ip"` allows 20 confirmation emails/minute/IP. Use something like:

```python
ACCOUNT_RATE_LIMITS = {
    "signup": "5/h/ip",
    "login_failed": "10/m/ip,5/300s/key",
    "reset_password": "5/h/ip,3/h/key",
    "confirm_email": "1/3m/key",
}
```

For non-allauth views (custom contact/waitlist forms), use **django-ratelimit**: `@ratelimit(key="ip", rate="5/h", method="POST", block=False)` and render a 429 when `request.limited`.

### 5.3 Honeypot field

Add a hidden input real users never fill (zero user friction) to catch naive bots; reject the submission if it's non-empty.

### 5.4 Operational hygiene

- Keep `ACCOUNT_EMAIL_VERIFICATION = "mandatory"` so unverified accounts can't act.
- Monitor your ESP dashboard for sudden send spikes.
- After an incident, check sender reputation/blocklists.

**Anti-pattern to call out:** installing an anti-bot package (django-recaptcha, django-simple-captcha) and reading its keys in settings, but **never adding the field to the form** – this gives a false sense of security while the endpoint stays wide open.

**Verify with a test, don't just eyeball it.** Assert that (a) the form/page renders the CAPTCHA, (b) a POST without a valid CAPTCHA is rejected and sends zero emails (`len(mail.outbox) == 0`), and (c) a POST with a valid CAPTCHA succeeds and sends exactly one.

---

## 6. Models

- Always add `__str__`
- Use `related_name` where helpful
- `blank=True` for forms, `null=True` for DB nullability
- Always use migrations; optimise with `select_related`/`prefetch_related`; index frequent lookups

### Query efficiency
Always pick the most targeted ORM call for the job:

| Goal | Use | Avoid |
|------|-----|-------|
| Does any row exist? | `.exists()` | `.count()`, `len()`, `.all()`, `bool(qs)` |
| How many rows? | `.count()` | `len(qs)` |
| One specific object | `.get()` / `get_object_or_404()` | fetching a list then indexing |
| Only need a few fields | `.values()` / `.values_list()` / `.only()` | fetching full model objects |
| Related objects in a loop | `select_related()` (FK/OneToOne) or `prefetch_related()` (M2M/reverse FK) | per-iteration queries ("N+1") |
| Bulk create/update | `.bulk_create()` / `.bulk_update()` | looping `.save()` |
| Aggregate (sum, max…) | `.aggregate()` / `.annotate()` | fetching rows and computing in Python |

In **templates**, Django auto-calls callables — write `queryset.exists` (no parentheses) to get the same `SELECT 1 LIMIT 1` benefit.

---

## 7. Views & URLs

- Validate and sanitise all input
- Prefer `get_object_or_404`
- Paginate lists
- URL names should be descriptive and end with `/`

### Always have a logged-in home page

**Every project has a signed-in landing page, and signing in goes there.** Never drop a user on their profile/settings page, or back on the marketing site, after login — those are destinations, not starting points.

The page must:
- **Show the tools available to them** — the actions this user can actually take, as visible entry points (primary action prominent), not just navbar links they have to hunt for
- **Summarise key information where appropriate** — headline figures, recent items, anything needing attention. Skip this rather than pad it with numbers no one acts on
- **Reflect permissions** — surface only what this user can do; an empty state for a new user should point at the first useful action rather than showing zeros

Implement it as the site root, branching on authentication, so `/` is correct for everyone and there's no redirect hop:

```python
# config/views.py
def home(request):
    """Marketing landing when logged out; a tooled dashboard when logged in."""
    if not request.user.is_authenticated:
        return render(request, "pages/home.html")
    ...
    return render(request, "pages/dashboard.html", {"stats": stats, ...})
```

Then point allauth at it — cookiecutter-django defaults `LOGIN_REDIRECT_URL` to a `users:redirect` view that lands on the user's own detail page, which is **not** what we want:

```python
# config/settings/base.py
LOGIN_REDIRECT_URL = "home"
```

Keep the summary queries efficient (§6) — this page loads on every sign-in, so use `.count()`/`.aggregate()` and `annotate()` rather than pulling rows to count in Python.

---

## 8. Templates

- Templates live in the app that owns them: `<project_name>/<appname>/templates/<appname>/`
- Do **not** use a global `<project_name>/templates/` directory for app templates
- `base.html` lives at the project level; all app templates extend it
- HTMX partials: `<project_name>/<appname>/templates/<appname>/partials/_<name>.html`
- Use template inheritance; keep logic minimal; always `{% load static %}` and enable CSRF

### allauth template overrides

**Require [django-allauth-bulma](https://github.com/andytwoods/django-allauth-bulma)** — do not hand-copy Bulma allauth templates into a new project. allauth renders every page through a small set of "elements" (`button`, `field`, `form`, `panel`, …), so overriding those elements once styles *all* allauth pages: login, signup, email management, password change, TOTP setup, recovery codes, passkey management and social connections.

```bash
uv add git+https://github.com/andytwoods/django-allauth-bulma.git --tag v0.1.0
```

```python
# config/settings/base.py — ordering is load-bearing
THIRD_PARTY_APPS = [
    ...
    # The app-directories template loader is first-match-wins, so these must
    # precede `allauth` or its own templates win.
    "allauth_bulma",
    "allauth_bulma.pages",  # optional: passkey-first login/signup (see §4)
    "allauth",
    "allauth.account",
    "allauth.mfa",
    ...
]
```

If the project is built from a Dockerfile, the build stage needs `git` installed to resolve the git dependency (`python:*-slim` images don't ship it).

**Bridging into your `base.html`.** Out of the box the package renders a standalone shell. To fold allauth's pages into the project's own chrome, shadow `allauth_bulma/base.html` in `<project_name>/templates/`:

```django
{% extends "base.html" %}

{% block title %}{% block head_title %}{% endblock head_title %}{% endblock title %}
{% block main %}{% block allauth_bulma_main %}{% endblock allauth_bulma_main %}{% endblock main %}
{% block inline_javascript %}{% block extra_body %}{% endblock extra_body %}{% endblock inline_javascript %}
```

- `{% extends %}` must be the **first** tag — put any explanatory comment after it.
- Hook `allauth_bulma_main`, **never** `content`. The package's layouts declare `{% block content %}` *inside* that wrapper for allauth's own pages to fill; naming both `content` makes Django resolve them to the same block and the centred-column wrapper is silently dropped. This requires `base.html` to have a block that *wraps* its content block (`main`).

**What the package provides:** Bulma versions of `alert`, `badge`, `button`, `button_group`, `details`, `field`, `fields`, `form`, `h1`, `h2`, `hr`, `p`, `panel`, `provider`, `provider_list`, `table`; entrance (5-tablet/4-desktop) and manage (8-tablet/7-desktop) layouts; and `account/base_manage_password.html`. `button.html` maps allauth tags to Bulma classes: `danger`→`is-danger`, `success`→`is-success`, `warning`→`is-warning`, `secondary`→`is-light`, `outline`→`is-outlined`, `prominent`→`is-fullwidth`, default→`is-primary`.

**Project-specific overrides** still go in `<project_name>/templates/account/` or `<project_name>/templates/mfa/` — allauth picks them up automatically, and `TEMPLATES["DIRS"]` beats app directories, so they win over the package. Prefer extending the package's template and filling a block over copying it wholesale.

---

## 9. Forms

- Prefer ModelForms
- Use **crispy-forms** with the **Bulma** template pack (`crispy-bulma`). `CRISPY_TEMPLATE_PACK = "bulma"` in `base.py`.
- **Exception: allauth forms.** allauth renders whole forms through its `fields` element, which django-allauth-bulma (§8) implements with its own `{% bulma_field %}` tag rather than crispy — so the package carries no crispy dependency. Leave that element alone; crispy still applies to every non-allauth form.

---

## 10. Frontend

### CSS — Bulma
- **Bulma** is the only CSS framework. No Bootstrap, Tailwind, or others.
- Self-hosted at `<project_name>/static/css/bulma.min.css` (v1.0.3)
- Use Bulma classes in templates; write custom CSS only when Bulma has no suitable class
- Custom styles go in `<project_name>/static/css/project.css` (compiled from SCSS)

### Self-hosted static libraries
All JS/CSS dependencies are self-hosted in `<project_name>/static/`. Do not load them from CDNs. Exception: very large dependencies (e.g. **Pyodide**) may use a CDN when justified.

| File | Purpose |
|------|---------|
| `htmx.min.js` | AJAX navigation and partial updates |
| `sweetalert2.min.js` + `.min.css` | Modal dialogs and confirmations |
| `chart.umd.min.js` (v4.4.7) | Chart rendering |
| `bulma.min.css` (v1.0.3) | CSS framework |
| `fontawesome/css/all.min.css` | Icons |
| `project.js` / `project.css` | Project-wide behaviours and styles |

### JavaScript — modals and notifications
- Use **SweetAlert2** for modal dialogs/alerts/confirmations — never `alert()` or `confirm()`
- Use `window.showToast(message, type)` (defined in `project.js`) for transient client-side notifications
- Server-side Django messages are automatically rendered as toasts via `base.html`

### HTMX — when to use it
- Use **htmx** for all dynamic partial updates. No React, Vue, or other JS frameworks.
- **Same-endpoint pattern:** HTMX hits the same URL as the full page; the view detects HTMX via `request.headers.get("HX-Request")` and returns a partial template. Do **not** create separate URL routes for HTMX.
- Default to HTMX for: form submissions, consent/surveys/profile intake, session flow transitions, any server-rendered content swap

### HTMX — form validation responses
- Return **HTTP 422** with the partial form template when HTMX form validation fails. The project's HTMX config (`project.js`) swaps on 422. Do **not** return 200 for failed submissions.

### Vanilla JS — when to use it
Only for cases where HTMX genuinely does not apply:
- **Pyodide / code editor** — CodeMirror, Web Worker execution, timer, telemetry
- **Chart rendering** — Chart.js/Plotly.js consuming JSON from API endpoints

### JSON API endpoints
Acceptable only for:
- Aggregate chart data consumed by Chart.js/Plotly.js
- Pyodide test result submission (client posts JSON after local execution)

Do **not** create JSON endpoints for anything that could be an HTMX partial swap.

### Accessibility (a11y) widget
- A persistent accessibility panel is built into `base.html`, toggled via an "Accessibility" button in the navbar and footer. **Do not remove it.**
- Preferences are stored in `localStorage` under `a11yPrefs` and applied via `data-*` attributes on `<html>` before the DOM renders (prevents flash)
- The four axes:

| Axis | `data-*` attribute | Values |
|------|--------------------|--------|
| Theme | `data-theme` | `light` \| `dark` \| `system` |
| Contrast | `data-a11y-contrast` | `standard` \| `low` \| `high` |
| Text size | `data-a11y-text` | `standard` \| `large` \| `extra-large` |
| Motion | `data-a11y-motion` | `standard` \| `reduced` |

- Target `html[data-*]` selectors in `project.css` when writing preference-aware CSS (don't rely solely on `@media` queries)

### Radio button behaviour
Radio buttons are **deselectable** (clicking a selected radio unchecks it). This is intentional, handled in `project.js`. Do not override it.

---

## 11. Settings

- Use env vars; never commit secrets
- Settings split: `base.py` (all envs) / `local.py` (dev) / `production.py` (prod) — do not collapse them
- **Local dev:** SQLite database, console email backend
- **Production:** PostgreSQL, Brevo SMTP (`smtp-relay.brevo.com:587`) via `django-anymail`
- **Error tracking:** Rollbar in production (`django-rollbar`, `ROLLBAR_ACCESS_TOKEN` env var). Not in dev.

### Local SQLite override

cookiecutter-django defaults `DATABASE_URL` to Postgres in `base.py`. For local dev, override in `local.py`:

```python
from .base import BASE_DIR

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": str(BASE_DIR / "db.sqlite3"),
        "ATOMIC_REQUESTS": True,
    },
}
```

### Required fix: cookiecutter-django sites migration on SQLite

cookiecutter-django officially supports **Postgres only**, so its generated
`<project>/contrib/sites/migrations/0003_set_site_domain_and_name.py` runs
Postgres-specific sequence SQL (`SELECT last_value from django_site_id_seq` /
`alter sequence …`). On SQLite this fails with
`OperationalError: no such table: django_site_id_seq`. Upstream has declined to
fix this ([issue #3587](https://github.com/cookiecutter/cookiecutter-django/issues/3587),
wontfix), so **every new project must patch this migration** — guard the
sequence block by backend vendor:

```python
# in _update_or_create_site_with_sequence(...)
if created and connection.vendor == "postgresql":
    # SQLite (local dev) has no sequences, so this is skipped.
    max_id = site_model.objects.order_by("-id").first().id
    with connection.cursor() as cursor:
        cursor.execute("SELECT last_value from django_site_id_seq")
        ...
```

The guard is a no-op on Postgres (preserves the sequence sync) and lets SQLite
migrate cleanly.

### Local development tooling (dev-only)
- Use **django-debug-toolbar** in `local.py` only — never in production.
- **Require [django-loginout-panel](https://github.com/andytwoods/django-loginout-panel)** — a debug-toolbar panel giving one-click login/logout while developing. Install as a dev dependency and wire it into `local.py` only:

```bash
uv add --dev django-loginout-panel
```

```python
# config/settings/local.py — inside the existing debug-toolbar block
INSTALLED_APPS += ["loginout_panel"]

# Defining DEBUG_TOOLBAR_PANELS is required for the panel to appear;
# list loginout_panel's panel first, then the standard toolbar panels.
DEBUG_TOOLBAR_PANELS = [
    "loginout_panel.panel.LoginOutPanel",  # NB: the class lives in .panel, not the package root
    # ... standard debug_toolbar.panels.* entries ...
]

# The account the panel logs in as. Keep it overridable via env.
LOGINOUT_USERNAME = env("LOGINOUT_USERNAME", default="dev@example.com")
```

- Optional hardening settings: `LOGINOUT_SERVER` (restrict endpoints to an IP) and `LOGINOUT_TRUST_XFF` (only when behind a trusted proxy).
- No models/migrations. Because login is **email-only** (§3), set `LOGINOUT_USERNAME` to a real local user's email.

---

## 12. Don't double-report errors – disable Django's mail_admins emails when you use an error tracker (Rollbar/Sentry)

If you run an error tracker (Rollbar, Sentry, etc. – this project uses Rollbar in production, see section 11), Django's built-in admin error emails are redundant noise. Turning them off is less obvious than it looks because of a logging gotcha.

**The gotcha.** Django's default logging attaches a `mail_admins` handler (`django.utils.log.AdminEmailHandler`) to the built-in `django` logger, and emails `settings.ADMINS` on ERROR-level `django.request` / `django.security.*` records (gated by the `require_debug_false` filter, so it only fires when `DEBUG=False`). Defining your own `LOGGING` dict does **not** remove this if you set `disable_existing_loggers: False` (the common default): the built-in `django` logger survives with `mail_admins` still attached. Worse, if your custom child loggers (`django.request`, etc.) use `propagate: True`, their records bubble up to that surviving parent and trigger the email anyway — even though `mail_admins` appears nowhere in your config.

**The fix.** Redefine the parent `django` logger explicitly with only the handlers you want, and stop children propagating into stray handlers:

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "handlers": {"console": {"class": "logging.StreamHandler", ...}},
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        # Overrides Django's default `django` logger, dropping mail_admins.
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "django.request": {"handlers": ["console"], "level": "ERROR", "propagate": False},
        "django.security.DisallowedHost": {"handlers": ["console"], "level": "ERROR", "propagate": False},
    },
}
```

**Keep `ADMINS` set.** Don't blank `ADMINS` to silence emails — it's also used by deliberate `django.core.mail.mail_admins()` / `mail_managers()` calls (e.g. signup notifications) and is a common fallback for a contact address. Removing the `mail_admins` logging handler is the correct, targeted fix; the `mail_admins()` function sends directly and is unaffected.

**Verify with a test**, since the behavior is config-driven and easy to get wrong:

```python
def test_request_errors_do_not_email_admins(settings):
    from django.core import mail; import logging
    mail.outbox.clear()
    logging.getLogger("django.request").error("boom")
    logging.getLogger("django.security.DisallowedHost").error("bad host")
    assert mail.outbox == []
```

(Run with `DEBUG=False` and the locmem email backend so the `require_debug_false` filter is active.)

**Anti-pattern:** assuming "I never added `mail_admins`, so no error emails are sent." With `disable_existing_loggers: False`, the default handler persists invisibly.

---

## 13. Background Tasks

- Use **Huey** with a Redis backend. **Never Celery.**
- `tasks.py` contains only task-decorated functions — no business logic
- Task functions: accept/validate raw input, call helpers, handle only queue concerns (scheduling, retries)
- All business logic lives in `helpers/` modules (e.g. `helpers/task_helpers.py`) — reusable by views, management commands, and tasks alike
- Write helpers to be idempotent where possible (safe for retries)

---

## 14. New Project Setup

When setting up a new project, install and configure [code-review-graph](https://code-review-graph.com/) — a Python tool that builds a structural knowledge graph of the codebase so AI assistants read only what matters (8× token reduction):

```bash
uv add code-review-graph                          # add to project dependencies
uv run code-review-graph install --platform claude-code  # writes MCP config, hooks, and CLAUDE.md instructions
uv run code-review-graph build                    # parse the codebase (first run ~10s for 500-file project)
```

This only needs to be run once per project. Commit the resulting `.claude/settings.json` and `.mcp.json` (shared config) but add `.claude/settings.local.json` to `.gitignore`.

The graph auto-updates on every git commit via the installed pre-commit hook. Restart Claude Code after setup to pick up the new MCP config.

---

## 15. Phased Work (TASKS.md / ACTIONS.md)

When implementing work broken into phases in a `TASKS.md`, `ACTIONS.md`, or similar file:
- After completing each phase, commit all changes to git before starting the next phase
- Write a commit message that gives a clear synopsis of what the phase accomplished — not just "phase 2 done" but a meaningful summary of the actual changes (e.g. "Add WebAuthn passkey login with conditional UI and abort-controller fix")
- Use the standard commit format: short imperative subject line, then a blank line and bullet points for detail if the phase was substantial

## 16. Testing

- Write unit tests for all new features; cover success and failure paths
- Never hard-code URL paths in assertions — use `django.urls.reverse()` with named routes
- For redirect assertions that depend on the current URL, use `request.path` or `request.get_full_path()`
