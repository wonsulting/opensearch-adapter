# OpenSearch Adapter — Architecture

This document describes what `wonsulting/opensearch-adapter` is, how it is layered, and every class it
ships. It complements the usage-oriented [README](../README.md); for the developer/operations workflow see
the [runbook](./runbook.md).

## Purpose

OpenSearch Adapter is a thin, fluent, Laravel-oriented wrapper over the official OpenSearch PHP client. It
turns index and document operations into small builder objects and value objects so application code does not
have to assemble raw request arrays by hand, and wraps responses in read-only result objects.

The package is deliberately minimal: it holds no connection state of its own, ships no configuration, and
delegates all transport, connection management, and Laravel container wiring to its dependency
[`wonsulting/opensearch-client`](https://github.com/wonsulting/opensearch-client).

- Composer package: `wonsulting/opensearch-adapter` (type `library`, MIT).
- PSR-4 root: `OpenSearch\Adapter\` → `src/`.
- Tests namespace: `OpenSearch\Adapter\Tests\` → `tests/`.

## Layer diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Application (e.g. the WonsultingAI app)                                     │
│   resolves IndexManager / DocumentManager from the Laravel container       │
└───────────────────────────────┬────────────────────────────────────────────┘
                                 │  fluent builders + value objects
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ OpenSearch\Adapter  (THIS package)                                         │
│   Managers · Builders · Value objects · Result wrappers · Exceptions       │
│   Client trait: holds a \OpenSearch\Client, clones on connection()          │
└───────────────────────────────┬────────────────────────────────────────────┘
                                 │  ClientBuilderInterface::default() / connection()
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ wonsulting/opensearch-client  (OpenSearch\Laravel\Client\*)                │
│   ServiceProvider (auto-discovered) · ClientBuilder(Interface) · config     │
│   binds ClientBuilderInterface → ClientBuilder as a container singleton     │
└───────────────────────────────┬────────────────────────────────────────────┘
                                 │  builds
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ opensearch-php SDK   \OpenSearch\Client   (opensearch-project/opensearch-php)│
│   HTTP transport to an OpenSearch cluster                                   │
└──────────────────────────────────────────────────────────────────────────┘
```

**The opensearch-php SDK is a transitive dependency**, pulled in through `wonsulting/opensearch-client`; it is
*not* a direct requirement in this package's `composer.json`. At the time of writing, a fresh install resolves
`wonsulting/opensearch-client 2.0.1` → `opensearch-project/opensearch-php 2.6.0`. Because the SDK constraint
floats through opensearch-client, its exact version is owned there, not here.

Everything below the adapter — the Laravel `ServiceProvider`, `ClientBuilderInterface`, the concrete
`ClientBuilder`, and `config/opensearch.client.php` — lives in `wonsulting/opensearch-client`. **This repository
ships no service provider and no config file of its own.**

## Directory map

```
src/
├── Client.php                      trait: holds the \OpenSearch\Client + connection switching
├── Documents/
│   ├── Document.php                value object (id + content)
│   ├── DocumentManager.php         manager: index / delete / deleteByQuery / search
│   └── Routing.php                 value object (documentId → routing value)
├── Indices/
│   ├── Alias.php                   value object
│   ├── Index.php                   value object (name + ?Mapping + ?Settings)
│   ├── IndexManager.php            manager: index & alias lifecycle
│   ├── Mapping.php                 builder (delegates field calls to MappingProperties)
│   ├── MappingProperties.php       builder (magic __call → field types via Str::snake)
│   └── Settings.php                builder (magic __call → settings keys via Str::snake)
├── Exceptions/
│   ├── BulkOperationException.php
│   └── RawResultReadOnlyException.php
└── Search/
    ├── SearchParameters.php        request builder (explicit, non-magic)
    ├── SearchResult.php            result wrapper (root)
    ├── Hit.php  Aggregation.php  Bucket.php  Highlight.php
    ├── Suggestion.php  Explanation.php
    └── RawResult.php               read-only ArrayAccess trait (shared by all wrappers)
```

`tests/` mirrors this under `tests/Unit/{Documents,Indices,Search,Exceptions}` plus
`tests/Extensions/BypassFinalExtension.php`.

## Managers

Both managers are ordinary (non-`final`) classes that gain their constructor and connection handling from the
`Client` trait (see [DI / bootstrap flow](#di--bootstrap-flow)). Every mutating method returns `$this` for
fluent chaining.

### `IndexManager` — `src/Indices/IndexManager.php`

Wraps `$this->client->indices()`.

| Method | Purpose |
| --- | --- |
| `open(string $indexName): self` | Open an index |
| `close(string $indexName): self` | Close an index |
| `exists(string $indexName): bool` | Existence check |
| `create(Index $index): self` | Create from an `Index` value object; emits `body.mappings` / `body.settings` only when non-empty |
| `createRaw(string $indexName, ?array $mapping = null, ?array $settings = null): self` | Create from raw arrays |
| `putMapping(string $indexName, Mapping $mapping): self` | Update mapping from a `Mapping` builder |
| `putMappingRaw(string $indexName, array $mapping): self` | Update mapping from a raw array |
| `putSettings(string $indexName, Settings $settings): self` | Update settings from a `Settings` builder (wrapped in `body.settings`) |
| `putSettingsRaw(string $indexName, array $settings): self` | Update settings from a raw array |
| `drop(string $indexName): self` | Delete an index |
| `putAlias(string $indexName, Alias $alias): self` | Create/update an alias; conditionally emits `is_write_index`, `routing`, `filter` |
| `putAliasRaw(string $indexName, string $aliasName, ?array $settings = null): self` | Alias from raw settings |
| `deleteAlias(string $indexName, string $aliasName): self` | Remove an alias |
| `getAliases(string $indexName): Collection` | `collect(array_keys($raw[$index]['aliases'] ?? []))` |

### `DocumentManager` — `src/Documents/DocumentManager.php`

Wraps `$this->client->bulk()`, `deleteByQuery()`, and `search()`.

| Method | Purpose |
| --- | --- |
| `index(string $indexName, Collection $documents, bool $refresh = false, Routing $routing = null): self` | Bulk-index `Document[]`; attaches per-doc `routing` when `Routing::has($id)`; throws `BulkOperationException` on any item error |
| `delete(string $indexName, array $documentIds, bool $refresh = false, Routing $routing = null): self` | Bulk-delete by IDs; same routing + error handling |
| `deleteByQuery(string $indexName, array $query, bool $refresh = false): self` | Delete-by-query from a raw query array |
| `search(SearchParameters $searchParameters): SearchResult` | Runs `client->search(...)` and wraps the response in a `SearchResult` |

`$refresh` is serialized as the string `'true'` / `'false'`. See
[Bulk error semantics](#bulk-error-semantics) for how failures are surfaced.

## Builders

There are two distinct styles. `MappingProperties`, `Mapping`, and `Settings` use PHP magic (`__call` +
`Str::snake`) to coin OpenSearch keys from method names. `SearchParameters` is deliberately explicit.

### `MappingProperties` — `src/Indices/MappingProperties.php`

The core field-type builder (`final`, `Arrayable`). Each magic call registers one field, keyed by the first
argument, whose `type` is derived from the method name via `Str::snake` (camelCase → snake_case, e.g.
`geoPoint` → `geo_point`, `scaledFloat` → `scaled_float`):

```php
public function __call(string $method, array $arguments): self
{
    $argumentsCount = count($arguments);

    if ($argumentsCount === 0 || $argumentsCount > 2) {
        throw new BadMethodCallException(sprintf('Invalid number of arguments for %s method', $method));
    }

    $property = ['type' => Str::snake($method)];

    if (isset($arguments[1])) {
        $property += $arguments[1];              // merge extra params (boost, null_value, …)
    }

    $this->properties[$arguments[0]] = $property; // arguments[0] = field name

    return $this;
}
```

- `->geoPoint('location')` → `['location' => ['type' => 'geo_point']]`
- `->text('title', ['boost' => 2])` → `['title' => ['type' => 'text', 'boost' => 2]]`
- Passing 0 or >2 arguments throws `BadMethodCallException`.

Two field types are **real methods** (not magic) because they nest sub-properties: `object(string $name,
$parameters = null)` and `nested(string $name, $parameters = null)`. Their `$parameters` may be an array or a
`Closure` that receives a fresh `MappingProperties`; `normalizeParametersWithProperties()` flattens a nested
`properties` sub-builder to an array.

### `Mapping` — `src/Indices/Mapping.php`

The top-level mapping wrapper (`final`, `Arrayable`). It constructs its own `MappingProperties` and uses
Laravel's `ForwardsCalls` trait to proxy any unknown method to it — so all field calls flow through the
snake-case magic above while the outer chain stays on `Mapping`:

```php
public function __call(string $method, array $parameters): self
{
    $this->forwardCallTo($this->properties, $method, $parameters);
    return $this;
}
```

Beyond field delegation it adds real methods: `enableFieldNames()` / `disableFieldNames()` (`_field_names.enabled`),
`enableSource()` / `disableSource()` (`_source.enabled`), `dynamicTemplate(string $name, array $parameters)`
(appends to `dynamic_templates`), and `toArray()` which assembles `_field_names`, `_source`, `properties`, and
`dynamic_templates` (each only when set / non-empty).

### `Settings` — `src/Indices/Settings.php`

Settings builder (`final`, `Arrayable`). Its `__call` accepts **exactly one** argument and keys it by
`Str::snake($method)`:

```php
public function __call(string $method, array $arguments): self
{
    $argumentsCount = count($arguments);

    if ($argumentsCount === 0 || $argumentsCount > 1) {
        throw new BadMethodCallException(sprintf('Invalid number of arguments for %s method', $method));
    }

    $this->settings[Str::snake($method)] = $arguments[0];

    return $this;
}
```

- `->index([...])` → `['index' => [...]]`
- `->analysis([...])` → `['analysis' => [...]]`

### `SearchParameters` — `src/Search/SearchParameters.php`

The search request builder (`final`, `Arrayable`). Unlike the mapping builders it is **not magic** — every
parameter is an explicit, typed, fluent method, so its signatures *are* its schema. Two placement patterns:

- Top-level request keys: `indices(array)` → `index` (comma-joined), `searchType(string)`,
  `preference(string)`, `routing(array)` (comma-joined), `explain(bool = true)`.
- `body.*` keys: `query`, `highlight`, `sort`, `rescore`, `from`, `size`, `suggest`, `source` → `_source`,
  `collapse`, `aggregations`, `postFilter` → `post_filter`, `trackTotalHits` → `track_total_hits`,
  `indicesBoost` → `indices_boost`, `trackScores` → `track_scores`, `minScore` → `min_score`,
  `scriptFields` → `script_fields`.

`toArray()` returns the accumulated params, which `DocumentManager::search()` passes straight to the SDK.

### Discovering valid methods

Because the mapping/settings builders coin keys from method names, there is **no runtime validation** of which
type or setting names are legal — `Str::snake()` will happily turn any method name into a key. Discoverability
comes from three sources:

1. **`@method` PHPDoc catalogs** at the top of `Mapping`, `MappingProperties`, and `Settings`. These are the
   authoritative list of intended field types / settings and drive IDE autocomplete and PHPStan. `Mapping` and
   `MappingProperties` document ~37 field types (`alias`, `binary`, `boolean`, `byte`, `completion`,
   `constantKeyword`, `date`, `dateNanos`, `dateRange`, `denseVector`, `double`, `doubleRange`, `flattened`,
   `float`, `floatRange`, `geoPoint`, `geoShape`, `halfFloat`, `histogram`, `integer`, `integerRange`, `ip`,
   `ipRange`, `join`, `keyword`, `long`, `longRange`, `nested`, `object`, `percolator`, `rankFeature`,
   `rankFeatures`, `scaledFloat`, `searchAsYouType`, `shape`, `short`, `sparseVector`, `text`, `tokenCount`,
   `wildcard`); `Settings` documents `index` and `analysis`.
2. **`SearchParameters`' explicit signatures** — the method list is the schema.
3. **README examples.** The second (params) argument to a field method and the value passed to a `Settings`
   key are passed through verbatim to OpenSearch, so their valid keys are defined by OpenSearch itself, not the
   adapter.

## Value objects

| Class | File | Holds | Public API |
| --- | --- | --- | --- |
| `Document` | `src/Documents/Document.php` | `string $id`, `array $content` | `id(): string`; `content(string $key = null)` — whole array, or a dotted-path lookup via `Arr::get`; `toArray()` → `['id'=>…, 'content'=>…]` |
| `Routing` | `src/Documents/Routing.php` | `array` (documentId → routing value) | `add(string $documentId, string $value): self`; `has(string $documentId): bool`; `get(string $documentId): ?string` |
| `Index` | `src/Indices/Index.php` | `name`, `?Mapping`, `?Settings` | `__construct(string $name, Mapping $mapping = null, Settings $settings = null)`; `name()`, `mapping()`, `settings()` |
| `Alias` | `src/Indices/Alias.php` | `name`, `bool isWriteIndex`, `?array filter`, `?string routing` | `__construct(string $name, bool $isWriteIndex = false, ?array $filter = null, ?string $routing = null)`; `name()`, `isWriteIndex()`, `filter()`, `routing()` |

`Document` implements `Arrayable`; `Routing`, `Index`, and `Alias` do not. `Index` and `Alias` are immutable
aggregates consumed by `IndexManager`.

## Result wrappers

Every search response is returned as a `SearchResult`, from which the other wrappers are reached lazily. All
seven wrappers are `final`, implement `ArrayAccess`, and `use RawResult` (below) — so each is a read-only view
over its slice of the raw response, constructed with a single `array`.

| Class | File | Public API (beyond `ArrayAccess` + `raw()`) |
| --- | --- | --- |
| `SearchResult` | `src/Search/SearchResult.php` | `hits(): Collection<Hit>` (maps `hits.hits`); `total(): ?int` (`hits.total.value`); `suggestions(): Collection` (maps `suggest`, each into `Suggestion`); `aggregations(): Collection<Aggregation>` |
| `Hit` | `src/Search/Hit.php` | `indexName(): string`; `score(): ?float`; `sort(): ?array`; `document(): Document` (from `_id` + `_source`); `highlight(): ?Highlight`; `innerHits(): Collection`; `innerHitsTotal(): Collection`; `explaination(): ?Explanation` |
| `Aggregation` | `src/Search/Aggregation.php` | `buckets(): Collection<Bucket>` |
| `Bucket` | `src/Search/Bucket.php` | `docCount(): int` (`doc_count`, default 0); `key()` (mixed) |
| `Highlight` | `src/Search/Highlight.php` | `snippets(string $field): Collection` |
| `Suggestion` | `src/Search/Suggestion.php` | `text(): string`; `offset(): int`; `length(): int`; `options(): Collection` |
| `Explanation` | `src/Search/Explanation.php` | `value(): float` (default 0); `description(): string`; `details(): Collection<Explanation>` (recursive) |

> **Note:** `Hit::explaination()` is misspelled in the source (it reads the `_explanation` key). Callers must
> use the misspelled method name. Renaming it would be a breaking API change and is out of scope for the
> documentation phase.

### The `RawResult` trait (read-only `ArrayAccess`) — `src/Search/RawResult.php`

Shared by every result wrapper. It supplies the constructor, the four `ArrayAccess` methods, and a `raw()`
accessor. Reads null-coalesce on a miss; **writes and unsets throw** `RawResultReadOnlyException`. The
`#[ReturnTypeWillChange]` attributes keep the wrappers compatible across the PHP 7.4 → 8.x `ArrayAccess`
signature change.

```php
trait RawResult
{
    private array $rawResult;

    public function __construct(array $rawResult)
    {
        $this->rawResult = $rawResult;
    }

    #[ReturnTypeWillChange]
    public function offsetExists($offset) { return isset($this->rawResult[$offset]); }

    #[ReturnTypeWillChange]
    public function offsetGet($offset) { return $this->rawResult[$offset] ?? null; }

    #[ReturnTypeWillChange]
    public function offsetSet($offset, $value) { throw new RawResultReadOnlyException(); }

    #[ReturnTypeWillChange]
    public function offsetUnset($offset) { throw new RawResultReadOnlyException(); }

    public function raw(): array { return $this->rawResult; }
}
```

## Exceptions

The package ships exactly two exceptions (`src/Exceptions/`), both `final`, both extending `\Exception`.

- **`RawResultReadOnlyException`** — no-arg constructor, fixed message `"Raw result can not be modified."`.
  Thrown by the `RawResult` trait on any mutation attempt.
- **`BulkOperationException`** — constructed with the full raw bulk response; builds a human-readable message
  in the constructor and exposes the raw payload:
  - `rawResult(): array` — the untouched raw bulk response (partial successes + per-item errors).
  - `context(): array` — `['rawResult' => …]`, log-context shape.
  - `makeErrorMessage()` (private) — counts `items`; when the first item has an `error`, includes its
    `type` + `reason` (worded `Error:` for a single item, `First error:` for many); always appends guidance to
    call `BulkOperationException::rawResult()` for full details.

## DI / bootstrap flow

The adapter defines no container bindings. Wiring comes entirely from `wonsulting/opensearch-client`, whose
`ServiceProvider` is **auto-discovered** via Laravel package discovery (`extra.laravel.providers`). On
`register()` it merges the `opensearch.client` config and binds the client builder as a singleton:

```php
$this->app->singletonIf(ClientBuilderInterface::class, ClientBuilder::class);
```

The managers get their constructor from the `Client` trait, which type-hints the interface:

```php
public function __construct(OpenSearchClientBuilderInterface $clientBuilder)
{
    $this->clientBuilder = $clientBuilder;
    $this->client = $clientBuilder->default();   // resolve the default \OpenSearch\Client once
}
```

So resolving a manager from the container (`app(IndexManager::class)` or constructor injection) autowires the
bound `ClientBuilderInterface`, and the manager immediately builds and stores the **default** connection's
`\OpenSearch\Client`. Thereafter every operation calls through `$this->client->…`.

`ClientBuilderInterface` (`OpenSearch\Laravel\Client\ClientBuilderInterface`, in opensearch-client) is small:

```php
interface ClientBuilderInterface
{
    public function default(): Client;
    public function connection(string $name): Client;
}
```

### Configuration

Publish the config from the dependency:

```bash
php artisan vendor:publish --provider="OpenSearch\Laravel\Client\ServiceProvider"
```

This creates `config/opensearch.client.php`, which defines a **default connection name** and a map of named
**connections**:

```php
return [
    'default' => env('OPENSEARCH_CONNECTION', 'default'),
    'connections' => [
        'default' => [
            'hosts' => [
                env('OPENSEARCH_HOST', 'localhost:9200'),
            ],
        ],
    ],
];
```

The two config concepts the adapter actually depends on are the default connection (via
`ClientBuilderInterface::default()`) and named connections (via `connection($name)`). The full connection hash
shape (hosts, auth, etc.) is owned by `wonsulting/opensearch-client` — refer to that package's documentation.

### Connection-switching semantics — `src/Client.php`

`connection()` returns a **clone** pointed at the named connection; the original manager is untouched. This
makes connection switching immutable and side-effect-free:

```php
public function connection(string $name): self
{
    $self = clone $this;
    $self->client = $self->clientBuilder->connection($name);
    return $self;
}
```

```php
$other = $documentManager->connection('reporting');   // targets 'reporting'
// $documentManager still targets the default connection
```

## Bulk error semantics

`DocumentManager::index()` and `delete()` submit the **entire** batch in one `bulk` call — they do not fail
fast per item. After the call, if the response's `errors` flag is truthy, they raise a single
`BulkOperationException` carrying the whole response:

```php
$rawResult = $this->client->bulk($params);

if ($rawResult['errors'] ?? false) {
    throw new BulkOperationException($rawResult);
}
```

Because a bulk response can mix successes and failures, catch the exception and call `->rawResult()` to inspect
every item's outcome:

```php
try {
    $documentManager->index('books', $documents);
} catch (BulkOperationException $e) {
    report($e);                 // $e->context() carries the raw payload for logging
    $details = $e->rawResult(); // full per-item successes + errors
}
```

`deleteByQuery()` and `search()` do not use this path — they surface errors through the SDK directly.
