# OpenSearch Adapter — Operational Runbook

How to install, test, analyse, and release `wonsulting/opensearch-adapter`, and how to use a local checkout
from the WonsultingAI app. For what the package *is* and how it is structured, see
[architecture.md](./architecture.md).

> **Read this first.** The dependency/CI modernization has landed (EN-724 raised the platform floor and
> removed `orchestra/testbench`; EN-728 rebuilt the CI matrix). A plain `composer install` now resolves
> cleanly and CI is green, running a PHP 8.2–8.5 × `illuminate/support` ^11–^13 matrix. Note that some
> detailed figures further down — the "Observed output" tool versions and the test base-class BypassFinal
> note — still describe the pre-modernization state and are being refreshed in a separate docs pass.

## Prerequisites

- **PHP** and **Composer 2.x**.
- Documented target runtime: **PHP 8.2+** (per the README and `CONTRIBUTING.md`).
- `composer.json` floor now matches: **`php: ^8.2`** and **`illuminate/support: ^11 || ^12 || ^13`** (Laravel
  11–13) — see the [compatibility note](#compatibility-note).
- No database or running OpenSearch cluster is needed for the test suite: tests are pure unit tests that mock
  the client.

## Install

```bash
composer install
```

On a fresh checkout this resolves cleanly (the old Laravel-9 security-advisory block was removed together with
`orchestra/testbench` in the EN-724 modernization). There is intentionally **no committed `composer.lock`**
(this is a library; `/vendor` and `/composer.lock` are git-ignored).

## Compatibility note

The documented support and the actual constraints now agree across the README, `composer.json`, and CI:

| Source | PHP | Laravel |
| --- | --- | --- |
| README / `CONTRIBUTING.md` | 8.2+ | 11.x – 13.x |
| `composer.json` (`require`) | `^8.2` | `illuminate/support: ^11 \|\| ^12 \|\| ^13` |
| `composer.json` (`require-dev`) | — | — (no `orchestra/testbench`) |
| CI `test.yml` matrix | 8.2 / 8.3 / 8.4 / 8.5 | `illuminate/support` ^11 / ^12 / ^13 (excl. 8.2 × ^13) |

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

CI lives in `.github/workflows/`. The three quality workflows trigger on **push and pull_request to
`master`**.

| Workflow file | Name | PHP | Runs |
| --- | --- | --- | --- |
| `test.yml` | Tests | matrix PHP 8.2 / 8.3 / 8.4 / 8.5 × `illuminate/support` ^11 / ^12 / ^13 (excl. 8.2 × ^13) | `composer test` |
| `code-style.yml` | Code style | 8.3 | `composer check-style` |
| `static-analysis.yml` | Static analysis | 8.3 | `composer analyse` |
| `stale.yml` | Close stale issues/PRs | — | scheduled housekeeping (unrelated) |

Notes:

- `test-coverage` and `fix-style` have **no CI job** — they are local-only.
- `test.yml` pins `illuminate/support` per matrix cell rather than running a plain `composer install`:
  `composer require "illuminate/support:^<v>" --no-interaction --no-update` followed by
  `composer update --no-interaction --prefer-stable`. There is no `orchestra/testbench`.
  `code-style.yml` and `static-analysis.yml` use plain `composer install --no-interaction`.

## Testing / tooling notes

- **Test layout:** a single `unit` testsuite (`phpunit.xml.dist`) over `tests/Unit`. There is no
  feature/integration suite.
- **Base class:** every test extends `PHPUnit\Framework\TestCase` directly. There is no `orchestra/testbench`
  dependency; install-time compatibility across Laravel versions is proven by the `test.yml` matrix (which
  pins `illuminate/support` per cell), not by a Testbench base class.
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

1. Ensure `master` is green: `composer test`, `composer analyse`, and `composer check-style` all pass.
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

### `composer install` security-advisory block (historical — resolved)

Earlier revisions pinned `orchestra/testbench ^7.5`, which pulled a Laravel 9 (`laravel/framework` v9.x) dev
tree. Every Laravel 9 release carries security advisories, so Composer's default policy refused to install and
all three quality workflows were red at the dependency step. The EN-724 modernization removed
`orchestra/testbench` entirely (the tests are plain PHPUnit and mock the client), so a fresh `composer install`
now resolves cleanly and no workaround is needed.

### `composer analyse` reports errors on PHP 8.4

Expected under the current code + PHP 8.4, as described in [Observed output](#observed-output): five PHP 8.4
implicit-nullable deprecations plus one `OpenSearch\Client::search()` array-shape finding. Not introduced by
this docs change; part of the deferred modernization.

### `composer test-coverage` prints "No code coverage driver available"

Install Xdebug or PCOV locally if you need coverage. It is not required for `composer test`, and CI runs with
coverage disabled.
