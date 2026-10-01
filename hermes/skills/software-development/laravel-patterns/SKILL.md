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

## Feature Parity Across Egress Paths

### A new column must reach every path a user reads or re-enters data through

Adding a field to a model is one line; the places that can silently drop it are the ones
that matter. When a field lands, check all of them and open the gaps as their own work:

- form create/edit **and** the edit-fill and reset lists (a property absent from the reset
  array leaks the previous record's value into the next form);
- import mapping **and** the importer's change-detection field list **and** the user-facing
  header hint rendered next to the upload control;
- export column definitions;
- the index table's column list.

**Rule:** search the new field name across the whole app (`search_files` on content) and
treat every hit that is a *definition* (migration, model) as the start, not the end — each
consumer that should read it and does not is a data-loss path. An export that omits a column
the form collects is silent loss: the user types a value, sees it saved, and it never leaves
the system.

When an export or import's contract was specified before the field existed, the drift is in
the spec, not the code — say so explicitly rather than blaming the implementation, and check
whether the spec is the artifact to update.

## External Integrations

### Rate-limiting a public route behind a proxy needs a trusted forwarded-for header

`ThrottleRequests::resolveRequestSignature()` keys anonymous requests on
`$request->ip()`, and `getClientIps()` only reads `X-Forwarded-For` when that header is among
the trusted proxy headers. Configurations that deliberately trust only
`X-Forwarded-Host/Proto/Port/Prefix` — the standard hardening against IP spoofing — fall back
to `REMOTE_ADDR`, which behind a load balancer is the proxy itself.

**Rule:** a `throttle:` on a route with no authenticated user is a per-bucket limit on
whatever the signature resolves to. Behind a proxy that is one shared bucket, so the limit
becomes a global budget for all visitors and everyone past it gets 429 — while a path that
looks unthrottled in testing is throttled in production. Before shipping one, confirm which
header the signature actually resolves to behind the real topology, and give the route a
signature that does not depend on the client IP if the answer is "the proxy".

### Turning a transport failure into a value: apply it at every caller, or record why not

The useful part of a typed-result adapter is that a connection failure stops being an
exception travelling through call sites that have to guess. It only buys that where it is
used, so a scheduled job that still calls the raw service keeps its old failure path.

**Rule:** when introducing a failure-as-value abstraction, enumerate every caller of the
underlying service — HTTP controllers, queued jobs, scheduled commands — and either convert
all of them or write down in the abstraction's docblock why the converted call sites are the
complete set. A job whose failure is routed through the queue's failure handler is a
legitimate reason to stay unconverted; an unexamined third caller is not.

## Operational Data

### An append-only table written by a high-frequency job needs a retention owner

A run-record table fed by a job on a short schedule grows linearly with no natural ceiling:
a five-minute job is ~105k rows a year. A timestamp index keeps the *reads* cheap, so nothing
looks wrong for months — the cost is storage, plus an ever-growing backup.

**Rule:** before shipping a per-run record table, find the project's existing retention
command and check whether it actually covers the new table — a maintenance command that
handles one model is not retention for the next one. If nothing covers it, either extend the
existing command or add a prune with a `--keep-days` flag. Grep for the table name across
the console commands to prove which side of that line you are on; do not infer coverage from
the existence of a retention command.

### A duration shared by a writer and a reader belongs in one constant

When a writer stores data with a TTL and a reader decides "is this fresh" by comparing against
a time window, the two numbers encode one policy and must move together. A literal in each
place compiles, passes tests, and diverges the first time someone changes the TTL — the reader
then reports the data as stale while the writer is still refreshing it.

**Rule:** express the duration once (a class constant, or the reader reading the same TTL
config the writer used) so changing it is a one-line change. Also remember a timezone
mismatch bites here: comparing an app-timezone wall clock against `time()` is only correct
because the app sets the default timezone at bootstrap; reading a stored timestamp through a
raw query facade returns a string, not a cast instance, and loses that guarantee.

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