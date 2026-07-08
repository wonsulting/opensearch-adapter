# OpenSearch Adapter — Operational Runbook

How to install, test, analyse, and release `wonsulting/opensearch-adapter`, and how to use a local checkout
from the WonsultingAI app. For what the package *is* and how it is structured, see
[architecture.md](./architecture.md).

> **Read this first.** As of this writing a plain `composer install` **fails** on a fresh checkout, and CI is
> **red**, because of an unresolved security-advisory block on the pinned dev dependencies. This is a known
> issue with a documented local workaround (see [Troubleshooting](#troubleshooting--known-issues)); the real
> fix is a separate dependency/CI modernization task. Every command in this runbook was executed while writing
> it, and the outputs shown are the real ones observed on the versions noted.

## Prerequisites

- **PHP** and **Composer 2.x**.
- Documented target runtime: **PHP 8.2+** (per the README and `CONTRIBUTING.md`).
- Actual `composer.json` floor: **`php: ^7.4 || ^8.0`** and **`illuminate/support` for Laravel 6–13**. This is
  wider than the documented support matrix — see the [compatibility note](#compatibility-note).
- No database or running OpenSearch cluster is needed for the test suite: tests are pure unit tests that mock
  the client.

## Install

```bash
composer install
```

On a fresh checkout under the current dependency pins this currently **fails** at the dependency-resolution
step (security advisories on the transitively-required Laravel 9). See
[Troubleshooting](#troubleshooting--known-issues) for the reason and the local workaround that produces a
working `vendor/`. There is intentionally **no committed `composer.lock`** (this is a library; `/vendor` and
`/composer.lock` are git-ignored).

## Compatibility note

The stated compatibility and the actual constraints disagree; this is a known inconsistency to reconcile in a
later modernization phase. Documented here so the discrepancy is not mistaken for a bug:

| Source | PHP | Laravel |
| --- | --- | --- |
| README / `CONTRIBUTING.md` | 8.2+ | 11.x – 13.x |
| `composer.json` (`require`) | `^7.4 || ^8.0` | 6 – 13 (`illuminate/support`) |
| `composer.json` (`require-dev`) | — | 9 (via `orchestra/testbench ^7.5`) |
| CI `test.yml` matrix | 7.4, 8.0, 8.1, 8.2 | testbench 5.0 / 6.0 / 7.0 / 8.5 (Laravel 7 / 8 / 9 / 10) |

## Composer scripts

Defined in `composer.json`:

| Script | Command | Purpose |
| --- | --- | --- |
| `test` | `./vendor/bin/phpunit --testdox` | Run the unit suite |
| `test-coverage` | `./vendor/bin/phpunit --testdox --coverage-text` | Suite + text coverage (needs a coverage driver) |
| `check-style` | `./vendor/bin/php-cs-fixer fix --allow-risky=yes --dry-run --diff --show-progress=dots --verbose` | Style check (read-only) |
| `fix-style` | `./vendor/bin/php-cs-fixer fix --allow-risky=yes` | Apply style fixes in place |
| `analyse` | `./vendor/bin/phpstan analyse` | PHPStan (level `max`, `src` only) |

`--allow-risky=yes` is required because the rule set (`.php-cs-fixer.dist.php`) enables risky rules such as
`declare_strict_types` and the `php_unit_*` rules.

### Observed output

Executed locally on **PHP 8.4.23** with the versions a fresh install resolves today (PHPUnit 9.6.35,
PHPStan 1.12.33, PHP-CS-Fixer 3.95.12, opensearch-php 2.6.0). On the CI PHP versions (7.4–8.2) the PHP-8.4
deprecation notices below do not fire.

**`composer test`** — passes:

```
...
Time: 00:00.068, Memory: 12.00 MB

OK, but incomplete, skipped, or risky tests!
Tests: 122, Assertions: 172, Risky: 2.
```

The 2 "risky" results are only PHP 8.4 emitting `Implicitly marking parameter … as nullable is deprecated`
notices from `DocumentManager`, `Document`, and `Index` (the code uses the legacy `Type $x = null` form). The
assertions still pass; this is PHP-8.4-local noise, not a failure.

**`composer check-style`** — passes (0 of 40 files need fixing):

```
Found 0 of 40 files that can be fixed in ... seconds
```

It also prints a warning that PHP-CS-Fixer is running on PHP 8.4 while the project floor is 7.4, and that a few
configured rules are deprecated (`compact_nullable_typehint`, `function_typehint_space`,
`no_trailing_comma_in_singleline_array`, `visibility_required`). These are warnings only.

**`composer fix-style`** — no-op on a clean tree: `Fixed 0 of 40 files`.

**`composer analyse`** — currently **fails** with 6 errors:

```
[ERROR] Found 6 errors
```

Five are PHP 8.4 `implicitly nullable via default value null` reports (`DocumentManager`, `Routing`, `Index`);
one is a pre-existing `OpenSearch\Client::search()` array-shape mismatch flagged at PHPStan level `max`. These
reflect drift between the code (written for PHP 7.4 / an older SDK) and current PHP + SDK versions, and are
among the things the deferred modernization will address. The `phpstan-baseline.neon` include exists but is
empty, so nothing is currently suppressed.

**`composer test-coverage`** — runs the suite but reports `Warning: No code coverage driver available` unless
Xdebug or PCOV is installed. CI runs with `coverage: none`, so coverage is never produced there either.

## CI ↔ script mapping

CI lives in `.github/workflows/`. The three quality workflows trigger on pushes to any branch **except
`master`**, and ignore tags (so tag-based releases don't re-run them).

| Workflow file | Name | PHP | Runs |
| --- | --- | --- | --- |
| `test.yml` | Tests | matrix 7.4 / 8.0 / 8.1 / 8.2 | `composer test` |
| `code-style.yml` | Code style | 8.0 | `composer check-style` |
| `static-analysis.yml` | Static analysis | 8.0 | `composer analyse` |
| `stale.yml` | Close stale issues/PRs | — | scheduled housekeeping (unrelated) |

Notes:

- `test-coverage` and `fix-style` have **no CI job** — they are local-only.
- `test.yml` does **not** run a plain `composer install`; each matrix row force-installs its own
  testbench/phpunit versions:
  `composer require --no-interaction --dev orchestra/testbench:^<v> phpunit/phpunit:^<v>`.
  `code-style.yml` and `static-analysis.yml` use plain `composer install --no-interaction`.

## Testing / tooling notes

- **Test layout:** a single `unit` testsuite (`phpunit.xml.dist`) over `tests/Unit`. There is no
  feature/integration suite.
- **Base class:** every test extends `PHPUnit\Framework\TestCase` directly. `orchestra/testbench` is a
  dev-dependency and drives the CI matrix (to prove install-time compatibility across Laravel versions) but no
  test extends Testbench's `TestCase`.
- **Mocking `final` classes:** `tests/Extensions/BypassFinalExtension.php` (a PHPUnit `BeforeTestHook`) calls
  `DG\BypassFinals::enable()` before each test, which is why `dg/bypass-finals` is a dev-dependency.
- **Static analysis:** `phpstan.neon.dist`, level `max`, analyses `src` only.
- **Style:** `.php-cs-fixer.dist.php`, `@PSR2` plus a custom rule set, scanning `src` and `tests`.

## Testing against the WonsultingAI app locally (Composer path repository)

To validate a local adapter change against the app before publishing, point the app at your working copy with a
Composer `path` repository (Composer symlinks it, so edits are picked up immediately). This repo ships no such
config; add it in the **app's** `composer.json`:

```jsonc
// app composer.json
"repositories": [
    { "type": "path", "url": "../opensearch-adapter" }   // path to your local checkout
]
```

Then require the dev version in the app:

```bash
composer require wonsulting/opensearch-adapter:@dev
```

Confirm it symlinked rather than copied:

```bash
ls -l vendor/wonsulting/opensearch-adapter   # should be a symlink → ../opensearch-adapter
```

To revert to the published package, remove the `repositories` entry and re-require a real version
(`composer require wonsulting/opensearch-adapter:^<x.y>`). Note the WonsultingAI app is a separate repository
and is not part of this workspace; adjust the relative `url` to wherever the two are checked out.

## Release process

The package is distributed via Composer as `wonsulting/opensearch-adapter` and released by tagging `master`.
It follows [semantic versioning](https://semver.org/); keep the major in step with the
`wonsulting/opensearch-client` line it targets (currently `^2.0`).

1. Ensure `master` is green: `composer test`, `composer analyse`, and `composer check-style` all pass. (See
   [Troubleshooting](#troubleshooting--known-issues) — this is currently blocked until the dependency/CI fix
   lands.)
2. Choose the next semver tag (`vMAJOR.MINOR.PATCH`): PATCH for fixes, MINOR for backward-compatible features,
   MAJOR for breaking API changes (e.g. renaming `Hit::explaination()`, or dropping PHP/Laravel versions).
3. Tag and push from `master`:
   ```bash
   git checkout master && git pull
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```
4. Packagist publishes the new version. If a GitHub → Packagist webhook is configured, this is automatic;
   otherwise trigger an update on the package page. **Verify** the release appeared at
   `https://packagist.org/packages/wonsulting/opensearch-adapter`. The webhook wiring was not verified while
   writing this runbook — confirm it once before relying on auto-sync.

## Troubleshooting / known issues

### `composer install` fails with security-advisory errors

Symptom (fresh checkout, default Composer policy):

```
Your requirements could not be resolved to an installable set of packages.
  Problem 1
    - Root composer.json requires orchestra/testbench ^7.5 ...
    - orchestra/testbench[...] require laravel/framework ^9.x -> found laravel/framework[v9.x]
      but these were not loaded, because they are affected by security advisories (...)
```

**Cause:** `require-dev` pins `orchestra/testbench ^7.5`, which resolves to a Laravel 9 (`laravel/framework`
v9.x) dev tree. Every Laravel 9 release carries security advisories, and Composer's default policy refuses to
install advisory-flagged packages. Because there is no committed `composer.lock`, the resolver has to pick
versions fresh every time and hits the block.

**CI impact:** the same block hits `code-style.yml`/`static-analysis.yml` (plain `composer install`) and the
lower rows of `test.yml`, so **all three quality workflows are currently red**, failing within seconds at the
dependency step — before any test/analysis tool runs.

**Real fix (out of scope here):** bump the dev toolchain off advisory-blocked Laravel (newer
`orchestra/testbench` + `phpunit`, which entails migrating the `BypassFinalExtension` to PHPUnit 10+/updating
`phpunit.xml.dist`) and refresh the CI PHP/matrix versions. That is tracked as a separate dependency/CI
modernization task, not this documentation change.

**Local workaround (maintainers only — never commit):** to get a working `vendor/` today so you can run the
scripts, disable advisory blocking in your **global** Composer config and ignore platform requirements:

```bash
composer config --global policy.advisories.block false
composer install --ignore-platform-reqs
```

Revert the global setting when done (`composer config --global --unset policy.advisories.block`, or set it back
to `true`). Do not add either of these to the project's `composer.json`.

### `composer analyse` reports errors on PHP 8.4

Expected under the current code + PHP 8.4, as described in [Observed output](#observed-output): five PHP 8.4
implicit-nullable deprecations plus one `OpenSearch\Client::search()` array-shape finding. Not introduced by
this docs change; part of the deferred modernization.

### `composer test-coverage` prints "No code coverage driver available"

Install Xdebug or PCOV locally if you need coverage. It is not required for `composer test`, and CI runs with
coverage disabled.
