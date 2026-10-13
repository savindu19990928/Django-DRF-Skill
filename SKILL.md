---
name: django-pro
description: >-
  Expert guidance for building, reviewing, upgrading, and securing Django 6.x web apps and Django REST Framework (DRF 3.18+) APIs: models and ORM optimization, views and async, serializers and viewsets, authentication and permissions, background tasks (Django Tasks or Celery), Channels, testing, deployment, and security hardening. Use this skill whenever the user mentions Django, DRF, Django REST framework, settings.py, models.py, serializers, viewsets, migrations, the Django admin, ASGI/WSGI deployment, or asks to fix N+1 queries, secure an API, run a Django security review, or upgrade Django, even if they never say "Django" but are clearly working in a Django codebase. Use proactively for any Python web backend task in a Django project.
metadata:
  model: opus
---

# Django & DRF Pro: modern, stable, secure by default

Build and review Django and Django REST Framework code that is idiomatic, fast, testable, and secure by default. Prefer Django's built-in features before adding packages, because every dependency is extra attack surface and extra upgrade work. Treat security as part of the design of every model, serializer, view, and setting, not a pass at the end.

## Version baseline (verified October 2026)

| Component | Current | Notes |
|---|---|---|
| Django 6.1 | 6.1.2 (Oct 6, 2026) | Default target for new projects. Python 3.12-3.14. Mainstream support to Apr 2027, security fixes to Dec 2027. |
| Django 5.2 LTS | 5.2.18 | Choose when the team needs long support or cannot move to Python 3.12. Security fixes to Apr 2028. |
| Django 6.0 | 6.0.9 | Security-only until Apr 2027. Recommend moving to 6.1. |
| Django 6.2 LTS | expected Apr 2027 | Last release under the old versioning policy. Afterward releases are calendar-versioned (2028 in Jan 2028), each supported for three years. |
| Django 5.1 and older, 4.2 LTS | unsupported | No security fixes. Flag any project on these and propose an upgrade path. |
| DRF 3.18.0 (Aug 7, 2026) | current | Supports Django 5.2, 6.0, 6.1. Dropped Django 4.2/5.0/5.1. Python 3.10+. |

Versions drift, so when you have web access verify against djangoproject.com/download and pypi.org/project/djangorestframework before claiming "latest". Without web access, state this baseline as of October 2026. Always recommend the newest patch release of the chosen series: patch releases are fully compatible and carry the security fixes (6.1.2 alone fixed several moderate issues, including a header-parsing denial of service and privilege abuse in model formsets with editable primary keys).

## Working method

1. **Establish context.** Check `pyproject.toml` or requirements for Django, DRF, and Python versions, plus database, deployment target, who consumes the API (browser SPA, mobile app, third parties), tenancy model, and data sensitivity. If unknown, assume Django 6.1, DRF 3.18, Python 3.13, PostgreSQL 15+, and say so in one line.
2. **Pick the Django-idiomatic option first** (built-in tasks, CSP, caching, auth, admin) and reach for third-party packages only when the built-in falls short.
3. **Write code, tests, and security controls together.** Include negative tests (unauthenticated, wrong user, malformed input) alongside happy-path tests.
4. **Explain trade-offs briefly.** Mention query-count and migration implications for ORM or schema changes, and the security reasoning behind non-obvious choices.
5. **Verify** (suggest or run):
   ```bash
   python -Wd manage.py test            # or: pytest -q  (keep deprecation warnings visible)
   python manage.py makemigrations --check --dry-run
   python manage.py check --deploy --fail-level WARNING --settings=config.settings.production
   pip-audit                            # known-vulnerable dependencies
   ruff check --select S                # bandit-style security lint rules
   ```
   In Django 6.1, `check` runs against all configured databases when none is specified, so CI needs database access (or pass `--database`).

## What is new in Django 6.x worth using

**Django 6.1**
- **Fetch modes** for on-demand field loading: `Book.objects.fetch_mode(models.FETCH_PEERS)` behaves like an automatic `prefetch_related` and collapses most N+1 patterns to two queries. `FETCH_RAISE` raises `FieldFetchBlocked`, which is useful in hot paths and tests to catch accidental queries. `FETCH_ONE` is the default.
- **Database-level deletes**: `on_delete=DB_CASCADE`, `DB_SET_NULL`, `DB_SET_DEFAULT` run in SQL without loading objects. They skip `pre_delete`/`post_delete` signals, so do not use them where signal handlers do cleanup or auditing.
- **`MAILERS` setting** replaces `EMAIL_BACKEND` and the `EMAIL_*` settings, with a `using=` argument on send functions. `EMAIL_*`, `get_connection()`, `fail_silently`, and `connection=` are deprecated, so write new code against `MAILERS`.
- `UUID4` and `UUID7` database functions, `JSONNull` for explicit JSON null, `QuerySet.totally_ordered`, and `BitAnd`/`BitOr`/`BitXor` available outside `contrib.postgres`.
- **CSP helpers**: the `csp_nonce_attr` template tag, and nonces applied automatically in admin and built-in templates when the `csp` context processor is configured (system check `security.W027` warns if it is missing).
- **Security defaults**: PBKDF2 iterations raised to 1,500,000 (existing hashes upgrade on next login); signed cookies use a new salt derivation (`SIGNED_COOKIE_LEGACY_SALT_FALLBACK` exists only as a temporary migration aid); the default `algorithm` of `salted_hmac()` and `base64_hmac()` will change from sha1 to sha256, so pass `algorithm="sha256"` explicitly.
- **Deprecations to avoid in new code**: `select_related()` with no arguments, `values_list(flat=True)` without a field name, `transaction.savepoint()` (use `savepoint_create()`), and double-dot template lookups like `{{ a..b }}`. Warning classes are renamed `RemovedInDjango2028Warning`.
- **Dropped support**: PostgreSQL 14, MySQL below 8.4, MariaDB below 10.11, SQLite below 3.37.

**Introduced in Django 6.0 and still current**
- **Built-in background Tasks** (`django.tasks`, `@task`, `TASKS` setting). Use for simple fire-and-forget work, and check the docs for which production backends exist. Keep Celery for retries, schedules, workflows, and rate limits.
- **Built-in Content Security Policy** via `ContentSecurityPolicyMiddleware` and the `SECURE_CSP` / `SECURE_CSP_REPORT_ONLY` settings (see Security).
- **Template partials** (`{% partialdef %}` / `{% partial %}`), which pair well with HTMX.

## Architecture and ORM

- Keep apps small and cohesive. Put business rules in service functions or domain methods, not in views, serializers, or signals. Use signals sparingly, because they hide control flow.
- Start every project with a **custom user model** (`AbstractUser`) and explicit `DEFAULT_AUTO_FIELD`. Changing either later is painful.
- Use `select_related` for single-valued relations and `prefetch_related` (with `Prefetch` and filtered querysets) for multi-valued ones, or `FETCH_PEERS` when listing is dynamic. Use `only()`/`defer()`/`values()` to cut payloads, `annotate`/`Subquery`/`Exists` instead of Python loops, `bulk_create`/`bulk_update` for batches, and `F()` expressions for atomic updates.
- Index what you filter, order, and join on (`Meta.indexes`, `UniqueConstraint`, `CheckConstraint`, partial and covering indexes on PostgreSQL). Enforce invariants in the database as well as in validators.
- Wrap multi-step writes in `transaction.atomic()` and use `select_for_update()` for contended rows. Schedule side effects (emails, tasks) with `transaction.on_commit()` so they never fire for rolled-back work.
- Make migrations safe for zero-downtime deploys: add nullable columns first, backfill in batches in a separate data migration, then add constraints; avoid long table locks (use `AddIndexConcurrently` on PostgreSQL). Never edit applied migrations.
- PostgreSQL connection pooling: use Django's native pool (`OPTIONS: {"pool": True}`) or PgBouncer, and read the docs before combining either with persistent `CONN_MAX_AGE`.
- Public identifiers: prefer UUIDs (`UUID7` for index locality) to reduce enumeration, but never treat unguessable IDs as authorization.

## Async and background work

- Use async views, middleware, and async ORM methods (`aget`, `acreate`, `async for`) for I/O-bound endpoints under ASGI (Uvicorn, Granian, Daphne, or Hypercorn behind a reverse proxy). Do not call sync ORM code inside async code without `sync_to_async`, and do not block the event loop with sync HTTP calls.
- DRF views are synchronous by default, so under ASGI they run in a thread. Verify the current DRF async story before promising async DRF views; `adrf` is a third-party option.
- Use Django Channels for WebSockets. Authenticate in the connection handshake (`AuthMiddlewareStack` or token middleware), check `Origin` (`AllowedHostsOriginValidator`), authorize each group subscription, and rate-limit messages.
- Celery best practices: pass IDs rather than objects, make tasks idempotent, set time limits and retry with backoff, use `acks_late` deliberately, never trust task arguments from untrusted sources, and never use pickle serialization (stay on JSON).

## Django REST Framework

Structure: thin viewsets, explicit serializers, queryset scoping in `get_queryset()`, permissions as classes, business logic in services.

```python
class InvoiceWriteSerializer(serializers.ModelSerializer):
    customer = serializers.PrimaryKeyRelatedField(queryset=Customer.objects.none())

    class Meta:
        model = Invoice
        fields = ["customer", "amount", "due_date", "notes"]   # explicit allowlist
        # owner, status, is_paid are set server-side and never accepted from clients

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        request = self.context.get("request")
        if request and request.user.is_authenticated:
            # scope related objects to the caller, or clients can link to other tenants' rows
            self.fields["customer"].queryset = Customer.objects.filter(owner=request.user)


class InvoiceViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "invoices"

    def get_queryset(self):
        return Invoice.objects.filter(owner=self.request.user).select_related("customer")

    def get_serializer_class(self):
        if self.action in {"create", "update", "partial_update"}:
            return InvoiceWriteSerializer
        return InvoiceReadSerializer

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

- Use separate read and write serializers when the shapes differ. Keep nested writes explicit (override `create`/`update`) rather than relying on implicit behavior.
- DRF 3.18 changes to know: errors for list serializers (`many=True`) now use a dict format (a breaking change for clients that parse errors), `@throttle_scope` exists for function-based views, `UniqueTogetherValidator` supports `nulls_distinct`, and `SearchFilter` supports `unaccent`. Django 4.2, 5.0, and 5.1 are no longer supported.
- Paginate every list endpoint with a capped page size. Use cursor pagination for large or fast-growing tables.
- Document the API with an OpenAPI generator (drf-spectacular is the common choice) and keep schema endpoints behind authentication for private APIs.

## Security (apply by default, mention when relevant)

Frame reviews around the OWASP Top 10 and the OWASP API Security Top 10: broken access control (including object-level authorization), misconfiguration, injection, authentication failures, supply-chain and integrity failures, and logging gaps.

### Production settings baseline

```python
import os
from django.utils.csp import CSP

DEBUG = False
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]            # never committed, never shared across environments
SECRET_KEY_FALLBACKS = []                               # only during rotation; remove old keys promptly
ALLOWED_HOSTS = ["app.example.com"]                     # never ["*"] without your own Host validation
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]

SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000                          # ramp up from a small value first
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True                              # only once every subdomain is HTTPS-only
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # only if the proxy sets and strips this header
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"
SECURE_REFERRER_POLICY = "same-origin"
X_FRAME_OPTIONS = "DENY"

DATA_UPLOAD_MAX_MEMORY_SIZE = 5 * 1024 * 1024           # cap request bodies and field counts
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1000

PASSWORD_HASHERS = ["django.contrib.auth.hashers.Argon2PasswordHasher",   # pip install "django[argon2]"
                    "django.contrib.auth.hashers.PBKDF2PasswordHasher"]
# Keep all four AUTH_PASSWORD_VALIDATORS enabled.

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",     # first
    # ... sessions, common, csrf, auth, messages, clickjacking ...
    "django.middleware.csp.ContentSecurityPolicyMiddleware",
]
SECURE_CSP = {                                          # start with SECURE_CSP_REPORT_ONLY, then enforce
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],                # add django.template.context_processors.csp to TEMPLATES
    "object-src": [CSP.NONE],
    "base-uri": [CSP.NONE],
    "frame-ancestors": [CSP.NONE],
}
```

Load secrets from environment variables or a secret manager, add secret scanning (gitleaks or detect-secrets) to pre-commit and CI, and run `manage.py check --deploy` in CI so misconfiguration fails the build.

### Authentication and sessions
- Enforce strong password policy and rate-limit login, password reset, and OTP endpoints (django-axes or DRF scoped throttles). Add MFA for staff and sensitive roles (django-allauth or django-otp).
- Keep `django.contrib.auth.login()` for session rotation on login. Invalidate sessions on password change. Use generic error messages that do not reveal whether an account exists.
- Restrict the admin to a VPN or IP allowlist and require MFA. Renaming `/admin/` is optional and not a control on its own. Give staff least-privilege permissions.

### Access control (the most common real-world failure)
- Deny by default. Scope every queryset to what the caller may see, because object-level permission checks only run in `get_object()` and never filter list results.
- Check authorization on every related object a client can reference by ID (foreign keys in request bodies, nested routes, bulk endpoints).
- In multi-tenant apps, scope through a tenant-aware manager, consider PostgreSQL row-level security, and write cross-tenant tests.

### Injection, XSS, CSRF, redirects, SSRF
- Use the ORM or parameterized queries (`raw("... %s", [value])`, `cursor.execute(sql, params)`). Never build SQL with f-strings, and treat `extra()` and `RawSQL` as high risk.
- Rely on template autoescaping. Never call `mark_safe` or `|safe` on user input; use `format_html` and `json_script`. Sanitize rich text allowlist-style with `nh3`.
- Keep `CsrfViewMiddleware` on and avoid blanket `csrf_exempt`. Session-authenticated DRF endpoints are CSRF-protected; if you store JWTs in cookies, CSRF applies to them too.
- Validate redirect targets with `url_has_allowed_host_and_scheme`. When the server fetches user-supplied URLs, use allowlists, block private and link-local IP ranges, set timeouts, and disable redirects.
- Never use `pickle`, `yaml.load`, `eval`, or `subprocess(..., shell=True)` on untrusted data, and never pass `verify=False` to HTTP clients.

### File uploads
- Validate size, extension, and actual content (`ImageField` runs Pillow checks). Generate random filenames and store media outside the code path on object storage (django-storages).
- Never let the web server execute uploads. Serve user files from a separate domain or with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`, and treat SVG as active content.

### DRF-specific hardening
```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": ["rest_framework_simplejwt.authentication.JWTAuthentication"],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],  # DRF's own default is AllowAny
    "DEFAULT_RENDERER_CLASSES": ["rest_framework.renderers.JSONRenderer"],         # no BrowsableAPIRenderer in production
    "DEFAULT_PARSER_CLASSES": ["rest_framework.parsers.JSONParser"],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
        "rest_framework.throttling.ScopedRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {"anon": "30/min", "user": "300/min", "login": "5/min", "invoices": "120/min"},
    "DEFAULT_PAGINATION_CLASS": "config.api.pagination.CappedPagination",  # sets max_page_size / max_limit
    "PAGE_SIZE": 25,
    "EXCEPTION_HANDLER": "config.api.exceptions.handler",    # consistent errors, no internals leaked
    "NUM_PROXIES": 1,                                        # so throttling sees the real client IP behind a proxy
}
```
- **Default deny**: set `DEFAULT_PERMISSION_CLASSES` to `IsAuthenticated`, and opt out explicitly with `AllowAny` on public endpoints. Add role or object permissions as small, tested permission classes.
- **No mass assignment**: list serializer `fields` explicitly. Never use `fields = "__all__"` or `exclude`, since new model columns leak or become writable silently. Set ownership and privileged fields in `perform_create`/`perform_update`, and use `read_only_fields` and `write_only=True` (passwords, secrets).
- **Throttle with a shared cache** (Redis or Memcached). Per-process local-memory caches make limits meaningless across workers. Use scoped throttles for login, registration, password reset, and expensive endpoints.
- **Whitelist filtering and ordering**: set `filterset_fields`, `search_fields`, and `ordering_fields` deliberately, because ordering and filtering on sensitive fields leaks their values through side channels.
- **Tokens**: prefer short-lived access tokens (about 5-15 minutes) with rotating refresh tokens and blacklisting (`ROTATE_REFRESH_TOKENS`, `BLACKLIST_AFTER_ROTATION`, `rest_framework_simplejwt.token_blacklist`), a signing key separate from `SECRET_KEY`, and an explicit `ALGORITHM`. For browsers, prefer `HttpOnly` Secure SameSite cookies over `localStorage`. DRF's built-in `TokenAuthentication` tokens never expire, so avoid it for sensitive APIs. Use OAuth2/OIDC (django-oauth-toolkit, allauth) for third-party access.
- **CORS**: use django-cors-headers with an explicit origin allowlist. Never combine allow-all origins with credentials.
- **Webhooks**: verify signatures with `hmac.compare_digest`, check timestamps against replay, and respond quickly while processing in a task.
- **Limit bulk and nested work**: cap list sizes (`ListField(max_length=...)`), nesting depth, and bulk endpoint batch sizes.

### Data protection, logging, and supply chain
- Use TLS to the database (`sslmode=verify-full`), a least-privilege runtime DB role distinct from the migration role, and encrypted backups. Encrypt highly sensitive fields at the application level when needed, and keep keys outside the database.
- Log authentication failures, permission denials, and admin actions through the `django.security` logger. Never log tokens, passwords, or full request bodies. Use `@sensitive_variables` and `@sensitive_post_parameters`, and configure error reporting such as Sentry with `send_default_pii=False`.
- Lock dependencies with hashes (`uv lock` or `pip-compile --generate-hashes`), run `pip-audit` and Dependabot or Renovate, apply Django and DRF security patches promptly, and wait a short cooling-off period before adopting brand-new releases of third-party packages. Vet new packages for maintenance, ownership, and necessity.
- Containers: multi-stage builds, slim base image, non-root user, no secrets baked into layers.

### Security red flags to scan for in reviews
`fields = "__all__"`; missing or `AllowAny` permission classes on sensitive endpoints; viewsets with `queryset = Model.objects.all()` and no owner scoping; `Model.objects.get(pk=request.data[...])` without scoping; f-string SQL; `mark_safe`/`|safe` on user data; `csrf_exempt`; `DEBUG = True` or `ALLOWED_HOSTS = ["*"]` in production settings; `CORS_ALLOW_ALL_ORIGINS = True`; secrets in the repo; JWTs in `localStorage`; unvalidated redirect parameters; unbounded pagination or uploads; `BrowsableAPIRenderer` enabled in production; `pickle`/`yaml.load`/`eval`; `verify=False`.

## Testing and quality

- Use pytest-django with factory_boy, and `APIClient` with `force_authenticate` for DRF. Keep `TestCase` for transactional behavior and `TransactionTestCase` when testing `on_commit` or concurrency.
- Assert query counts (`django_assert_num_queries`) on list endpoints so N+1 regressions fail fast.
- Write a permission matrix test (parametrized over roles and endpoints) that asserts 401 for anonymous, 403 or 404 for other users' objects, and rejection of privileged fields in write payloads. Add throttling and malformed-input tests.
- Type-hint and docstring public interfaces, follow PEP 8 and Django coding style, lint with ruff, and measure coverage on critical paths rather than chasing a percentage.

## Deployment and operations

- Serve with Gunicorn (WSGI) or Uvicorn, Granian, or Daphne (ASGI) behind nginx or a cloud load balancer that terminates TLS, validates `Host`, and limits request size. Serve static files with WhiteNoise or a CDN and run `collectstatic` at build time.
- Use Redis or Memcached for cache and sessions (never the default local-memory cache across multiple processes), and set `LOGGING` explicitly with structured JSON logs.
- CI pipeline order: lint, type check, tests, migration check, `check --deploy`, `pip-audit`, container build and scan, then deploy with automatic migration gating and a rollback plan.
- Health checks, graceful shutdown, and observability (Sentry, OpenTelemetry, metrics) are part of "done" for a production service.

## Upgrading existing projects

1. Move one feature release at a time (for example 4.2 to 5.2 LTS to 6.1) and read each release's notes, since deprecations are removed two releases after they start warning.
2. Run the test suite with deprecation warnings visible, fix `RemovedInDjango...Warning` hits, then bump.
3. Upgrade DRF in step: DRF 3.18 requires Django 5.2 or newer, so older Django versions must move first.
4. After upgrading, re-run `check --deploy`, `makemigrations --check`, and `pip-audit`, and review the new security-relevant settings listed above.

## Response format

Lead with the solution, then explain trade-offs. Provide production-ready code with error handling, tests, and a migration or rollout note when schema or settings change. When the task touches authentication, data access, file handling, or settings, add a short "Security notes" list covering what you hardened and what remains for the user to decide. Keep answers proportional: a one-line ORM question deserves a short answer, not the full checklist.

## Example requests

- "Optimize this queryset that causes N+1 queries" (consider `FETCH_PEERS`, `select_related`, `prefetch_related`, and a query-count test)
- "Design a multi-tenant SaaS architecture with Django and DRF" (tenant scoping, permissions, indexes, isolation tests)
- "Review this DRF API for security issues" (run the red-flag scan, then settings, permissions, serializers, throttling)
- "Implement JWT authentication with refresh tokens in DRF" (short-lived tokens, rotation, blacklist, cookie vs header trade-offs)
- "Upgrade this project from Django 4.2 to 6.1" (staged path, deprecations, DRF compatibility)
- "Set up Django Channels for real-time notifications" (handshake auth, origin validation, group authorization)
- "Create a robust background task system" (Django Tasks versus Celery, idempotency, retries, `on_commit`)
