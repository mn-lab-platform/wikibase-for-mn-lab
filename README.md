# Mare Nostrum Lab Wikibase instance

This repository contains the deployment and configuration for the Mare Nostrum Lab Wikibase instance. It provides a Docker-based environment for running Wikibase together with the supporting services needed for structured data editing, querying, and publishing.

The project is based on the Wikibase Suite stack and is intended for deployment for Mare Nostrum Lab Platform. It extends the standard suite with specialized components, including the Thesaurus management service for semantic vocabularies.

### Index

* [Architecture & Services](#architecture--services)
* [Requirements](#requirements)
* [Quick Start](#quick-start)
* [1. Environment Configuration (`.env`)](#1-environment-configuration-env)
* [2. Building and Starting Services](#2-building-and-starting-services)

* [Managing Containers](#managing-containers)
* [UI & MediaWiki Customizations](#ui--mediawiki-customizations)
* [Thesaurus Code Documentation & Architecture](#thesaurus-code-documentation--architecture)
* [Testing](#testing)

---

## Architecture & Services

This deployment builds upon the upstream [Wikibase Suite Deploy](https://github.com/wmde/wikibase-release-pipeline) and orchestrates the following core components:

* **Wikibase (MediaWiki):** Core semantic platform engine with custom extensions.
* **MariaDB:** Relational storage for MediaWiki and Wikibase core entities.
* **Elasticsearch:** Full-text search engine powering internal wiki queries.
* **WDQS (Wikidata Query Service):** Blazegraph-backed SPARQL endpoint for graph traversals.
* **WDQS Frontend & Updater:** SPARQL query UI and continuous RDF sync daemon.
* **QuickStatements:** Batch data ingestion and automated entity modification tool.
* **Thesaurus Module:** Custom terminology and concept hierarchy management service.
* **Traefik:** Reverse proxy handling routing, SSL termination, and domain management.

---

## Requirements

* **Docker:** 22.0 or higher
* **Docker Compose:** v2 (2.10 or higher)
* **Git**

---

## Quick Start

Setting up the platform requires only two steps: preparing your `.env` configuration and starting the containers.

### 1. Environment Configuration (`.env`)

Navigate to the `deploy/` directory and generate your working `.env` file from the provided template:

```bash
cd deploy
cp template.env .env

```

Open `.env` in your text editor and configure the required variables:

| Variable | Description | Requirement / Example |
| --- | --- | --- |
| `WIKIBASE_HOST` | Primary domain for the Wikibase web interface | e.g. `thesaurus.mn.cenagis.edu.pl` |
| `WDQS_HOST` | Domain for the SPARQL Query Service | e.g. `thesaurus.mn.cenagis.edu.pl/query/` |
| `MW_ADMIN_NAME` | MediaWiki administrator username | **Required** |
| `MW_ADMIN_PASS` | MediaWiki administrator password (min. 10 chars) | **Required** (Set a strong password) |
| `MW_ADMIN_EMAIL` | Administrator contact email | **Required** |
| `DB_PASS` | MariaDB root password | **Required** (Set a secure password) |
| `SPARQL_ENDPOINT_URL` | Internal or public SPARQL endpoint URL | `thesaurus.mn.cenagis.edu.pl/sparql` |
| `THESAURUS_SECRET_KEY` | Secret key for Thesaurus module tokens/auth | **Required** (Set a random string) |

> [!WARNING]
> Never commit the generated `.env` file to git. Keep all secrets and production credentials restricted to your deployment target.

---

### 2. Building and Starting Services

From within the `deploy/` directory, build all customized images and launch the entire stack in detached mode:

```bash
docker compose up --build -d
```

Check the health status of all containers:

```bash
docker compose ps
```

Monitor logs in real time:

```bash
docker compose logs -f
```

---

## Managing Containers

### Stopping the Stack

To stop services while keeping database volumes intact:

```bash
docker compose down
```

To stop services and completely purge all persistent volumes:

```bash
docker compose down -v
```

---

## UI & MediaWiki Customizations

The `deploy/` directory includes reference documentation and source snippets for custom frontend behavior and form configurations tailored for this platform:

* **`deploy/MediaWiki-Common.css.md`**: Custom CSS styling applied across MediaWiki/Wikibase interface themes.
* **`deploy/MediaWiki-Common.js.md`**: Client-side JavaScript enhancements, UI hooks, and custom entity view behavior.
* **`deploy/project-Cradle.md`**: Custom Cradle schema templates and form definitions used for entity creation and metadata entry.

---

## Thesaurus Code Documentation & Architecture

### Repository Structure

```text
.
├── app/                  # Thesaurus service source code
│   ├── api/              # REST routes and API controllers
│   ├── core/             # Application config, environment loaders, security
│   ├── services/         # SPARQL adapters, SKOS mapping logic, data sync
│   ├── models/           # Data models and schemas (Pydantic / ORM)
│   └── tests/            # Unit and integration test suite
├── deploy/               # Deployment root
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── template.env      # Environment configuration template
│   ├── MediaWiki-Common.css.md
│   ├── MediaWiki-Common.js.md
│   └── project-Cradle.md
└── README.md

```

### Key Modules & Logic Flow

* **SPARQL Adapter (`app/services/`):** Generates and executes SPARQL queries against WDQS. It handles taxonomy traversal, parsing SKOS relationships (`skos:broader`, `skos:narrower`, `skos:related`), and concept extraction.
* **API Controller Layer (`app/api/`):** Exposes search endpoints, tree navigation APIs, and automated batch vocabulary import pipelines.
* **Local Persistence & Caching (`app/models/`):** Manages local entity metadata storage and query result caching to optimize latency and reduce load on the Blazegraph backend.

---

## Testing

Run the test suite inside the running Thesaurus application container:

```bash
docker compose exec app pytest
```