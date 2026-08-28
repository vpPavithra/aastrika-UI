# Release 2.1.2 — 2026-08-28

|                              |                                              |
| ---------------------------- | -------------------------------------------- |
| **Package**                  | `@aastrika_npmjs/comptency` (npm)           |
| **Version published**        | `2.1.2`                                      |
| **Baseline (previous)**      | `2.1.1`                                      |
| **Consumed by**              | eagle-fusion (`^2.1.2`)                      |
| **Commits**                  | `b9c2a61` (fix) + version bump               |
| **Author**                   | vpPavithra                                   |

## Summary

Patch release fixing the **Required** tab (self-assessment) on top of 2.1.1. Users on
that tab could see no data at all — or, once loading was fixed, data that loaded but
never rendered until an unrelated screen interaction happened. Both were silent failures
in the self-assessment data pipeline with no error surfaced. This release makes the
pipeline resilient to bad/partial API responses, removes a runtime dependency on lodash
methods that weren't reliably available in the consuming app's bundle, and fixes how the
component notifies Angular's (zoneless) change detector that new data has arrived.

## 🐛 Fixes

- **self-assessment data pipeline silently failing** — `getUserdetailsFromRegistry`
  threw when the registry response had no `result` key, which aborted the whole
  self-assessment fetch chain *before* course data was ever requested, with no error
  surfaced — course-fetching never ran and the tab quietly showed no data. Guarded with
  optional chaining, and added a `catchError` around the user-details step so a
  failed/empty registry lookup no longer blocks course fetching. Pipeline errors are now
  logged to the console instead of failing silently. (`b9c2a61`)
- **`_.flatMap` / `_.uniqBy` not a function** — these lodash 4.0+ methods weren't
  resolving on the consuming app's runtime bundle (older 3.x-compatible methods like
  `find`/`map`/`forEach` worked; 4.0-era additions didn't), throwing and killing the same
  pipeline. Replaced with native `Array.prototype.flatMap` and a `Map`-based dedup with
  identical semantics — no lodash dependency for either call. (`b9c2a61`)
- **data loaded but stayed invisible until an unrelated UI event** — the self-assessment
  component runs on zoneless change detection, so mutating fields inside a raw RxJS
  `subscribe()` callback wasn't itself recognized as a change; the view stayed stale
  until some unrelated Angular-bound DOM event (e.g. a mouse move elsewhere on the page)
  happened to trigger a check. A prior workaround (`cdr.detectChanges()`) forced an
  immediate synchronous re-check that collided with Angular's dev-mode double-check,
  throwing `NG0100 ExpressionChangedAfterItHasBeenCheckedError`. Replaced with
  `cdr.markForCheck()` in both the success and error paths, which flags the component
  dirty and lets the scheduler check it safely on its own next tick. (`b9c2a61`)

## 🏗️ Build / Chore

- `required-comptency-card.component.ts`: explicitly typed the `language`, `isMobileApp`,
  and `role` `@Input()`s (were implicit `any`). (`b9c2a61`)
- version bump: `package.json` (root + `projects/competency-ui`) → `2.1.2`.

## ⚠️ Deploy notes & risk

- **Config / env / secret changes:** none.
- **Backend / API contract dependencies:** none new. This release only changes how the
  library handles responses it already receives from the registry/FRAC/course endpoints
  introduced in prior releases.
- **Breaking changes:** none. The public API surface (exported components, modules,
  `RequestUtil`) is unchanged.
- **Change-detection assumption:** this release assumes the consuming app runs zoneless
  change detection (`provideZonelessChangeDetection`). If a consumer is still
  zone-based, `markForCheck()` remains correct and harmless there too — it's a superset
  of what zone-based apps already get automatically.

## ✅ Pre-publish checklist

- [ ] Version bumped in `projects/competency-ui/package.json` (the library manifest
      ng-packagr publishes)
- [ ] Build clean (`npm run build-lib`) → `dist/competency-ui` shows `2.1.2`
- [ ] Bundle audit: no `_.flatMap` / `_.uniqBy` calls remain in the self-assessment
      component; native replacements present
- [ ] Manual check: Required tab loads and renders self-assessment data immediately on
      first paint, with no `NG0100` in the console, without needing any extra UI
      interaction
- [ ] Consumer (eagle-fusion) `package.json` + lockfile updated to `^2.1.2`

## Publish & rollback

**Publish** — npm package (no Jenkins). Bump `projects/competency-ui/package.json`,
`npm run build-lib`, then:

```bash
cd dist/competency-ui
npm publish --access public        # @aastrika_npmjs publish rights required (+ --otp if 2FA)
```

Then bump the consumer (eagle-fusion `package.json` → `^2.1.2`, `yarn install`) and
redeploy.

**Rollback** — consumers pin the previous version (`2.1.1`), reinstall, redeploy.
Published npm versions are immutable and are not unpublished. Note: rolling back to
`2.1.1` reintroduces the Required-tab data-loading/rendering bugs this release fixes.

---
_File naming: `RELEASE_NOTES/release-<X.Y.Z>.md`._
