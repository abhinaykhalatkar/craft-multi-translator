# Maintaining this fork

This is a **maintained fork** of [`digitalpulsebe/craft-multi-translator`](https://github.com/digitalpulsebe/craft-multi-translator). It exists to carry one feature upstream declined: **translating `craft\elements\Category` elements**. Upstream is deprecating Categories, so the PR was rejected — but consumers who still use categories need the support, so we maintain it here.

The goal: stay current with every upstream **stable** release while keeping our Category patch cleanly on top.

---

## Branch model

| Branch | Role |
|---|---|
| `develop` | Default branch. Mirrors `upstream/develop`, and is the home of this doc + the `upstream-sync` GitHub Action (scheduled Actions only run from the default branch). |
| `category-support` | **The integration branch consumers pin.** Exactly `<latest upstream stable tag>` + one Category commit. Nothing else — no unreleased upstream churn. |
| `feature/category-element-support` | Historical record of the original (rejected) upstream PR. Left untouched. |

`category-support` is deliberately based on a **clean stable tag**, never on `develop`, so consumers get `stable + our patch` and nothing in between.

---

## How a consuming project pins this fork

In the project's `composer.json`:

```jsonc
"repositories": [
    { "type": "vcs", "url": "https://github.com/abhinaykhalatkar/craft-multi-translator" }
],
"require": {
    "digitalpulsebe/craft-multi-translator": "dev-category-support as <latest-stable>"
}
```

- The package name is **unchanged** (`digitalpulsebe/craft-multi-translator`) — composer replaces the upstream package inline. No Craft re-install, same plugin handle `multi-translator`.
- `dev-category-support` pins the branch; `as <latest-stable>` (e.g. `as 2.27.1`) aliases it to a real version so transitive constraints resolve.
- The project's committed `composer.lock` pins the exact commit SHA → reproducible deploys.
- The fork is **public**, so composer clones it in CI with no token. If it's ever made private, add a `COMPOSER_AUTH` token or deploy key.

---

## The automated watcher

[`.github/workflows/upstream-sync.yml`](.github/workflows/upstream-sync.yml) runs weekly (and on demand). It:

1. Fetches upstream tags.
2. Finds the latest upstream **stable** tag and compares it to the tag `category-support` is currently based on.
3. If upstream is ahead, **test-rebases** our Category commit on a throwaway branch and opens (or comments on) an **alert issue** reporting whether the rebase is clean or conflicts.

It **never** rewrites `category-support` — auto-force-pushing a branch a production project pins is precisely what we avoid. Detection + alerting only; a human applies the rebase below.

---

## Runbook: rebase onto a new upstream stable release

When the watcher opens an issue for, say, `2.28.0`:

```bash
cd /path/to/this/fork
git fetch upstream --tags
git push origin --tags                                   # mirror the new tag to the fork

# keep develop current (so the watcher's tag detection stays accurate)
git checkout develop && git merge upstream/develop && git push origin develop

# rebase our one Category commit onto the new stable tag
git checkout category-support
git rebase 2.28.0                                         # replays the Category commit onto 2.28.0
#   if it conflicts (CHANGELOG.md is the usual suspect): fix, `git add <file>`, `git rebase --continue`
git push --force-with-lease origin category-support
```

`--force-with-lease` is safe for consumers: their committed `composer.lock` pins the *previous* exact SHA, so nothing changes in their deploy until they deliberately update.

Then, in each consuming project (e.g. Secutex):

```bash
cd /path/to/project
# composer.json: bump the alias → "dev-category-support as 2.28.0"
composer update digitalpulsebe/craft-multi-translator --with-dependencies
php craft migrate/all --interactive=0                    # apply any upstream migrations shipped in 2.28.0
php craft plugin/list                                    # confirm multi-translator still Installed/Enabled
# commit composer.json + composer.lock
```

Close the watcher issue once done.

---

## Verifying the Category patch still applies

After any rebase:

```bash
php -l src/MultiTranslator.php
php -l src/helpers/ElementHelper.php
php -l src/services/TranslateService.php
php -l src/jobs/BulkTranslateJob.php
grep -n "findTargetCategory" src/services/TranslateService.php   # method present
grep -n "Category::class"   src/MultiTranslator.php              # in getSupportedElementClasses()
```

The Category change is **additive**: no migration, no `$schemaVersion` bump. If upstream ever refactors `getSupportedElementClasses()`, `ElementHelper::query()`, or `TranslateService::findTargetElement()`, the rebase will conflict there — resolve by re-applying the Category branch alongside upstream's new structure (see the original commit for the shape).

---

## What's in the Category patch

One commit. Touches:

- `src/MultiTranslator.php` — `Category::class` added to `getSupportedElementClasses()`
- `src/helpers/ElementHelper.php` — explicit `Category` branch in `query()`
- `src/services/TranslateService.php` — `findTargetCategory()`, a branch in `findTargetElement()`, type-aware debug-log `propagationMethod`, and a Category save path (`propagate=false`)
- `src/jobs/BulkTranslateJob.php` — per-element `try/catch` for `UnsupportedSiteException` / `InvalidConfigException`
- `README.md`, `CHANGELOG.md` — docs

No serializers touched; they were already element-type-agnostic. See the commit message on `category-support` for the full rationale.
