# Package Selection & Installation Protocol (personal working checklist)

Purpose: a fast, repeatable check to run *before* a new Composer/npm package lands in a PR — covering what Snyk and a reviewer typically **won't** catch on their own (license, maintenance health, whether it was even the right choice), not duplicating what they already do (known-CVE scanning in the pinned version).

Context this is written for: most feature work here is built on top of packages rather than hand-rolled code, so this needs to stay a 5–10 minute check, not a bureaucratic gate — otherwise it just gets skipped.

---

## 1. Before adding — evaluate the candidate

- **Real need?** Does Laravel itself, or something already installed, already solve this? Adding a package should replace non-trivial effort, not save 20 lines of code.
- **License.** MIT / BSD / Apache-2.0 → fine. **GPL/LGPL** → generally fine for server-side-only use (nothing is being distributed), but flag it for a second look. **AGPL** → stop and check before installing — AGPL's network-use clause can create source-disclosure obligations for a SaaS product like this one, which MIT/BSD don't.
- **Maintenance health** (Packagist/GitHub page, 2 minutes):
  - Last release within ~12 months?
  - Repo not archived, issues getting responses (even "not planned" is fine — silence isn't)?
  - More than one maintainer, or a single-person project with no activity in 6+ months? (bus-factor risk)
  - Stars/downloads are a sanity check only, not proof of quality.
- **Security history.** Check Packagist advisories / GitHub Security Advisories for this package *before* installing — not just wait for Snyk to flag it after the fact. Look specifically at *how* past CVEs were handled (patched fast vs. ignored for months) — that tells you more than whether one existed at all.
- **Laravel-specific wrapper packages.** Prefer wrappers maintained by known, active Laravel-ecosystem authors (e.g. spatie, mews) over a hand-rolled service provider around a raw PHP library. But treat the wrapper as its own dependency with its own risk — run this same checklist on it too, don't wave it through just because the underlying library looks fine.
- **Footprint.** Run `composer show -t <package>` (or `composer why-not` when comparing) to see what it pulls in transitively. A single-purpose package that suddenly drags in a large dependency tree is a signal to look closer, not necessarily a blocker.

## 2. Installing (mechanics)

- Pin a sensible version constraint — never `"*"`, and don't blindly accept whatever `composer require` proposes without glancing at it.
- Run `composer audit` **locally, before pushing** — don't wait for CI/Snyk to be the first check.
- Skim the package's own `composer.json` for install-time scripts/hooks — a package that runs arbitrary code on `composer install` is a supply-chain risk worth noticing.
- Check what config/migration files it publishes, and read the *defaults* — e.g. does a sanitizer's default config actually strip what you think it strips, or does it allow risky tags/attributes out of the box?
- Commit `composer.lock` together with `composer.json` in the same PR — never let them drift apart.

## 3. PR / review — what Snyk + reviewer catch, and what they don't

Snyk and `composer audit` catch: **known CVEs in the exact version you pinned.**

They do **not** catch: license problems, whether the package is actually still maintained, whether the config you're shipping with it is safe by default, or whether a lighter/better-maintained alternative existed. That's on this checklist, not the automated gate.

So the PR description should say, briefly:
- what the package does and why it was chosen,
- what alternatives were considered and why they lost,
- license, and the pinned version.

A reviewer approving a diff with no idea *why* this specific package was picked is rubber-stamping, not reviewing.

## 4. After merge

There's no Dependabot/Renovate configured yet (known gap — CI dependency scanning is inconsistent/bypassable, see `03_Application_Security_Vulnerabilities.md`). Until that's fixed: **re-run `composer audit` / `npm audit` periodically** (e.g. monthly) rather than assuming CI will surface a CVE disclosed *after* a package is already installed.

## 5. Document the decision

The repo's own docs convention already has a place for this: `docs/06-decisions/adr-XXX-*.md`. For any non-trivial package choice (i.e. anything beyond an obvious, single-option pick), write a short ADR — what, why, alternatives rejected, and why — instead of letting the reasoning live only in a PR description that gets harder to find later.

---

### Quick version (for a trivial/obvious pick)

1. License OK (not AGPL)?
2. Maintained in the last ~12 months, not archived?
3. `composer audit` clean locally?
4. Version pinned, lock file committed?
5. One line in the PR description on why this package.

If any of 1–3 gives you pause, slow down and do the full checklist above.

1. License not AGPL
2. 