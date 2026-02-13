# CLAUDE.md - Arches Project Guide

## Project Overview

Arches is an open-source, web-based, geospatial information system for cultural heritage inventory and management. It is built for recording immovable cultural heritage (archaeological sites, buildings, historic structures, landscapes, heritage ensembles/districts).

- **Version**: 8.0.3
- **License**: AGPL-3.0-or-later
- **Backend**: Django 5.2+ / Python 3.11+
- **Frontend**: Vue 3 + TypeScript (new), Knockout.js (legacy)
- **Database**: PostgreSQL with PostGIS
- **Search**: Elasticsearch 8.x
- **Task queue**: Celery 5.5
- **UI components**: PrimeVue 4.3
- **Build**: Webpack 5

## Quick Reference Commands

### Testing

```bash
# Python unit tests
python manage.py test tests --settings="tests.test_settings"

# Python tests with coverage
python -W default::DeprecationWarning -m coverage run manage.py test tests --settings="tests.test_settings"
coverage report
coverage json

# Frontend tests (Vitest)
npm run vitest
```

### Linting and Formatting

```bash
# Python formatting (Black)
black . --check --exclude=node_modules    # check
black . --exclude=node_modules            # fix

# JavaScript/TypeScript linting (ESLint)
npm run eslint:check                      # check arches/app/src
npm run eslint:fix:all                    # fix arches/app/src

# Prettier formatting
npm run prettier:check                    # check arches/app/src
npm run prettier:fix:all                  # fix arches/app/src

# TypeScript type checking
npm run ts:check
npm run ts:watch                          # watch mode
```

### Build

```bash
npm install                               # install frontend dependencies
npm run build_development                 # dev build (includes eslint + ts check)
npm run build_production                  # production build (includes eslint + ts check)
npm run build_test                        # test build
npm start                                 # webpack dev server with HMR
```

### Database

```bash
python manage.py setup_db                 # initialize database
python manage.py migrate                  # run migrations
python manage.py makemigrations --check   # verify no missing migrations
```

### Other Useful Commands

```bash
python manage.py packages -o import_business_data -s <path>   # import data
python manage.py es index_database                             # reindex Elasticsearch
python manage.py graph publish --graphs <graph_id>             # publish a graph
```

## Repository Structure

```
arches/                          # Main Python package
  settings.py                    # Django settings (~33KB)
  urls.py                        # URL routing (~27KB)
  celery.py                      # Celery configuration
  locale/                        # i18n translations
  app/
    models/
      models.py                  # Core ORM models (~95KB)
      graph.py                   # Graph proxy model (~107KB)
      resource.py                # Resource proxy model (~47KB)
      tile.py                    # Tile proxy model (~34KB)
      card.py                    # Card proxy model (~12KB)
      concept.py                 # Concept model (~68KB)
      migrations/                # 277+ database migrations
    views/                       # Django views (graph, resource, search, tile, api/)
    datatypes/                   # Pluggable data type implementations (~9 files)
    etl_modules/                 # ETL import/export modules (~14 modules)
    functions/                   # Graph functions
    search/
      components/                # Search filter components (term, map, provisional, etc.)
    src/arches/                  # Vue 3 + TypeScript frontend source
    templates/                   # Django templates
    media/
      js/                        # Legacy JS (Knockout, jQuery, RequireJS modules)
      css/                       # Stylesheets
    utils/                       # Utility modules
    permissions/                 # Permission framework
  management/commands/           # 39 custom Django management commands
  install/                       # Installation utilities and project templates
  db/                            # Database utilities, custom migration operations
tests/                           # Python test suite
  base_test.py                   # ArchesTestCase base class
  test_settings.py               # Test Django settings
  fixtures/                      # Test data and resource graphs
  models/                        # Model tests (13 files)
  views/                         # View tests
  search/, permissions/, bulkdata/, commands/, importer/, exporter/, ...
webpack/                         # Webpack configuration
  webpack.common.js              # Shared config (~550 lines)
  webpack.config.dev.js          # Development config
  webpack.config.prod.js         # Production config
cypress/                         # Cypress E2E tests
docker/                          # Docker configuration files
```

## Architecture Concepts

### Graph-Based Data Model

Arches uses a graph-based schema system where data structures are defined as **Graphs** and data instances are **Resources**:

- **Graph** (`GraphModel`): Defines a schema/ontology — the structure of a resource type
- **Node**: Individual data fields within a graph
- **Edge**: Relationships between nodes
- **NodeGroup**: Groups of nodes with shared cardinality
- **ResourceInstance**: An instance of a graph — an actual record
- **Tile** (`TileModel`): Data values within a resource, organized by node groups
- **Card** (`CardModel`): UI components for entering/viewing tile data
- **Widget**: Input controls bound to specific datatypes

### Proxy Models

Core business logic lives in proxy models that extend the base ORM models:

- `Graph` (extends `GraphModel`) — `arches/app/models/graph.py`
- `Resource` (extends `ResourceInstance`) — `arches/app/models/resource.py`
- `Tile` (extends `TileModel`) — `arches/app/models/tile.py`
- `Card` (extends `CardModel`) — `arches/app/models/card.py`

### Plugin System

Arches is extensible through several plugin types:

| Plugin Type | Location | Purpose |
|---|---|---|
| **Datatypes** | `arches/app/datatypes/` | Data validation, serialization, search indexing |
| **Widgets** | Registered via management command | UI input controls for datatypes |
| **Card Components** | Registered via management command | UI containers for data entry |
| **Functions** | `arches/app/functions/` | Graph-wide computed properties/validations |
| **ETL Modules** | `arches/app/etl_modules/` | Data import/export (CSV, Excel, JSON-LD, etc.) |
| **Search Components** | `arches/app/search/components/` | Search filters (term, map, concept, etc.) |
| **Plugins** | Registered via management command | Standalone UI pages |
| **Reports** | Registered via management command | Resource report views |

### Frontend Architecture

The frontend has two layers:

1. **Legacy (Knockout.js)**: Located in `arches/app/media/js/`. Uses RequireJS modules, Knockout bindings, and Django templates. Being phased out.
2. **Modern (Vue 3 + TypeScript)**: Located in `arches/app/src/arches/`. Uses PrimeVue components, single-file `.vue` components, and webpack bundling.

### Internationalization

- Python: Django `gettext_lazy`
- Vue: `vue3-gettext` with `__()` and `_n()` helpers
- Translation extraction: `npm run gettext:extract` / `npm run gettext:compile`
- Locale files in `arches/locale/`

## Code Style and Conventions

### Python

- **Formatter**: Black (line length: 88, the Black default)
- **Style**: PEP 8 with `max-line-length=140` for linting, `ignore: W503, E402, E203`
- **Imports**: Standard library, then Django/third-party, then arches modules
- **Models**: Use UUIDs as primary keys; JSONB fields for flexible data; `I18n_TextField`/`I18n_JSONField` for multilingual text
- **Views**: Mix of class-based views (CBV) and function-based views; API views in `arches/app/views/api/`
- **License header**: All Python files include the AGPL-3.0 license docstring at the top

### JavaScript / TypeScript

- **Formatter**: Prettier (`singleAttributePerLine: true`)
- **Linter**: ESLint with Vue 3, TypeScript, and Prettier plugins
- **Semicolons**: Required (`semi: ["error", "always"]`)
- **TypeScript**: Strict mode enabled, `noEmit: true` (type checking only)
- **Module resolution**: Bundler (webpack)
- **Component style**: Vue 3 single-file components with `<script setup lang="ts">`

### Pre-commit Hooks

Configured in `.pre-commit-config.yaml`:
1. Black (Python formatting)
2. Prettier (JS/TS/Vue formatting, scoped to `arches/app/src`)
3. ESLint (JS/TS/Vue linting, scoped to `arches/app/src`)
4. TypeScript checking (`vue-tsc --noEmit`)

### Git Commit Messages

- Use present tense ("Add feature" not "Added feature")
- Use imperative mood ("Move cursor to..." not "Moves cursor to...")
- Limit the first line to 72 characters
- Reference an issue number (e.g., "improve contributing guidelines docs #1926")

### Branch Naming

- Format: `{ticket_number}_{short_description}` (e.g., `1231_cool_new_feature`)
- Active development branches: `dev/8.1.x`, `dev/8.0.x`, `dev/7.6.x`
- Stable branch: `master`

## CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/main.yml`) runs on pull requests:

1. **Build target branch** — baseline test on Python 3.12
2. **Build feature branch** — tests on Python 3.11, 3.12, 3.13
3. **Check frontend coverage** — fails if coverage decreases
4. **Check Python coverage** — fails if coverage decreases

### CI Steps Per Branch

1. Install system dependencies (Java 8, GDAL, libxml2, libpq)
2. Set up Elasticsearch 8
3. `pip install . --group dev`
4. `python manage.py check` (ensure frontend config files exist)
5. Install Arches applications
6. `npm install`
7. `npm run build_test`
8. `npm run prettier:check`
9. `black . --check --exclude=node_modules`
10. Check for CRLF line endings (disallowed except in `.xml` files)
11. `npm run vitest` (frontend tests + coverage)
12. `python manage.py makemigrations --check` (no missing migrations)
13. Python unit tests with coverage
14. Upload coverage artifacts

## Testing Details

### Python Tests

- **Framework**: Django TestCase via custom `ArchesTestRunner`
- **Base class**: `ArchesTestCase` in `tests/base_test.py`
- **Settings**: `tests/test_settings.py`
- **Fixtures**: `tests/fixtures/` (resource graphs, business data, ontologies)
- **Coverage config**: `.coveragerc` (source: `arches/`, omits migrations/settings/urls)
- **Test directories**: `models/`, `views/`, `search/`, `permissions/`, `bulkdata/`, `commands/`, `importer/`, `exporter/`, `localization/`, `utils/`

### Frontend Tests

- **Framework**: Vitest with jsdom environment
- **Config**: `vitest.config.mts`
- **Setup**: `vitest.setup.mts` (mocks `arches` and `vue3-gettext` modules)
- **Pattern**: `*.test.ts` files co-located with source in `arches/app/src/`
- **Coverage**: Clover XML format to `coverage/frontend/`

### E2E Tests

- **Framework**: Cypress
- **Config**: `cypress.json` (baseUrl: `http://localhost:8000/`)
- **Tests**: `cypress/integration/`

## Key Dependencies

### Python

| Package | Purpose |
|---|---|
| Django 5.2+ | Web framework |
| psycopg2 | PostgreSQL adapter |
| elasticsearch 8.x | Search engine client |
| celery 5.5 | Async task queue |
| django-guardian | Object-level permissions |
| django-oauth-toolkit | OAuth2 support |
| rdflib, PyLD | RDF/linked data support |
| openpyxl | Excel file processing |
| pyproj/pyshp | Geospatial data processing |

### JavaScript

| Package | Purpose |
|---|---|
| Vue 3 | Modern frontend framework |
| PrimeVue 4.3 | UI component library |
| Knockout 3.5 | Legacy frontend framework |
| jQuery 3.6 | Legacy DOM manipulation |
| Mapbox GL 1.13 | Map rendering |
| Leaflet 1.6 | Alternative map rendering |
| D3 7.9 | Data visualization |
| Cytoscape 3.18 | Graph visualization |
| CKEditor 4.22 | Rich text editing |

## Database

- **Engine**: PostgreSQL with PostGIS extension
- **Default DB name**: `arches`
- **Key features**: JSONB fields, UUID primary keys, custom SQL trigger functions, PostGIS geospatial queries
- **Migrations**: 277+ Django migrations in `arches/app/models/migrations/`
- **Custom operations**: `arches/db/migration_operations/` (CreateExtension, CreateFunction, django-migrate-sql-deux)

## Docker

```bash
docker-compose up -d                  # start all services (arches, nginx, postgres, elasticsearch)
```

Services defined in `docker-compose.yml`: arches (app), nginx (reverse proxy), db (PostGIS), elasticsearch, letsencrypt.

Test composition in `docker-compose-test.yml` for CI environments.

## AI Analytics Vision (Heritage Data)

### Context

Arches manages cultural heritage data with rich Hebrew text fields (cultural assessments, conservation recommendations, historical descriptions, risk analysis) alongside structured geospatial and relational data. The analytics layer should serve heritage professionals who need insights from both text and quantitative data.

### Priority 1: Intelligent Analytics Router (DS + LLM)

A flexible analytics system that automatically routes queries to the appropriate engine:

| Query Type | Engine | Example |
|---|---|---|
| Text analysis (Hebrew) | LLM | "Extract cultural values from assessment fields" |
| Geospatial analysis | PostGIS / GeoPandas | "Cluster heritage sites by proximity to water sources" |
| Statistical analysis | Python DS pipeline | "Correlation between threat type and conservation state" |
| Combined | DS pipeline → LLM | "Analyze spatial patterns, then generate narrative report" |

**Architecture approach**: Build as Arches plugin(s), leveraging the existing plugin system (ETL modules, functions, search components) without modifying core. The Router component (lightweight LLM) classifies incoming queries and delegates to the appropriate pipeline.

**Key capabilities for heritage text fields**:
- Extract structured data from free-text fields (cultural values, periods, materials, stakeholders)
- Compare assessments across sites (e.g., mills along different rivers)
- Identify gaps in documentation (missing fields, thin descriptions)
- Generate draft assessments based on existing patterns
- Cross-reference Hebrew/Arabic place names and historical references

**DS pipeline capabilities**:
- Geospatial clustering and proximity analysis (PostGIS)
- Statistical correlation across structured fields
- Topic modeling on large text corpora (1000+ records)
- Time-series analysis on historical periods
- Image classification for heritage site photos (CV models)

### Priority 2: Agent-Based Resource Modeling (Future)

An AI co-pilot that assists in designing Arches Graph/Resource Models:
- Accepts natural language descriptions of what needs to be documented
- Proposes graph structure (nodes, edges, datatypes, cardinality) aligned with CIDOC-CRM ontology
- Explains modeling tradeoffs (normalization, search performance, flexibility)
- Iterates based on feedback
- Creates the model via Arches API

This is a separate initiative from analytics, targeting the schema design phase rather than data querying.

### Design Principles

- **Plugin-based**: All AI features as Arches plugins, not core modifications
- **Engine-agnostic**: Router pattern allows swapping/adding DS or LLM backends
- **Hebrew-first**: All text analysis must handle Hebrew (and Arabic) natively
- **Transparent**: Users see which engine handled their query and why
- **Incremental**: Start with LLM-only text analysis, add DS pipelines progressively
