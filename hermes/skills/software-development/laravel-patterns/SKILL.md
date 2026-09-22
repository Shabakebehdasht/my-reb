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

## Database Constraints

### SoftDeletes + foreign key = `delete()` is not enough

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