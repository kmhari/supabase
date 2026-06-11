# Supastack Studio Patches

This branch (`supastack-studio`) is what [Supastack](https://github.com/kmhari/selfbase) builds its
shared platform Studio image from (`supastack/studio-platform`, `IS_PLATFORM=true`). It is
**upstream `supabase/supabase` `master` + the patches listed below, nothing else**.

**Policy: smallest possible diff against upstream.** Every patch must be listed here with its
reasoning. Anything solvable in the Supastack layer instead (build args, entrypoint, reverse proxy,
API) is done there — the image build wrapper (placeholder-apex substitution, healthcheck override,
`NEXT_PUBLIC_*` selection) modifies **zero** files in this repo.

**Verify this list yourself:**

```sh
git fetch origin master
git log --oneline origin/master..supastack-studio     # the patch commits
git diff origin/master...supastack-studio --stat      # every touched file
```

**Maintenance:** the branch is kept current by fast-forwarding `master` from upstream and rebasing
this branch onto it (Supastack's `scripts/sync-studio-fork.sh`; pushed with `--force-with-lease`).
Commit shas change on every rebase — patches are identified by title. Adding a patch = commit on
this branch **+ an entry in this file**.

---

## 1. Email-only sign-in

- **Commit title**: `patch(supastack): email-only sign-in (disable dashboard_auth github/sso/signup/testimonial/tos)`
- **Files**: `packages/common/enabled-features/enabled-features.json` (5 boolean flips)
- **Since**: 2026-06-02

Flips five feature flags `true` → `false`:

| Flag | Why |
|---|---|
| `dashboard_auth:sign_up` | Supastack creates dashboard users via `/setup` and org invites — open self-signup on a self-hosted operator console is an account-creation hole |
| `dashboard_auth:sign_in_with_github` | No GitHub IdP configured on the control-plane GoTrue; the button would dead-end |
| `dashboard_auth:sign_in_with_sso` | Same — no SSO IdP |
| `dashboard_auth:show_testimonial` | Cloud marketing copy; not applicable self-hosted |
| `dashboard_auth:show_tos` | supabase.com Terms links; not applicable self-hosted |

**Why a source patch**: `enabled-features.json` is compiled into the bundle; there is no runtime
override for these flags in self-hosted platform mode.

**If removed**: sign-up + GitHub/SSO buttons reappear on `/dashboard/sign-in`; GitHub/SSO error
against the control-plane GoTrue, and sign-up mints unmanaged accounts.

## 2. hCaptcha only when a site key is configured

- **Commit title**: `patch(supastack): render hCaptcha only when NEXT_PUBLIC_HCAPTCHA_SITE_KEY is set`
- **Files**: `apps/studio/components/interfaces/SignIn/SignInForm.tsx`,
  `apps/studio/components/interfaces/SignIn/ForgotPasswordWizard.tsx`
  (wrap `<HCaptcha>` in a key-presence conditional)
- **Since**: 2026-06-11

Upstream renders `<HCaptcha sitekey={process.env.NEXT_PUBLIC_HCAPTCHA_SITE_KEY!} …>`
unconditionally — the `!` assumes Cloud always configures a key. Supastack's hermetic image build
bakes no key (and excludes `.env` files), so the widget initialized with `undefined` and **blocked
sign-in** with "hCaptcha has failed to initialize".

The patch renders the widget only when a key was baked at build time. The skip path is upstream's
own code: `captchaRef.current?.execute()` optional-chains to a no-op and `signInWithPassword`
sends `captchaToken: undefined`, which GoTrue accepts when captcha verification is disabled.

**Alternative rejected**: baking hCaptcha's universal test key — it loads `js.hcaptcha.com` on
every login for a check that always passes: an external dependency for theater. Captcha can be
re-enabled by baking a real `NEXT_PUBLIC_HCAPTCHA_SITE_KEY` into the image build *and* enabling
captcha verification on the GoTrue side.

**If removed**: sign-in breaks on any build without a site key.
