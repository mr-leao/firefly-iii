# Firefly III

Self-hosted personal finance manager. PHP 8.5, Laravel 13, Blade, Vite. License is AGPL-3.0-or-later. Product docs live at https://docs.firefly-iii.org/ (separate `firefly-iii/docs` repository). This file is for working in the tree. It is not a user guide.

The release branch is `main`. Upstream pull requests are accepted only against `develop`. Mergify closes pull requests whose base is `main`.

## Upstream contact

Do not open issues, pull requests, or security advisories, and do not email the maintainer, unless the user explicitly asks.

Published policy: https://docs.firefly-iii.org/explanation/contributing/. Automated and AI-generated submissions may be closed and the account blocked. You must be allowed to license the change under AGPL-3.0-or-later (developer certificate of origin). Spelling and translation fixes go to Crowdin, not GitHub.

If the user explicitly asks you to submit:

- Base the pull request on `develop` and reference an existing issue (`Fixes #1234`). Pull requests without an issue are closed.
- Ask the user to talk to the maintainer first when the change introduces a financial concept, is more than about 25 lines, or only fixes spelling or comments.
- Fill in `.github/pull_request_template.md`, including the AI disclosure checkboxes. Leave the disclosure in place.
- Put this trailer on every commit you create:

  ```
  Assisted-by: [Model Name] via [Tool Name]
  ```

  Example: `Assisted-by: Grok 4.7 via Grok Build`.
- `agents.md` and the pull request template say to add 🍌🍌🍌 to the subject of a pull request, issue, or security advisory.

Security reports go only to james@firefly-iii.org, with a real attack path. Read `.github/security.md` before drafting one. In particular, do not report these as vulnerabilities: user-configured webhook, ntfy, SimpleFIN, or Slack URLs; XSS with no attack that gets the payload in; shared cache flush or purge; IDOR without a working example; anything you have not confirmed in this tree. Only the latest minor (see `releases.md`) is maintained. No backports.

## Where to change things

| Change | Location |
| --- | --- |
| Web UI | `app/Http/Controllers`, `app/Http/Requests`, `resources/views` (Blade) |
| JSON API v1 | `app/Api/V1` (controllers split into List, Show, Store, Update, Destroy), `routes/api.php`, `app/Transformers` |
| Persistence | `app/Repositories/<Area>` behind an interface, bound in the matching `app/Providers/*ServiceProvider.php`. Set the user and user group on the repository. |
| Read transaction lists | `app/Helpers/Collector/GroupCollector` |
| Create domain records | `app/Factory` (these are application factories, not Eloquent factories) |
| Validation beyond form requests | `app/Validation` |
| Model side effects | `app/Handlers/Observer`, registered with `#[ObservedBy]` |
| Transaction rules | `app/TransactionRules` (engine: `SearchRuleEngine`) and `app/Rules` |
| Commands | `app/Console/Commands`, signatures prefixed `firefly-iii:` or `upgrade:` |
| Scheduled work | `app/Jobs`, `app/Support/Cronjobs`. Cron entry is `GET /api/v1/cron/{cliToken}`. |
| Web routes | `routes/web.php` |
| Config | `config/`, especially `config/firefly.php` |

`composer.json` maps `Domain\` to `domain/`. That directory is not in the tree. Application code is `FireflyIII\` under `app/`.

Mago perimeter (`mago.toml`): `FireflyIII\Services` and `FireflyIII\Transformers` may depend on themselves, native PHP, and `FireflyIII\Models`. The `FireflyIII\` namespace may use `Carbon\Carbon`. Keep new code inside those bounds.

The app stays offline until the user opts in. Exchange rates, webhooks, and update checks belong in jobs or cron, not in the request path.

## Money model

Bookkeeping is double entry.

- A `TransactionGroup` is what the user sees as one transaction. It has one or more `TransactionJournal` rows (splits).
- Each journal has `Transaction` rows. A normal split is two legs: source amount negative, destination amount positive. Account types decide the meaning.
- Journal types (`TransactionTypeEnum`): Withdrawal, Deposit, Transfer, Opening balance, Reconciliation, Liability credit, Invalid.
- Account types (`AccountTypeEnum`) are stored as the enum string (`Asset account`, `Expense account`, …), not the case name.
- Amounts, foreign amounts, and native (primary-currency) amounts are strings. Casts on `Transaction` return strings. Add, subtract, compare, and round with `bcmath` (`bcadd`, `bcsub`, `bcmul`, `bcdiv`, `bccomp`). `app/Support/Steam.php` and `app/Support/Amount.php` are the shared helpers.
- Each journal has a currency. `foreign_amount` is the other side of a multi-currency split. `native_amount` is the value in the administration's primary currency. When you change an amount, update the native amounts in the same write path.
- Records carry `user_id` and `user_group_id`. Route binders (`routeBinder` on the models) resolve an id only for the authenticated user and throw `NotFoundHttpException` otherwise. New queries need the same scope. Roles are `app/Enums/UserRoleEnum`; repositories use `UserGroupTrait`.
- Financial models use soft deletes. Observers react to deletion. Do not hard-delete user data unless the existing purge path already does.

Search queries go through `app/Support/Search/QueryParser`. `phpunit.xml` sets `QUERY_PARSER_IMPLEMENTATION=new`.

## PHP style

Match the surrounding file. The formatter is Mago (`mago.toml`), not Pint.

- `declare(strict_types=1);` in every PHP file.
- Copy the AGPL header from a neighbor. Filename on the first comment line, copyright year is the year you add the file, `james@firefly-iii.org`.
- Four spaces, LF, final newline, no trailing whitespace. PHP print width is 160. Mago uses `null|Type`, no trailing commas, aligned assignments, and sorted class methods.
- Prefer `Safe\` functions (`use function Safe\...`) where the file already uses them.
- Do not call `$request->all()`. Mago enables `no-request-all`.
- Form requests and `app/Validation` validate input. Controllers stay thin and call a repository or factory.
- Log with `Log::debug(sprintf(...))` the way neighbors do. Do not log secrets, passwords, or full account numbers.
- Leave a behavior change as an issue rather than a `TODO` in the code.

## JavaScript and views

Frontend source is `resources/assets/v3`: one Vite entry per page, Alpine (CSP build), Bootstrap 5, AdminLTE 4, Chart.js. Templates are Blade in `resources/views`. Twig has been removed; comments that still say `groups.twig` are stale.

- A new page script must be added to the `input` list in `resources/assets/v3/vite.config.js`. Build output is `public/build`.
- Prettier print width for JS is 120 (`.prettierrc`). ESLint is `eslint.config.mjs` over `resources/assets/v3/**/*.js`.
- `public/v1` is legacy bundled assets. Change `resources/assets/v3` and rebuild.
- `npm run dev` in `resources/assets/v3` reads TLS certificates from `/vagrant/tls-certificates/wildcard.sd.internal.{key,crt}` and uses the origin hardcoded in `vite.config.js`. Leave that dev-server block alone unless the task is about it.

## Translations

Edit English only: `resources/lang/en_US` (PHP) and the English entries under `resources/locales` (frontend JSON). Other locales come from Crowdin (`crowdin.yml`). Do not hand-edit them.

## Data changes

Ship a migration in `database/migrations` and, when existing installations must change rows, a command under `app/Console/Commands/Upgrade` wired from `firefly-iii:upgrade-database` (`UpgradesDatabase`). `composer update` runs that command. A migration alone does not upgrade existing databases.

Supported database drivers are MySQL/MariaDB (default), PostgreSQL, and SQLite. Keep SQL portable across those three.

## Verify

PHPUnit (`phpunit.xml`) forces `APP_ENV=testing` and SQLite. `.env.example` says the real `.env` must not set `APP_ENV=testing`.

```bash
composer unit-test
composer integration-test
./vendor/bin/phpunit -c phpunit.xml --filter ClassName
```

- Unit tests: `tests/unit`, namespace `Tests\unit\...`. No database unless the test already boots one.
- Integration tests: `tests/integration`, namespace `Tests\integration\...`, extend `Tests\integration\TestCase` (`RefreshDatabase`, seeded). Use `createAuthenticatedUser()` for a fresh user and group.
- Feature tests exist and are thin. `*.php.disabled` files are out of the suite. Leave them disabled.
- New behavior needs a test that fails before the change. Prefer a unit test for pure logic (dates, search parser, amounts) and an integration test for HTTP or API.

Static analysis, when `vendor/` is installed:

```bash
./vendor/bin/mago fmt
./vendor/bin/mago lint
./vendor/bin/phpstan analyse -c .ci/phpstan.neon
```

PHPStan is level 6. `.ci/phpstan.neon` ignores many identifiers on purpose. Do not raise the level or "fix" ignored categories as part of an unrelated change. Locally, `.ci/phpstan.sh` writes `phpstan-report.txt`.

Do not run `.ci/phpcs.sh` or `.ci/phpmd.sh` as a routine check. They run Composer inside `.ci/php-cs-fixer` and `.ci/phpmd`. Do not run Rector (`.ci/rector.sh`) across the tree unless the task is a Rector migration.

Frontend, from the repository root (npm workspaces include `resources/assets/v3`):

```bash
npm install
npm --prefix resources/assets/v3 run build
```

## Local setup

```bash
cp .env.example .env
composer install
php artisan key:generate
```

Set `DB_*` for a reachable MySQL or PostgreSQL database, then:

```bash
php artisan migrate --seed
php artisan firefly-iii:upgrade-database
php artisan firefly-iii:laravel-passport-keys
```

`APP_KEY` is a 32-character string. Default database in `.env.example` is MySQL host `db`, database `firefly`. Docker and Kubernetes install steps are in the docs, not in this repository.

## Leave alone

- Generated or downloaded translation files.
- `composer.lock` and `package-lock.json`, unless the task changes dependencies.
- Ignored PHPStan errors, disabled tests, and commented-out upgrade commands in `UpgradesDatabase`.
- The dev-server TLS and origin settings in `vite.config.js`.
- User data scoping. A query without `user_id` / `user_group_id` (or the authenticated relation) is a bug.
