# AGENTS.md — activerecord-mysql2rgeo-adapter

ActiveRecord connection adapter for MySQL with built-in support for spatial (RGeo) column types.

## Repository Overview

- **Language:** Ruby (gem)
- **Purpose:** Extends `Mysql2Adapter` (Rails) with spatial column types backed by the RGeo library
- **Active branch:** `rails-8-1-support` (targets ActiveRecord 8.1 + MySQL 8.0)

## Key Files

| File | Purpose |
|------|---------|
| `lib/active_record/connection_adapters/mysql2rgeo_adapter.rb` | Main adapter class (`Mysql2RgeoAdapter`), type map, spatial column options |
| `lib/active_record/connection_adapters/mysql2rgeo/spatial_column.rb` | `SpatialColumn` — extends `MySQL::Column` with spatial metadata |
| `lib/active_record/connection_adapters/mysql2rgeo/schema_statements.rb` | `new_column_from_field` override to build spatial columns from DB metadata |
| `lib/active_record/connection_adapters/mysql2rgeo/spatial_column_info.rb` | Queries `INFORMATION_SCHEMA` for spatial column metadata (type, srid) |
| `lib/active_record/type/spatial.rb` | `Type::Spatial` — cast type for all spatial columns; `#type` always returns `:geometry` |
| `test/tasks_test.rb` | Schema dump / schema load tests (most relevant to schema.rb regeneration) |
| `test/ddl_test.rb` | Column creation / DDL tests |

## Spatial Type Mapping

All spatial SQL types (`geometry`, `point`, `multipolygon`, `linestring`, etc.) are registered in `Mysql2RgeoAdapter::TYPE_MAP` via `initialize_type_map`. Each maps to `Type::Spatial`, whose `#type` method always returns `:geometry`. This means:

- `column.type` is `:geometry` for **all** spatial columns (geometry, point, multipolygon, etc.)
- `column.sql_type` is the raw MySQL type (e.g. `"multipolygon"`)
- `column.limit` returns `{ type: "multi_polygon", srid: 0 }` (snake_case form of the geometry type name)

Schema dumps always write `t.geometry "col_name", limit: {...}` regardless of the underlying spatial type.

## Rails 8.1 Gotcha — `valid_type?`

Rails 8.1's `SchemaDumper` calls `@connection.valid_type?(column.type)` before dumping each column. The instance `valid_type?` delegates to the **class-level** `Mysql2RgeoAdapter.native_database_types`, which is inherited from `AbstractMysqlAdapter` and does NOT include spatial types. The **instance** `native_database_types` override (which does include spatial types) is not consulted by the class-level `valid_type?`.

**Fix:** Override `valid_type?` at the instance level in `Mysql2RgeoAdapter` so it uses the instance `native_database_types`.

## Running Tests

Tests require a running MySQL 8 instance. See `test/database.yml` (or create `test/database_local.yml`) for connection config.

```bash
# Run all tests (default gemfile)
bundle exec rake test

# Run with Rails 8.1 gemfile
BUNDLE_GEMFILE=gemfiles/ar81.gemfile bundle exec rake test
```

Use `appraisal` to test across Rails versions:
```bash
bundle exec appraisal ar81 rake test
```

## Conventions

- All lib files use `# frozen_string_literal: true`
- Tests use Minitest (`ActiveSupport::TestCase`) with Mocha for mocking
- Column creation tests live in `ddl_test.rb`; schema dump tests in `tasks_test.rb`
