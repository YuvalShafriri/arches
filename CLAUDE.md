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

### Background and Motivation

The Arches heritage database contains rich, domain-specific data about cultural heritage sites. A representative example is a dataset of 38 historic flour mills (טחנות קמח) across Israel, where each resource record includes:

- **Free-text fields in Hebrew**: cultural assessments (הערכה תרבותית), conservation recommendations (המלצות שימור), historical descriptions (תיאור), spirit of place (רוח המקום), risk descriptions (תיאור סיכונים)
- **Structured fields**: conservation state (מצב השתמרות), threat types (סוג האיומים), management status (סטטוס ניהול), land use (ייעודי קרקע), historical periods (תקופה)
- **Geospatial data**: Points, Polygons, GeometryCollections (site boundaries, historical routes, dams, water channels)
- **References and media**: images, documentation files, external links, bibliography

Heritage professionals need to ask questions that span all of these data types. A single question like "which Crusader-era mills near waterways are at biological risk and lack a full cultural assessment?" touches text analysis, geospatial queries, and structured field filtering simultaneously.

The core insight from our analysis: **neither LLM nor traditional Data Science alone covers all needs**. LLM excels at understanding Hebrew text, extracting meaning, comparing narratives, and generating reports. DS/ML excels at quantitative analysis over large datasets, geospatial computation, and statistical modeling. The system must be flexible enough to use both, and smart enough to choose the right tool for each query.

### Priority 1: Intelligent Analytics Router (DS + LLM)

A flexible analytics system that automatically routes queries to the appropriate engine.

#### Routing Logic

```
User query (natural language, Hebrew/English)
    ↓
  Router (lightweight LLM classifier)
    ├→ Text analysis query?        → LLM pipeline
    ├→ Geospatial/spatial query?   → PostGIS / GeoPandas pipeline
    ├→ Statistical/quantitative?   → Python DS pipeline
    ├→ Image analysis?             → CV model pipeline
    └→ Combined/multi-step?        → DS pipeline → results → LLM narrative
```

#### Query Type Matrix

| Query Type | Engine | Example | Why Not the Other |
|---|---|---|---|
| Text analysis (Hebrew) | LLM | "Extract cultural values from הערכה תרבותית fields" | DS can't understand Hebrew narrative context |
| Text comparison | LLM | "Compare conservation approaches: Nahal Amud mills vs. Nahal Tzipori mills" | Requires semantic understanding of domain text |
| Gap detection | LLM | "Which sites have thin descriptions or missing assessments?" | Needs judgment about content quality, not just field presence |
| Draft generation | LLM | "Generate a cultural assessment draft for טחנת אבו רבאח based on similar sites" | Creative text generation |
| Geospatial clustering | PostGIS/GeoPandas | "Group sites by proximity to water sources" | LLM can't compute distances on coordinates |
| Statistical correlation | Python DS | "Is there a significant correlation between threat type and conservation state?" | Requires chi-square / statistical tests on encoded data |
| Topic modeling | Python DS (NLP) | "What themes emerge across 1000+ cultural assessment texts?" | LLM too expensive at scale; classic NLP more efficient |
| Time-series | Python DS | "How has conservation state changed across documentation periods?" | Requires temporal aggregation and trend analysis |
| Image classification | CV models | "Classify site photos by architectural period" | LLM vision not specialized enough for heritage architecture |
| Combined | DS → LLM | "Analyze spatial patterns of at-risk sites, then generate a policy report" | DS computes, LLM narrates |

#### Practical Estimation for Current Dataset

With ~38 heritage sites and rich text fields, the current workload breaks down approximately:
- **80-90% LLM-addressable**: Most questions are about understanding, comparing, and extracting from Hebrew text
- **10-20% requiring DS**: Geospatial queries, statistical tests, and any future scaling to 1000+ records

This ratio will shift toward DS as the dataset grows.

#### Architecture Approach

Build as Arches plugin(s), leveraging the existing plugin system (ETL modules, functions, search components) without modifying core:

- **Router plugin**: Receives natural language queries, classifies intent, delegates to the appropriate pipeline
- **LLM pipeline plugin**: Connects to LLM API, manages prompts with heritage domain context, handles Hebrew text
- **DS pipeline plugin**: Runs Python-based analysis (pandas, geopandas, scipy, sklearn) on exported Arches data
- **Results aggregator**: Combines outputs from multiple pipelines into a unified response

#### Key Capabilities for Heritage Text Fields

- Extract structured data from free-text fields (cultural values, periods, building materials, stakeholders, historical figures)
- Compare assessments across sites (e.g., mills along Nahal Amud vs. Nahal Na'aman vs. Nahal Tzipori)
- Identify documentation gaps (missing fields, incomplete descriptions, sites without cultural assessment)
- Generate draft assessments for under-documented sites based on patterns in well-documented ones
- Cross-reference Hebrew/Arabic place names and historical references across records
- Summarize conservation recommendations across a group of sites for policy documents

#### DS Pipeline Capabilities

- Geospatial clustering and proximity analysis (PostGIS native queries)
- Statistical correlation across structured fields (threat types, conservation state, management status)
- Topic modeling on large text corpora when dataset scales beyond LLM cost-effectiveness
- Time-series analysis on historical periods and usage changes
- Image classification for heritage site photos (architectural style, period, condition)

### Priority 2: Agent-Based Resource Modeling (Future)

A separate initiative targeting the **schema design phase** rather than data querying. This is an AI co-pilot that assists heritage professionals in designing Arches Graph/Resource Models:

- Accepts natural language descriptions of what needs to be documented (e.g., "I need to record flour mills with their water systems, historical ownership, and construction phases")
- Proposes graph structure (nodes, edges, datatypes, cardinality) aligned with CIDOC-CRM ontology
- Explains modeling tradeoffs: normalization vs. query simplicity, search performance vs. flexibility, granularity vs. usability
- Iterates based on professional feedback
- Creates the finalized model via Arches API

This addresses a real pain point: Arches graph modeling is powerful but complex, requiring understanding of ontologies, nodegroups, cardinality rules, and datatype selection. A co-pilot lowers the barrier significantly.

**Not in current scope** — to be developed after the analytics router is functional.

### LLM Usage Strategy and Cost Control

The system uses LLMs at three distinct layers, each with a different model tier and token budget:

1. **Router (classification)** — A small, fast model (e.g. Haiku-class). Receives the user query, outputs a short structured classification (query type + target pipeline). Minimal tokens — tens, not hundreds. Runs on every query.
2. **Analysis pipeline (text understanding)** — The strongest available model (e.g. Opus, GPT-4.5, Gemini 2.5 Pro — whichever is best for Hebrew at the time). Heritage text analysis in Hebrew demands top-tier language understanding: domain-specific terminology, cultural nuance, morphological complexity. At the current scale (~38 sites), volume is low enough that premium model cost is justified by the quality gap. The system should be **model-agnostic** — configurable to swap between providers as the landscape evolves.
3. **Narrative layer (optional)** — Same strongest-tier model. Takes DS pipeline results (numbers, tables, spatial data) and generates a human-readable summary or report. Quality of Hebrew narrative output matters for professional heritage documents. Runs only for combined queries where DS computes and LLM narrates.

**Cost control principles:**
- **Right-size the model per task**: Router = small/cheap, Analysis + Narrative = best available. The cost asymmetry is by design — classification is high-frequency/low-cost, analysis is low-frequency/high-quality. Never send large payloads to the router.
- **Pre-filter data before sending to LLM**: Query only the relevant records from the database first (SQL/PostGIS), then send the filtered subset to the LLM — never dump the entire dataset into a prompt.
- **Cache repeated patterns**: Common query types (e.g., "summarize site X") produce reusable prompt templates. Cache LLM responses for identical inputs within a session.
- **Set token budgets per query type**: Define max input/output token limits for each pipeline stage. A gap-detection query on 38 sites should not consume the same budget as a full comparative report.
- **Prefer DS when scale grows**: As the dataset scales beyond ~100 records, shift statistical and pattern-detection tasks to DS pipelines (topic modeling, clustering) rather than processing them through the LLM.
- **Monitor and alert**: Track token usage per query type. Flag queries that exceed expected budgets for review.

### Design Principles

- **Plugin-based**: All AI features built as Arches plugins, not core modifications. This ensures upgradeability and separation of concerns
- **Engine-agnostic**: Router pattern allows swapping or adding DS/LLM backends without changing the query interface
- **Hebrew-first**: All text analysis must handle Hebrew (and Arabic) natively, including RTL text, morphological complexity, and domain-specific heritage terminology
- **Transparent**: Users see which engine handled their query and why, building trust in the results
- **Incremental**: Start with LLM-only text analysis (covers 80-90% of current needs), add DS pipelines progressively as the dataset grows and quantitative needs emerge
- **Domain-aware**: Prompts and pipelines are tuned for cultural heritage vocabulary and concepts, not generic
- **Cost-conscious**: Every LLM call is justified — right model, right data scope, right token budget. No "send everything and hope" approach
