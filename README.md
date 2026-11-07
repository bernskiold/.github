# .github

# Reusable workflows

Shared GitHub Actions workflows for Bernskiold repositories. Call them from a
thin workflow in your repo with
`uses: bernskiold/.github/.github/workflows/<file>@main`.

| Workflow | Purpose |
| --- | --- |
| [`wordpress-plugin-tests.yml`](.github/workflows/wordpress-plugin-tests.yml) | PHPUnit against real WordPress over a PHP x WP matrix, optional PHP/JS lint, committed-build check, Jest and Playwright e2e |
| [`wordpress-plugin-release.yml`](.github/workflows/wordpress-plugin-release.yml) | Bump versions, build, commit, move the tag and attach the plugin ZIP to the release |
| [`satis-notify.yml`](.github/workflows/satis-notify.yml) | Ask packages.bernskiold.com to rebuild its Satis index for this repo (any Composer package) |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Enable auto-merge (squash) on Dependabot patch and minor PRs |
| [`update-changelog.yml`](.github/workflows/update-changelog.yml) | Prepend the release notes to `CHANGELOG.md` |

## `wordpress-plugin-tests.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `php-versions` | string (JSON array) | `'["8.2","8.3","8.4","8.5"]'` | PHP versions in the PHPUnit matrix |
| `wp-versions` | string (JSON array) | `'["7.0","latest"]'` | WordPress versions in the PHPUnit matrix |
| `node-version` | string | `'26'` | Node.js version |
| `build` | boolean | `true` | Run `npm ci && npm run build` before the PHP tests (skipped if there is no `package.json`) |
| `js-unit` | boolean | `false` | Run `npm run test:unit` |
| `e2e` | boolean | `false` | Start wp-env (`npm run env:start`) and run Playwright (`npm run test:e2e`); the report in `artifacts/` is uploaded |
| `php-lint` | boolean | `false` | Run `composer lint` (phpcs) |
| `js-lint` | boolean | `false` | Run `npm run lint:js` and `npm run lint:css`, each only if the script exists |
| `check-dist` | boolean | `false` | Run `npm run build` and fail if `git status --porcelain` shows changes, i.e. the committed build output does not match the sources |
| `working-directory` | string | `'.'` | Plugin directory relative to the repo root. Every npm, composer and PHPUnit step runs there, and the npm cache, composer install and Playwright report paths follow it. Use it for a plugin inside a site repo, e.g. `wp-content/plugins/ehs-courses` |

The plugin must provide `bin/install-wp-tests.sh [wp-version] [db-name] [db-user] [db-pass] [db-host]`
(as in bm-concierge / bm-block-library) that prints `KEY=value` lines such as
`WP_PHPUNIT__DIR=...`, and `composer test` running PHPUnit. A MySQL 8 service is
provided. The `php-lint` and e2e jobs use the first entry of `php-versions`.

For a plugin that lives inside a site repo, point the workflow at it:

```yaml
jobs:
    tests:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-tests.yml@main
        with:
            working-directory: wp-content/plugins/ehs-courses
```

```yaml
# .github/workflows/tests.yml
name: Tests

on:
    push:
        branches: [main]
    pull_request:

permissions:
    contents: read

concurrency:
    group: tests-${{ github.ref }}
    cancel-in-progress: true

jobs:
    tests:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-tests.yml@main
        with:
            php-versions: '["8.2","8.3","8.4","8.5"]'
            wp-versions: '["7.0","latest"]'
            js-unit: true
            e2e: true
```

## `wordpress-plugin-release.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `build` | boolean | `true` | Run `npm ci --ignore-scripts && npm run build` before committing |
| `node-version` | string | `'26'` | Node.js version for the build |
| `main-file` | string | `''` | Main plugin file with the `Version:` header; empty means `<repo-name>.php` |
| `version-files` | string | `'src/Plugin.php'` | Newline-separated files in which `version = '...'` is bumped (missing files are skipped with a warning; pass `''` for none) |
| `composer` | boolean | `false` | Run `composer install --no-dev --optimize-autoloader` after the release commit and ship `vendor/` in the ZIP |

On a published release it strips a leading `v` from the tag, bumps
`package.json`, the `Version:` header and the `version-files`, builds, commits
"Prepare release x.y.z" to the default branch, force-moves the tag onto that
commit and attaches `<repo-name>.zip` (made with `git archive`, so
`.gitattributes` `export-ignore` applies) to the release.

With `composer: true`, or when the repo has a `.distignore`, the ZIP is built
from the `git archive` output plus the production `vendor/`, and `.distignore`
(rsync patterns, as used by `wp dist-archive`) is applied before zipping.
`vendor/` is installed after the release commit, so it is never committed.
The commit and the tag move target the repo's default branch, so `main` and
`master` repos both work.

**Name the caller workflow `Build Release`.** The tag is moved with
`GITHUB_TOKEN`, and pushes made with `GITHUB_TOKEN` never trigger other
workflows, so `push: tags` does not fire for the final commit. The Satis caller
below listens to `workflow_run` of `Build Release` instead, which is what
makes Satis pick up the bumped release commit.

```yaml
# .github/workflows/release.yml
name: Build Release

on:
    release:
        types: [published]

permissions:
    contents: write

jobs:
    release:
        uses: bernskiold/.github/.github/workflows/wordpress-plugin-release.yml@main
        with:
            version-files: |
                src/Plugin.php
```

## `satis-notify.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `repository-url` | string | `''` | URL as listed in `satis.json`; empty means `https://github.com/<owner>/<repo>` |
| `satis-repository` | string | `'bernskiold/packages.bernskiold.com'` | Repository that builds the Satis index |

Needs the `SATIS_DISPATCH_TOKEN` org secret, so call it with `secrets: inherit`.
When triggered by `workflow_run` it only dispatches if that run succeeded.
Concurrent triggers for the same repository collapse into one dispatch.
Works for Laravel packages and other Composer packages too (drop the
`workflow_run` trigger if the repo has no "Build Release" workflow).

```yaml
# .github/workflows/notify-satis.yml
name: Notify Satis

on:
    push:
        tags: ["*"]
    release:
        types: [published]
    # The release workflow moves the tag with GITHUB_TOKEN, which never fires
    # `push: tags`. This catches the final, version-bumped commit.
    workflow_run:
        workflows: ["Build Release"]
        types: [completed]
    workflow_dispatch:

jobs:
    notify:
        uses: bernskiold/.github/.github/workflows/satis-notify.yml@main
        secrets: inherit
```

## `dependabot-auto-merge.yml`

No inputs. Enables auto-merge (squash) on Dependabot patch and minor updates;
majors are left for review. Requires "Allow auto-merge" in the repo settings.

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot Auto-Merge

on: pull_request_target

jobs:
    auto-merge:
        uses: bernskiold/.github/.github/workflows/dependabot-auto-merge.yml@main
        permissions:
            contents: write
            pull-requests: write
```

## `update-changelog.yml`

No inputs. Prepends the release notes to `CHANGELOG.md` on the repo's default branch (`main` or `master`).

```yaml
# .github/workflows/update-changelog.yml
name: Update Changelog

on:
    release:
        types: [released]

jobs:
    update:
        uses: bernskiold/.github/.github/workflows/update-changelog.yml@main
        permissions:
            contents: write
```
