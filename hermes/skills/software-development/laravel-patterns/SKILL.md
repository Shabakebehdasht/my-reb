---
name: laravel-patterns
description: "Laravel pitfalls: model events, cache batching, factories."
version: 1.0.0
author: Hermes
license: MIT
metadata:
  hermes:
    tags: [laravel, php, pitfalls, model-events, cache, testing]
    category: software-development
---

# Laravel Patterns & Pitfalls

## When to Use

Use this skill when working on any Laravel project — implementing model observers, designing cache invalidation, writing tests with factories, managing event listeners, or debugging FK constraint violations.

Procedures and hard-won rules for Laravel development — model events, cache invalidation, testing with factories, database constraints. Standalone rules; the project's AGENTS.md provides domain-specific context.

## Model Events

### `saved` event fires twice during `update()`

In Laravel 13, `Model::saved()` can fire multiple times per `update()` call — once before changes are committed (changes array empty, originals not yet reset) and once after. Use `getChanges()` to detect what actually changed, not `isDirty()`.

**Rule:** In `saved` callbacks, detect attribute changes via `$model->getChanges()` (returns only attributes persisted in the last save), not `isDirty()` (compares against original loaded state, unreliable mid-event).

For detecting "was this a create or update", use `$model->wasRecentlyCreated` — but note it stays `true` on subsequent saves of the same in-memory instance. Reload from DB if you need a clean state.

### `deleted` event has no dirty attributes

`isDirty()` and `getChanges()` are both empty in `deleted` events. Handle delete-specific logic separately from saved/update logic.

## Cache Invalidation Batching

### Batch mode needs a separate flag, not an empty-array check

When implementing request-scoped batch buffering for cache increments, use a dedicated `bool $batching` flag — NOT `!empty($this->pending)`. An empty pending array at batch start means `empty()` returns true, bypassing the buffer entirely.

```php
// WRONG — empty array bypasses buffer on first increment
if (! empty($this->pending)) { ... }

// CORRECT — separate flag tracks batch scope
private bool $batching = false;
public function batch(Closure $callback): mixed {
    $this->batching = true;
    $this->pending = [];
    try {
        $result = $callback();
    } finally {
        $this->flushPending();
        $this->batching = false;
    }
    return $result;
}
```

Always use `try/finally` in `batch()` to guarantee `flushPending()` runs even on exception.

## Testing with Factories & Cache

### Set cache AFTER factory setup, not before

Model factories trigger `saved` events that increment cache versions. If you `Cache::put('x_version', 0)` before `User::factory()->create()`, the factory's internal model creation will bump the version back up. Always set cache assertions AFTER all factory/model creation is complete.

```php
// WRONG
Cache::put('gis_version', 0);
$user = User::factory()->create(); // creates Person -> bumps gis to 1
// gis_version is now 1, not 0

// CORRECT
$user = User::factory()->create();
Cache::put('gis_version', 0); // reset AFTER factory side effects
$person->update([...]);
$this->assertEquals(0, Cache::get('gis_version'));
```

### User factory creates Person in `afterMaking`

If `UserFactory` creates a backing `Person` via `afterMaking()`, the Person's `saved` event fires during factory setup. Factor in these cascading cache invalidations when testing cache behavior on User or related models.

## Query Pitfalls

### `ORDER BY` + `LIMIT` returns the OLDEST N, not the newest

`orderBy('day')->limit(30)` after a `groupBy('day')` returns the 30 *oldest* buckets, because ORDER BY is applied before LIMIT. Any "last N days/weeks" chart or list built this way silently shows the oldest window once the table holds more than N distinct periods.

**Rule:** take the newest N in a subquery, then re-sort for display in the outer query. Flipping only the existing `orderBy` to `desc` returns the right rows in reversed axis order — it is not a fix.

```php
// WRONG — oldest 30 days
Ticket::whereIn('unit_id', $ids)
    ->selectRaw('date(created_at) as day, count(*) as count')
    ->groupBy('day')->orderBy('day')->limit(30)->get();

// RIGHT — newest 30 days, displayed oldest → newest
$daily = DB::table('tickets')->whereIn('unit_id', $ids)
    ->selectRaw('date(created_at) as day, count(*) as count')
    ->groupBy('day')->orderByDesc('day')->limit(30);

DB::query()->fromSub($daily, 'daily')->orderBy('day')->get();
```

The resulting axis is **sparse** — the N most recent periods *that have rows*, not N consecutive periods. A test that asserts "N consecutive days" fails against correct code; assert the real contract instead (≤ N buckets, strictly ascending, newest bucket reaches the newest data).

### A model without `HasFactory` has no `::factory()`, whatever factories exist

A `XyzFactory` file on disk does not mean `Xyz::factory()` resolves — that requires `use HasFactory;` on the model. Without it you get `BadMethodCallException: Call to undefined method`, which reads like a broken factory definition but is a missing trait on the model.

**Rule:** before reaching for `Model::factory()`, confirm the model actually declares `HasFactory` (`search_files` for `HasFactory` in the model, or read the model). If it doesn't, build the row with `Model::create([...])` and populate foreign keys explicitly. A factory file with no `HasFactory` consumer is dead code — do not add the trait just to make a test terse.

### Back-dating a row: `created_at` is usually not fillable

`$fillable` governs mass assignment, and timestamps are typically excluded. Passing `created_at` to `create()` is silently dropped (or throws under strict mode), so the row lands at "now" and any test about time windows passes for the wrong reason.

**Rule:** create the row, then set the timestamp and persist:

```php
$row = Ticket::create([...]);
$row->forceFill(['created_at' => $day, 'updated_at' => $day])->save();
```

Give time-window fixtures a **midday** timestamp. The app timezone and the database session timezone often differ, and `date(column)` buckets in the *session* zone — a midnight value can fall in the previous day and shift the whole window.

## End-to-End Tests

### A failing E2E test may be the test that is wrong

Browser-level tests assert against a *rendering*, and the rendering is often sparse, paginated, or deduped in ways the test author did not model. When an E2E test fails but the feature-level test for the same behavior passes, suspect the E2E assertion before the application code — then prove it: read the actual rendered payload and compare it to what the code is contracted to produce.

**Rule:** before changing production code to satisfy a browser test, confirm which side encodes the wrong assumption, and state the finding out loud. An E2E test that demands "N consecutive rows" fails legitimately against a chart or list that is defined as "the N most recent rows that have data" — assert the real invariant (count bounded, order monotonic, newest record present) instead of a shape the feature never promised.

Read the payload the app already exposes to the browser rather than inferring it from pixels. When a charting library holds the data, the rendered instance is the source of truth:

```ts
const chart = (window as any).Highcharts.charts.find((c: any) => c?.renderTo === container);
const categories = chart.xAxis[0].categories;
```

Verify the date/label conversion in both languages instead of hand-rolling calendar arithmetic on one side. Format dates with the platform's own calendar implementation (e.g. `Intl.DateTimeFormat` with the `persian` calendar) and resolve labels back to real dates by a **round-trip** — a label is a match only if formatting the candidate date reproduces the label. Hand-written month-offset approximations drift across year boundaries and silently assert against the wrong date.

## Static Analysis

### A regenerated baseline is evidence, not a formality

Baselines that record an occurrence **count** per file go stale when your change removes one of those call sites, and the analyzer then reports "expected to occur N times, but occurred only M" — a non-ignorable error that blocks the build even though the code improved.

**Rule:** when an edit removes a baselined call, regenerate and read the diff. A count decreasing is correct. Any *added* entry means a real new error just got suppressed — investigate it instead of accepting the regeneration. Never hand-edit a count to silence the mismatch; regenerate so the file stays reproducible.

## Test Environment Safety

### A test harness that swaps config must restore it on every exit path

A run that backs up `.env` and substitutes a test config leaves the wrong file in
place when it fails, because a failure skips the restore step. The next task then
runs against the test database and the damage is invisible until something is
overwritten.

**Rule:** make the restore unconditional (`trap`/cleanup block) rather than the
last line of a success path, and prove it by checking the restored file's
identity after the run — not by assuming the script reached the end.

```bash
# Verify identity, not just existence
grep -E '^DB_DATABASE=' .env     # must name the dev DB, not the test DB
```

Service credentials are covered by the `git-workflow` skill's masked-secret rule
— never mutate one from a masked or inferred value.

Models with `SoftDeletes` only set `deleted_at` — the row stays, and FK constraints still block related deletes. Use `forceDelete()` when you need to actually remove the row for FK-sensitive cleanup in tests.

```php
// WRONG — soft-deletes, FK still blocks person delete
$user->delete();
$person->delete(); // FK violation!

// CORRECT
$user->forceDelete(); // actually removes the row
$person->delete();
```

## Event Cache

### Stale event cache after removing listeners

`bootstrap/cache/events.php` caches event-to-listener mappings. After deleting a listener class and removing it from `EventServiceProvider::$listen`, run `php artisan event:clear` — otherwise the Dispatcher tries to `include()` the deleted file and crashes.

```bash
php artisan event:clear
```

This is separate from `config:clear` and `route:clear`.

## Verification

- After cache batching changes: run the full test suite — dedup bugs silently pass unit tests but break feature tests that assert exact version counts.
- After removing event listeners: always `event:clear` before running tests.
- After model event changes: test both create and update paths — they exercise different code paths in `saved` callbacks.