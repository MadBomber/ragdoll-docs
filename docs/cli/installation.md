# CLI Installation

This guide covers installing and setting up the Ragdoll command-line interface.

## Prerequisites

- Ruby 3.2.0 or higher
- PostgreSQL with pgvector extension
- Access to a running Ragdoll system

## Installation

### Install from RubyGems

```bash
gem install ragdoll-cli
```

### Install from Source

```bash
git clone https://github.com/MadBomber/ragdoll-cli.git
cd ragdoll-cli
bundle install
rake install
```

### Verify Installation

```bash
ragdoll --version
ragdoll health
```

## Configuration

The CLI requires configuration to connect to your Ragdoll system:

### Environment Variables

```bash
export RAGDOLL_DATABASE_URL="postgresql://user:pass@localhost/ragdoll"
export RAGDOLL_LLM_PROVIDER="openai"
export RAGDOLL_API_KEY="your-api-key"
```

### Configuration File

Create `~/.ragdoll/config.yml`:

```yaml
database:
  url: "postgresql://user:pass@localhost/ragdoll"
  
llm:
  provider: "openai"
  api_key: "your-api-key"
  model: "gpt-4"

logging:
  level: "info"
  file: "~/.ragdoll/logs/ragdoll.log"
```

### Using ragdoll config

```bash
# Set database URL
ragdoll config set database.url "postgresql://user:pass@localhost/ragdoll"

# Set LLM provider
ragdoll config set llm.provider "openai"
ragdoll config set llm.api_key "your-api-key"

# View current configuration
ragdoll config list
```

## Troubleshooting

### Connection Issues

```bash
# Test database connectivity
ragdoll health --database

# Test LLM provider connectivity  
ragdoll health --llm

# Verbose output for debugging
ragdoll health --verbose
```

### Permission Issues

```bash
# Ensure CLI is in PATH
which ragdoll

# Check gem installation
gem list ragdoll-cli
```

### Configuration Problems

```bash
# Show configuration file location
ragdoll config path

# Validate configuration
ragdoll config validate

# Reset to defaults
ragdoll config reset
```

## Next Steps

- [Commands](commands.md) - Learn about available commands
- [Configuration](configuration.md) - Advanced configuration options
- [Examples](examples.md) - Common usage patterns