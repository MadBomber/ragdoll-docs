# CLI Configuration

Detailed configuration options for the Ragdoll CLI.

## Configuration Sources

The CLI loads configuration from multiple sources in this order:

1. Command-line flags
2. Environment variables
3. Configuration file (`~/.ragdoll/config.yml`)
4. System defaults

## Configuration File

### Location

Default location: `~/.ragdoll/config.yml`

Custom location:
```bash
ragdoll --config /path/to/config.yml <command>
```

### Format

```yaml
# Database configuration
database:
  url: "postgresql://user:password@localhost:5432/ragdoll"
  pool_size: 10
  timeout: 30

# LLM Provider configuration
llm:
  provider: "openai"  # openai, anthropic, ollama, etc.
  api_key: "your-api-key"
  base_url: "https://api.openai.com/v1"  # Optional for custom endpoints
  model: "gpt-4"
  temperature: 0.7
  max_tokens: 2048

# Embedding configuration  
embeddings:
  provider: "openai"
  model: "text-embedding-3-small"
  dimensions: 1536

# Search configuration
search:
  default_limit: 10
  similarity_threshold: 0.7
  hybrid_search: true

# Logging configuration
logging:
  level: "info"  # debug, info, warn, error
  file: "~/.ragdoll/logs/ragdoll.log"
  max_size: "10MB"
  max_files: 5

# Processing configuration
processing:
  chunk_size: 1000
  chunk_overlap: 200
  extract_images: true
  ocr_enabled: true

# CLI-specific settings
cli:
  output_format: "table"  # table, json, csv
  color_output: true
  progress_bars: true
  confirm_destructive: true
```

## Environment Variables

All configuration options can be set via environment variables using the pattern `RAGDOLL_<SECTION>_<KEY>`:

```bash
# Database
export RAGDOLL_DATABASE_URL="postgresql://localhost/ragdoll"
export RAGDOLL_DATABASE_POOL_SIZE="10"

# LLM
export RAGDOLL_LLM_PROVIDER="openai"
export RAGDOLL_LLM_API_KEY="your-key"
export RAGDOLL_LLM_MODEL="gpt-4"

# Embeddings
export RAGDOLL_EMBEDDINGS_PROVIDER="openai"
export RAGDOLL_EMBEDDINGS_MODEL="text-embedding-3-small"

# Logging
export RAGDOLL_LOGGING_LEVEL="debug"
export RAGDOLL_LOGGING_FILE="/var/log/ragdoll.log"
```

## LLM Provider Configuration

### OpenAI

```yaml
llm:
  provider: "openai"
  api_key: "sk-..."
  model: "gpt-4"
  temperature: 0.7
```

### Anthropic

```yaml
llm:
  provider: "anthropic"
  api_key: "sk-ant-..."
  model: "claude-3-sonnet-20240229"
  max_tokens: 4096
```

### Ollama (Local)

```yaml
llm:
  provider: "ollama"
  base_url: "http://localhost:11434"
  model: "llama2"
```

### Azure OpenAI

```yaml
llm:
  provider: "azure"
  api_key: "your-key"
  base_url: "https://your-resource.openai.azure.com"
  model: "gpt-4"
  api_version: "2024-02-15-preview"
```

## Database Configuration

### PostgreSQL

```yaml
database:
  url: "postgresql://user:password@host:port/database"
  pool_size: 10
  timeout: 30
  ssl_mode: "require"  # disable, allow, prefer, require
```

### Connection String Format

```
postgresql://[user[:password]@][host][:port][/dbname][?param1=value1&...]
```

Common parameters:
- `sslmode=require` - Require SSL connection
- `application_name=ragdoll-cli` - Set application name
- `connect_timeout=10` - Connection timeout in seconds

## Configuration Management

### View Configuration

```bash
# Show all configuration
ragdoll config list

# Show specific section
ragdoll config list --section llm

# Show single value
ragdoll config get llm.provider
```

### Update Configuration

```bash
# Set single value
ragdoll config set llm.model "gpt-4-turbo"

# Set nested value
ragdoll config set database.pool_size 20
```

### Validate Configuration

```bash
# Validate all settings
ragdoll config validate

# Test connections
ragdoll health
```

### Reset Configuration

```bash
# Reset to defaults
ragdoll config reset

# Reset specific section
ragdoll config reset --section llm
```

## Security Best Practices

### API Keys

1. **Use environment variables** for sensitive values:
   ```bash
   export RAGDOLL_LLM_API_KEY="your-key"
   ```

2. **Restrict file permissions**:
   ```bash
   chmod 600 ~/.ragdoll/config.yml
   ```

3. **Use key management services** in production:
   ```bash
   export RAGDOLL_LLM_API_KEY="$(aws secretsmanager get-secret-value --secret-id ragdoll/api-key --query SecretString --output text)"
   ```

### Database Connections

1. **Use SSL connections**:
   ```yaml
   database:
     url: "postgresql://user:pass@host/db?sslmode=require"
   ```

2. **Limit connection privileges**:
   - Create dedicated database user
   - Grant minimal required permissions

3. **Use connection pooling**:
   ```yaml
   database:
     pool_size: 10
     timeout: 30
   ```

## Troubleshooting

### Configuration Issues

```bash
# Show configuration file location
ragdoll config path

# Validate configuration
ragdoll config validate

# Debug configuration loading
ragdoll --verbose config list
```

### Connection Problems

```bash
# Test database connection
ragdoll health --database --verbose

# Test LLM connection
ragdoll health --llm --verbose
```

### Permission Errors

```bash
# Check file permissions
ls -la ~/.ragdoll/config.yml

# Fix permissions
chmod 600 ~/.ragdoll/config.yml
```