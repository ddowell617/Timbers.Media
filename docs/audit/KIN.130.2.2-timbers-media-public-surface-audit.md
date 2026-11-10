# KIN.130.2.2 — Timbers Media Public Surface Audit — module detail

> **Moved from Timbers Infrastructure** `docs/roadmap/implementation/KIN.130.2.2-timbers-media-public-surface-audit.md` at commit `a8e451dec75eb0266bc5339858a3e94d771fead8` (blob `d2cb3a27be6a6110d515a83fa208de5c1ebc41bc`) under GF.200.3 step 4 (audit finding G7). The sections below are copied verbatim, with their original numbers. The gate record (purpose, audited baseline, ownership and boundary conclusions, gates and decisions) stays in Timbers Infrastructure: [KIN.130.2.2](https://github.com/ddowell617/timbers-infrastructure/blob/main/docs/roadmap/implementation/KIN.130.2.2-timbers-media-public-surface-audit.md). Section references to numbers not included here point to that record. This is a dated static audit; it is not re-verified against current source.

## 4. Implemented module surface

### 4.1 Module declaration

| Surface | Verified value | Compatibility significance |
| --- | --- | --- |
| Extension name | `timbers_media_metadata` | Module machine name is a public install and dependency contract. |
| Display name | `Timbers Media - Metadata` | Administrative extension listing label. |
| Package | `Timbers Media` | Groups the module in Drupal extension administration. |
| Core requirement | `^11.0` | Current declared Drupal support range. |
| Dependency | `drupal:file` | Required runtime module dependency. |

The module intentionally does not depend on Media, Kinnect Platform, Kinnect MIR, Media Enhance, EXIF contrib, Flysystem, or a product-specific field module.

### 4.2 Route and navigation surface

Source verification found no:

- `*.routing.yml` files;
- `*.links.menu.yml` files;
- `*.links.task.yml` files;
- `*.links.action.yml` files;
- permissions YAML;
- controllers or forms.

Therefore, the current Timbers Media metadata capability has no administration page, menu link, local task, local action, or HTTP route.

This absence is architecturally significant. Consumer administration UX must remain owned by the consuming project unless a separately approved reusable Timbers Media administration capability is documented and validated.

### 4.3 Service IDs

The following service IDs are source verified and must be treated as current public container contracts:

| Service ID | Implementation or alias | Purpose |
| --- | --- | --- |
| `logger.channel.timbers_media_metadata` | Logger channel | Module-specific logging. |
| `timbers_media_metadata.gps_coordinate_normalizer` | `GpsCoordinateNormalizer` | Converts EXIF GPS components to decimal coordinates. |
| `timbers_media_metadata.exif_metadata_reader` | `ExifMetadataReader` | Reads local or stream-wrapper-backed EXIF metadata. |
| `timbers_media_metadata.exif_metadata_extractor` | `ExifMetadataExtractor` | Extracts and normalizes supported EXIF values. |
| `timbers_media_metadata.metadata_extractor` | Alias to EXIF extractor | Default provider-neutral extractor entry point. |
| `timbers_media_metadata.runtime_validator` | `MetadataRuntimeValidator` | Verifies safe failure for unreadable sources. |
| `timbers_media_metadata.field_mapper` | `DefaultFieldMapper` | Maps normalized keys to consumer-selected field names. |
| `timbers_media_metadata.write_policy` | `DefaultMetadataWritePolicy` | Applies conservative generic write eligibility rules. |

The alias `timbers_media_metadata.metadata_extractor` is the preferred consumer entry point. Consumers should not depend on the EXIF concrete implementation unless an implementation-specific dependency is intentional and documented.

### 4.4 Interface aliases and PHP contracts

The following interface aliases are source verified:

| Interface | Container alias |
| --- | --- |
| `Drupal\timbers_media_metadata\Contract\MetadataExtractorInterface` | `timbers_media_metadata.metadata_extractor` |
| `Drupal\timbers_media_metadata\Contract\FieldMapperInterface` | `timbers_media_metadata.field_mapper` |
| `Drupal\timbers_media_metadata\Contract\MetadataWritePolicyInterface` | `timbers_media_metadata.write_policy` |

Public method signatures:

```php
MetadataExtractorInterface::extract(string $uri): MetadataExtractionResult
```

```php
FieldMapperInterface::map(MetadataExtractionResult $metadata, array $mapping): array
```

```php
MetadataWritePolicyInterface::mayWrite(
  string $field_name,
  mixed $existing_value,
  mixed $new_value,
  array $context = [],
): bool
```

These interfaces, their namespaces, method signatures, parameter semantics, and return types are compatibility-sensitive public surfaces.

### 4.5 Result DTO contract

`MetadataExtractionResult` is a final immutable value object with these constructor properties:

| Property | Type | Meaning |
| --- | --- | --- |
| `sourceUri` | `string` | Original source URI supplied by the consumer. |
| `values` | `array<string, mixed>` | Normalized provider-neutral metadata values. |
| `warnings` | `list<string>` | Non-fatal extraction warnings. |

Source-verified public methods:

- `MetadataExtractionResult::empty(string $source_uri, string $warning = ''): self`
- `get(string $key): mixed`
- `has(string $key): bool`
- `isEmpty(): bool`
- `toArray(): array`

The serialized `toArray()` structure is:

```text
source_uri
values
warnings
```

This array shape is a public compatibility obligation for consumers, validation evidence, and any future API adapter that serializes the DTO.

### 4.6 Normalized metadata key registry

The current normalized key set is source verified:

| Family | Keys |
| --- | --- |
| Image | `image.width`, `image.height`, `image.orientation`, `image.quality` |
| File | `file.size_mb`, `file.modified` |
| Capture | `capture.created`, `capture.created_source`, `capture.created_extracted_at`, `capture.digitized` |
| Camera/source | `camera.make`, `camera.model`, `source.software` |
| Descriptive | `media.author`, `media.title`, `media.description`, `media.keywords`, `media.comments` |
| Location | `gps.latitude`, `gps.longitude` |

Normalized key names and value semantics should be treated as a versioned contract. Renaming, removing, or changing types requires consumer inventory, compatibility planning, and release notes.

Current notable value semantics include:

- EXIF dates normalized to `YYYY-MM-DDTHH:MM:SS` without timezone information;
- `capture.created_source` currently emitted as `exif.datetimeoriginal`;
- `capture.created_extracted_at` generated in UTC format without a timezone suffix;
- file size emitted in megabytes rounded to two decimal places;
- GPS coordinates emitted as decimal floats;
- empty, null, and empty-array values removed from normalized output.

These semantics should be validated before Kinnect consumer adoption because downstream field types and provenance policy may require a richer representation.

## 5. Read behavior and runtime dependencies

### 5.1 Source URI contract

The extractor accepts a Drupal-compatible source URI string. It does not accept Media entities, File entities, field items, or Kinnect-specific DTOs.

The approved adoption boundary remains:

```text
source URI
  -> Timbers Media metadata extractor
  -> MetadataExtractionResult
  -> consumer-owned mapping, policy, write, queue, repair, and UX
```

### 5.2 Stream-wrapper handling

The EXIF reader:

1. obtains the source URI scheme from Drupal's stream-wrapper manager;
2. reads local stream wrappers through a real path;
3. copies non-local/virtual sources to a deterministic `temporary://` URI;
4. extracts metadata from the local temporary copy;
5. attempts to delete the temporary copy in a `finally` block.

The deterministic temporary URI includes a SHA-256 hash of the source URI and basename. Changes to this behavior may affect collision handling, concurrent extraction, cleanup, and security assumptions and therefore require explicit validation.

### 5.3 EXIF providers

The reader uses this fallback order:

1. Drupal EXIF contrib integration when `Drupal\exif\ExifFactory` exists;
2. PHP native `exif_read_data()` when available;
3. structured warning result when neither source succeeds.

EXIF contrib is an optional runtime integration, not a declared module dependency.

### 5.4 Failure behavior

Ordinary unsupported, unavailable, missing, or unreadable source conditions must return a `MetadataExtractionResult` with warnings instead of throwing.

The runtime validator confirms this contract by extracting from a deliberately missing `public://` URI and returning a serializable status summary.

This safe-failure behavior is a migration gate for all consumers.

### 5.5 Logging

Read failures are written to `logger.channel.timbers_media_metadata` at warning level. Consumer deployments should account for potential warning volume during bulk processing and avoid using the runtime validator as a high-frequency health probe.

## 6. Mapping and write-policy contracts

### 6.1 Field mapper

The default mapper accepts a consumer-supplied mapping shaped as:

```text
target field name -> normalized metadata key
```

Only mappings with string field names and string normalized keys are considered. A mapped field is returned only when the extraction result contains the requested key.

The mapper does not:

- inspect Drupal field definitions;
- validate target field types;
- load or write entities;
- transform values for site-specific field storage;
- know media bundles or Kinnect field names.

### 6.2 Default write policy

The default policy allows a write when:

- the new value is not null or an empty string; and
- the existing value is null, an empty string, or an empty array;

or when the consumer explicitly supplies `['overwrite' => TRUE]` in the context.

The policy does not itself perform a write. Consumers remain responsible for entity access, field existence, cardinality, validation, translation, revision, audit, provenance, and preservation rules.

Because `field_name` is not currently used by the default implementation, consumers must not infer that the policy validates field-specific behavior.

## 7. Absent public surfaces

Static inspection confirms that the audited module currently defines no:

- permissions;
- roles;
- routes or HTTP APIs;
- menus, tasks, or actions;
- configuration schema or default configuration;
- State API keys;
- key-value collections;
- cache bins or cache tags owned by the module;
- content or configuration entities;
- database tables or schema hooks;
- install/update/uninstall hooks;
- event subscribers;
- plugins or tagged extension providers;
- queues or queue workers;
- cron handlers;
- batch operations;
- Drush commands;
- JavaScript, CSS, Twig, or theme hooks;
- bundled media fields or media bundles;
- Composer package manifest in the audited snapshot.

These absences must not be misread as permanently forbidden surfaces. Any future addition requires its own ownership and compatibility review before implementation.

## 8. Consumer surface and adoption obligations

### 8.1 Known intended consumer

Kinnect Platform is the documented first consumer candidate. Adoption is planned but not implemented by the audited Timbers Media snapshot.

The Kinnect adapter must remain thin and Kinnect-owned. It may:

- resolve a source URI;
- call `timbers_media_metadata.metadata_extractor`;
- receive or wrap `MetadataExtractionResult`;
- map normalized keys to Kinnect-owned fields;
- invoke Kinnect-owned write policy and workflow logic.

It must not cause Timbers Media to depend on:

- Kinnect field names or media bundles;
- Media Enhance service tags;
- Kinnect queues;
- repair or readiness dashboards;
- recognition or MIR;
- privacy, provenance, or preservation policy;
- enterprise reports or UX.

### 8.2 Adoption validation matrix

Before any consumer write path is enabled, validate:

| Scenario | Required result |
| --- | --- |
| Missing source | Warning result; no exception; no write. |
| Readable JPEG with EXIF | Supported normalized values returned. |
| JPEG without EXIF | Empty or warning result; no hard failure. |
| GPS-bearing image | Decimal latitude and longitude returned. |
| Non-local stream wrapper | Temporary-copy path succeeds and cleans up. |
| Unsupported file type | Warning result; no write. |
| Existing target value | Conservative default prevents overwrite unless explicitly authorized. |
| Unknown normalized key | Mapper omits the target value. |

### 8.3 Versioning requirement

Before broad consumer adoption, Timbers Media needs a documented package and release strategy covering:

- Composer package identity;
- semantic-versioning expectations;
- stable versus experimental APIs;
- normalized-key deprecation policy;
- interface and service-ID compatibility;
- Drupal core support policy;
- changelog and upgrade guidance.

The absence of a Composer manifest in the audited snapshot is a distribution/readiness gap, not permission to invent consumer installation instructions in KIN.130.

## 9. Compatibility register

| Surface | Classification | Current obligation |
| --- | --- | --- |
| Module machine name | Public extension contract | Preserve or provide an explicit migration path. |
| Service IDs | Public container contract | Preserve aliases during implementation substitution. |
| Interface namespaces and signatures | Public PHP API | Semantic-version and deprecate before breaking. |
| `MetadataExtractionResult` constructor and array shape | Public DTO contract | Preserve serialization and warning semantics. |
| Normalized metadata keys | Public data contract | Version additions and deprecations; validate types. |
| Source URI input | Public invocation contract | Do not replace with entity-only APIs. |
| Safe-warning behavior | Runtime compatibility contract | Ordinary extraction failures must remain non-fatal. |
| Temporary-copy behavior | Runtime/security contract | Validate concurrency, cleanup, permissions, and remote wrappers before change. |
| Field mapping shape | Public adapter contract | Preserve target-field-to-normalized-key semantics or version it. |
| Default write behavior | Policy contract | Do not silently broaden overwrite behavior. |
| Module dependency on `drupal:file` | Package/runtime contract | Preserve until an approved dependency change. |
| Optional EXIF contrib integration | Graceful integration contract | Module must continue to operate without EXIF contrib. |

## 10. Findings requiring follow-up

### TMM-PS-001 — No Composer package manifest in snapshot

The repository cannot yet be treated as a source-verified Composer-distributed package based on this snapshot alone. Confirm whether packaging exists outside the snapshot or remains future work.

### TMM-PS-002 — Normalized timestamps lack timezone semantics

Capture timestamps are normalized without an offset, while extraction time is generated in UTC but lacks a `Z` suffix. Consumer adoption must document interpretation before persistent writes.

### TMM-PS-003 — Normalized key registry is documentation-based rather than formally versioned

The key list is implemented and documented but has no machine-readable schema or explicit version identifier. Do not add such a mechanism in this audit; record it for contract governance.

### TMM-PS-004 — Concrete EXIF reader is publicly registered

Consumers could depend directly on concrete service IDs even though the default extractor alias is preferred. Future implementation changes must preserve or deprecate those IDs intentionally.

### TMM-PS-005 — Optional EXIF contrib integration uses static factory access

The reader detects and invokes `ExifFactory` statically. This is a current implementation detail with testability and dependency-discovery implications. Do not refactor it under KIN.130.

### TMM-PS-006 — Deterministic temporary filename concurrency requires validation

Concurrent reads of the same non-local URI may target the same temporary URI. Consumer-scale validation should verify copy/replace and cleanup behavior before high-volume adoption.

### TMM-PS-007 — Logger volume requires bulk-processing validation

Every ordinary read failure logs a warning. Large unsupported or inaccessible batches could create excessive logs. Validate operational behavior without weakening the safe-warning contract.

### TMM-PS-008 — Generic write policy is not a product write authorization

The default policy only evaluates empty/overwrite semantics. Kinnect must layer access, provenance, preservation, audit, translation, and workflow policy outside Timbers Media.
