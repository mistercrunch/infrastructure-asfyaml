# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the infrastructure-asfyaml repository - a Python tool and library for processing Apache Software Foundation (ASF) .asf.yaml configuration files. The project enables Apache projects to configure various infrastructure features like GitHub settings, notifications, website publishing, and CI/CD through a single YAML configuration file.

## Development Commands

### Setup and Installation
```bash
# Install dependencies using Poetry
poetry install

# Install with development dependencies
poetry install --with dev

# Run the CLI tool
poetry run asfyaml-run --help
poetry run asfyaml-validate --help
```

### Testing and Quality Assurance
```bash
# Run all tests
poetry run pytest

# Run tests with verbose output
poetry run pytest -v

# Run specific test file
poetry run pytest tests/validator.py

# Run pre-commit hooks (linting, formatting, type checking)
poetry run pre-commit run --all-files

# Run type checking only
poetry run mypy asfyaml/

# Run linting only  
poetry run ruff check asfyaml/

# Auto-fix linting issues
poetry run ruff check --fix asfyaml/
```

### CLI Usage Examples
```bash
# Process a .asf.yaml file (dry-run mode)
poetry run asfyaml-run --repo /path/to/repo --noop

# Process a .asf.yaml file with GitHub token
poetry run asfyaml-run --repo /path/to/repo --token YOUR_TOKEN

# Validate a .asf.yaml file without processing
poetry run asfyaml-validate --repo /path/to/repo
```

## Architecture Overview

### Core Components

**ASFYamlInstance** (`asfyaml/asfyaml.py:62`): The main orchestrator class that:
- Loads and validates .asf.yaml configuration files
- Manages feature discovery and instantiation
- Coordinates execution of all enabled features
- Handles environment-based configuration (production, testing, etc.)

**ASFYamlFeature** (`asfyaml/asfyaml.py:230`): Base class for all features with:
- Plugin-style architecture using `__init_subclass__` for auto-registration
- Priority-based execution ordering (0=highest, 10=lowest, default=5)
- Environment-aware feature loading
- Schema validation support via StrictYAML

**Repository & DataObjects** (`asfyaml/dataobjects.py`): Core data structures for:
- Repository metadata (name, project, privacy status)
- Git commit and changeset processing
- ASF-specific mappings (LDAP to hostname, etc.)

### Feature System Architecture

Features are organized in `asfyaml/feature/` with each feature as a self-contained module:

- **GitHub Features** (`asfyaml/feature/github/`): Repository settings, branch protection, collaborators, etc.
- **Website Publishing** (`asfyaml/feature/website_*.py`): Staging and production website deployment
- **Notifications** (`asfyaml/feature/notifications.py`): Email notification routing
- **Build Systems** (`asfyaml/feature/pelican.py`, `jekyll.py`): Static site generators

### Key Design Patterns

1. **Plugin Architecture**: Features auto-register via metaclass, enabling easy extensibility
2. **Environment Separation**: Production vs testing vs noop modes for safe testing
3. **Priority-Based Execution**: Features run in configurable order (notifications first, cleanup last)
4. **Schema Validation**: StrictYAML ensures configuration correctness before processing
5. **Exception Handling**: Comprehensive error reporting with repository/branch/feature context

### Configuration Flow

1. `.asf.yaml` loaded and parsed by StrictYAML
2. Environment determination (production + optional overlays)
3. Feature discovery and filtering by environment
4. Individual feature schema validation
5. Feature instantiation with validated YAML subset
6. Execution in priority order with exception handling

## Working with Features

### Creating a New Feature
```python
# In asfyaml/feature/myfeature.py
from asfyaml.asfyaml import ASFYamlFeature
import strictyaml

class MyFeature(ASFYamlFeature, name="myfeature", priority=5):
    schema = strictyaml.Map({
        "enabled": strictyaml.Bool(),
        "config": strictyaml.Str()
    })
    
    def run(self):
        if self.yaml.enabled:
            # Access other features via self.instance.features
            # Access repo info via self.repository
            pass
```

### Testing Features
Features are tested in `tests/` using pytest. Test files follow the pattern of feature names (e.g., `tests/github_*.py` for GitHub features).

## Repository Structure

```
asfyaml/
├── __init__.py          # Package initialization
├── asfyaml.py          # Core orchestration classes  
├── cli.py              # Command-line interface
├── dataobjects.py      # Repository and git data structures
├── envvars.py          # Environment variable handling
├── mappings.py         # ASF-specific project mappings
├── validators.py       # Custom validation functions
└── feature/            # Feature implementations
    ├── github/         # GitHub-specific features
    ├── notifications.py # Email routing
    ├── pelican.py      # Pelican static site generator
    ├── jekyll.py       # Jekyll static site generator
    ├── website_*.py    # Website publishing features
    └── testfeature.py  # Example/test feature

tests/                  # Test suite
├── conftest.py        # Test configuration
├── helpers.py         # Test utilities
└── *.py              # Feature-specific tests
```

## Dependencies and Environment

- **Python**: >=3.10 required
- **Core**: PyYAML, StrictYAML, requests, asfpy (ASF Python utilities)
- **GitHub Integration**: PyGithub for API interactions
- **Development**: mypy, ruff, pytest, pre-commit
- **Build System**: Poetry for dependency management

## GitHub Integration

Many features interact with GitHub via PyGithub. When developing GitHub features:
- Always check for `GH_TOKEN` environment variable
- Use the `noop` environment for testing without actual API calls
- GitHub API rate limiting considerations in production
- Repository settings are applied via GitHub's REST API