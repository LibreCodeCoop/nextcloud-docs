# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This repository contains Portuguese documentation for Nextcloud in high-availability distributed environments using Sphinx. The documentation covers architecture, deployment, and maintenance of Nextcloud clusters across multiple datacenters.

**Key Topics:**
- Distributed systems architecture (primary/secondary datacenters)
- Database replication (PostgreSQL with Patroni)
- File synchronization (GlusterFS, GarageS3)
- Distributed coordination (etcd)
- Load balancing, backup (Duplicati), and monitoring (Zabbix)

## Build and Development Commands

### Building Documentation

```bash
# From the docs/ directory
cd docs
make html       # Build HTML documentation
make clean      # Clean build artifacts
make help       # Show all available targets
```

### Local Development Setup

```bash
# Install dependencies
pip install -r docs/requirements.txt

# Dependencies:
# - sphinx==7.1.2
# - sphinx-rtd-theme==1.3.0rc1
```

### Viewing Built Documentation

After building, open `docs/build/html/index.html` in a browser.

## Project Structure

```
nextcloud-docs/
├── docs/
│   ├── source/          # RST documentation source files
│   │   ├── index.rst    # Main documentation index with toctree
│   │   ├── conf.py      # Sphinx configuration
│   │   ├── inicio.rst   # Architecture overview
│   │   ├── banco.rst    # Database setup
│   │   ├── banco-replica.rst  # Database replication
│   │   ├── arquivos-sinc.rst  # File synchronization
│   │   ├── backup.rst         # Backup strategies
│   │   ├── nextcloud-instalacao.rst  # Installation guide
│   │   ├── nextcloud-update.rst      # Update procedures
│   │   └── assets/      # Images and diagrams
│   ├── build/           # Generated documentation (gitignored)
│   ├── Makefile         # Build commands
│   └── requirements.txt # Python dependencies
├── .readthedocs.yaml    # Read the Docs configuration
└── pyproject.toml       # Python project metadata
```

## Architecture

**Documentation Tool:** Sphinx with Read the Docs theme  
**Format:** reStructuredText (.rst files)  
**Language:** Portuguese (Brazilian)  
**Deployment:** Read the Docs (configured via .readthedocs.yaml)

### Documentation Organization

The docs are organized into sections via toctree:
1. **Introdução** (Introduction): System overview and setup checks
2. **Configuração do Ambiente** (Environment Setup): Host, DNS, reverse proxy
3. **Banco de Dados** (Database): Setup and replication
4. **Armazenamento** (Storage): File sync and backup
5. **Nextcloud**: Installation and updates

### Key Architectural Decisions Documented

- **File Sync:** GlusterFS (decentralized) or GarageS3 (distributed by design)
- **Database:** PostgreSQL with native replication + Patroni for failover
- **Coordination:** etcd for distributed consensus
- **Backup:** Duplicati to S3-compatible storage (Wasabi or MinIO)
- **Monitoring:** Zabbix
- **Automation:** Ansible for infrastructure as code

## Working with RST Files

### Common reStructuredText Patterns

```rst
# Titles use underlines
=============
Section Title
=============

Subsection
----------

# Links
`Link text <URL>`_

# Code blocks
::

   code here

# Images
.. image:: filename.png
   :alt: description
   :align: center

# Notes and warnings
.. note::
   Content

.. warning::
   Content

# Toctree for navigation
.. toctree::
   :maxdepth: 2
   :caption: Section Name
   
   file1
   file2
```

### When Editing Documentation

- Use 3 spaces for indentation in directives
- Place images in `docs/source/assets/`
- Add new pages to the toctree in `index.rst`
- Test builds locally before committing: `cd docs && make html`
- The project uses Sphinx extensions: duration, doctest, autodoc, autosummary, intersphinx

## Read the Docs Integration

The `.readthedocs.yaml` configuration:
- Ubuntu 22.04 build environment
- Python 3.10
- Auto-installs from `docs/requirements.txt`
- Uses `docs/source/conf.py` for Sphinx config

Changes pushed to the repository will automatically trigger Read the Docs builds.
