# Command Reference

Comprehensive reference for all Ragdoll CLI commands.

## Global Options

All commands support these global options:

- `--config PATH` - Specify configuration file path
- `--verbose` - Enable verbose output
- `--quiet` - Suppress non-error output
- `--help` - Show help information

## config

Manage Ragdoll configuration settings.

### Usage

```bash
ragdoll config <subcommand> [options]
```

### Subcommands

#### list
Display current configuration:

```bash
ragdoll config list
ragdoll config list --section database
```

#### set
Set configuration values:

```bash
ragdoll config set database.url "postgresql://localhost/ragdoll"
ragdoll config set llm.provider "openai"
```

#### get
Get specific configuration values:

```bash
ragdoll config get database.url
ragdoll config get llm.provider
```

#### validate
Validate configuration:

```bash
ragdoll config validate
```

## health

Check system health and connectivity.

### Usage

```bash
ragdoll health [options]
```

### Options

- `--database` - Check database connectivity only
- `--llm` - Check LLM provider connectivity only
- `--all` - Run all health checks (default)

### Examples

```bash
# Full health check
ragdoll health

# Database only
ragdoll health --database

# With verbose output
ragdoll health --verbose
```

## search

Perform searches across your content.

### Usage

```bash
ragdoll search <query> [options]
```

### Options

- `--limit N` - Number of results to return (default: 10)
- `--threshold FLOAT` - Similarity threshold (0.0-1.0)
- `--format FORMAT` - Output format: json, table, simple (default: table)
- `--include-metadata` - Include document metadata in results

### Examples

```bash
# Basic search
ragdoll search "machine learning algorithms"

# Limit results
ragdoll search "python" --limit 5

# JSON output
ragdoll search "database" --format json

# With metadata
ragdoll search "AI" --include-metadata
```

## update

Add or update documents in the system.

### Usage

```bash
ragdoll update <path> [options]
```

### Options

- `--recursive` - Process directories recursively
- `--force` - Force update even if file hasn't changed
- `--metadata KEY=VALUE` - Add custom metadata
- `--extract-images` - Extract and process images from documents

### Examples

```bash
# Single file
ragdoll update document.pdf

# Directory (recursive)
ragdoll update ./documents --recursive

# With metadata
ragdoll update report.pdf --metadata "author=John Doe" --metadata "department=Research"

# Force update
ragdoll update document.pdf --force
```

## delete

Remove documents from the system.

### Usage

```bash
ragdoll delete <identifier> [options]
```

### Options

- `--by-path` - Delete by file path
- `--by-id` - Delete by document ID
- `--confirm` - Skip confirmation prompt

### Examples

```bash
# Delete by path
ragdoll delete /path/to/document.pdf --by-path

# Delete by ID
ragdoll delete 12345 --by-id

# Skip confirmation
ragdoll delete document.pdf --by-path --confirm
```

## list

List documents and system information.

### Usage

```bash
ragdoll list [type] [options]
```

### Types

- `documents` - List all documents (default)
- `providers` - List available LLM providers
- `embeddings` - List embedding models

### Options

- `--limit N` - Number of items to show
- `--format FORMAT` - Output format: table, json, csv
- `--filter PATTERN` - Filter results by pattern

### Examples

```bash
# List documents
ragdoll list documents

# List providers
ragdoll list providers

# JSON format
ragdoll list documents --format json

# Filter by pattern
ragdoll list documents --filter "*.pdf"
```

## stats

Display system statistics.

### Usage

```bash
ragdoll stats [options]
```

### Options

- `--detailed` - Show detailed statistics
- `--refresh` - Force refresh of cached stats

### Examples

```bash
# Basic stats
ragdoll stats

# Detailed view
ragdoll stats --detailed
```

## status

Show current system status.

### Usage

```bash
ragdoll status [options]
```

### Options

- `--watch` - Continuously monitor status
- `--interval N` - Watch interval in seconds (default: 5)

### Examples

```bash
# Current status
ragdoll status

# Continuous monitoring
ragdoll status --watch

# Custom interval
ragdoll status --watch --interval 10
```

## Exit Codes

- `0` - Success
- `1` - General error
- `2` - Configuration error
- `3` - Connection error
- `4` - Not found error