# Ragdoll Command Line Interface

The Ragdoll CLI provides a powerful command-line interface for interacting with your Ragdoll RAG system. This tool allows you to manage documents, perform searches, monitor system health, and configure your setup directly from the terminal.

## Overview

The `ragdoll` command-line tool is part of the `ragdoll-cli` gem and provides access to all core Ragdoll functionality without needing to write code or use a web interface.

## Key Features

- **Document Management**: Upload, update, and delete documents
- **Search Operations**: Perform semantic and hybrid searches
- **System Monitoring**: Check health, stats, and status
- **Configuration Management**: Set up and manage LLM providers, database connections
- **Bulk Operations**: Process multiple files and directories

## Quick Start

```bash
# Install the CLI
gem install ragdoll-cli

# Check health
ragdoll health

# Search for content
ragdoll search "your query here"

# Upload a document
ragdoll update /path/to/document.pdf
```

## Available Commands

- `config` - Manage configuration settings
- `delete` - Remove documents from the system
- `health` - Check system health and connectivity
- `list` - List documents and system information
- `search` - Perform searches across your content
- `stats` - Display system statistics
- `status` - Show current system status
- `update` - Add or update documents

## Getting Help

Use the `--help` flag with any command to see detailed usage information:

```bash
ragdoll --help
ragdoll search --help
```

## Next Steps

- [Install the CLI](installation.md) - Get the command line tool set up
- [Command Reference](commands.md) - Detailed documentation for all commands
- [Configuration](configuration.md) - Set up your environment
- [Examples](examples.md) - Common usage patterns and workflows