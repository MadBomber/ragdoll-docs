# CLI Examples

Common usage patterns and workflows for the Ragdoll CLI.

## Getting Started

### Initial Setup

```bash
# Install the CLI
gem install ragdoll-cli

# Configure database connection
ragdoll config set database.url "postgresql://user:pass@localhost/ragdoll"

# Configure LLM provider
ragdoll config set llm.provider "openai"
ragdoll config set llm.api_key "sk-your-key-here"

# Test the setup
ragdoll health
```

### First Document Upload

```bash
# Upload a single document
ragdoll update ~/Documents/research-paper.pdf

# Upload with metadata
ragdoll update ~/Documents/report.pdf \
  --metadata "author=Jane Doe" \
  --metadata "department=Research" \
  --metadata "category=quarterly-report"

# Upload and extract images
ragdoll update ~/Documents/presentation.pptx --extract-images
```

## Document Management

### Bulk Document Processing

```bash
# Process entire directory
ragdoll update ~/Documents/papers --recursive

# Process specific file types
find ~/Documents -name "*.pdf" -exec ragdoll update {} \;

# Process with custom metadata based on path
for file in ~/Documents/research/*.pdf; do
  ragdoll update "$file" --metadata "category=research"
done
```

### Updating Existing Documents

```bash
# Force update (even if file hasn't changed)
ragdoll update document.pdf --force

# Update with new metadata
ragdoll update document.pdf \
  --metadata "status=reviewed" \
  --metadata "last-updated=$(date)"
```

### Removing Documents

```bash
# Delete by file path
ragdoll delete ~/Documents/old-document.pdf --by-path

# Delete by document ID
ragdoll list documents --format json | jq '.[] | select(.title | contains("obsolete")) | .id' | \
  xargs -I {} ragdoll delete {} --by-id --confirm
```

## Search Operations

### Basic Searches

```bash
# Simple search
ragdoll search "machine learning algorithms"

# Search with result limit
ragdoll search "Python programming" --limit 5

# Search with similarity threshold
ragdoll search "neural networks" --threshold 0.8
```

### Advanced Search Queries

```bash
# Search with JSON output for processing
ragdoll search "data science" --format json > search-results.json

# Search and extract specific fields
ragdoll search "AI research" --format json | \
  jq '.[] | {title: .title, score: .similarity_score, path: .file_path}'

# Search with metadata inclusion
ragdoll search "quarterly report" --include-metadata --format json | \
  jq '.[] | select(.metadata.department == "Research")'
```

### Search Workflows

```bash
# Create a search report
cat << 'EOF' > search-report.sh
#!/bin/bash
QUERY="$1"
echo "# Search Results for: $QUERY"
echo "Generated: $(date)"
echo ""
ragdoll search "$QUERY" --include-metadata --format json | \
  jq -r '.[] | "## \(.title)\n**Score:** \(.similarity_score)\n**Path:** \(.file_path)\n**Summary:** \(.summary // "No summary available")\n"'
EOF
chmod +x search-report.sh

# Usage
./search-report.sh "artificial intelligence" > ai-search-report.md
```

## System Monitoring

### Health Monitoring

```bash
# Basic health check
ragdoll health

# Detailed health check with timing
time ragdoll health --verbose

# Monitor system status continuously
ragdoll status --watch --interval 30
```

### Performance Monitoring

```bash
# Monitor search performance
echo "machine learning,$(time (ragdoll search "machine learning" --limit 1 >/dev/null) 2>&1 | grep real)"

# Batch performance testing
for query in "AI" "ML" "data science" "algorithms"; do
  echo -n "$query: "
  time ragdoll search "$query" --limit 5 >/dev/null
done 2>&1 | grep real
```

### System Statistics

```bash
# Basic statistics
ragdoll stats

# Detailed statistics with formatting
ragdoll stats --detailed --format json | jq '{
  documents: .total_documents,
  embeddings: .total_embeddings,
  storage: .storage_size,
  last_update: .last_updated
}'
```

## Batch Operations

### Processing Multiple Files

```bash
# Process files in parallel
find ~/Documents -name "*.pdf" -print0 | \
  xargs -0 -P 4 -I {} ragdoll update {}

# Process with progress tracking
total=$(find ~/Documents -name "*.pdf" | wc -l)
count=0
find ~/Documents -name "*.pdf" | while read file; do
  count=$((count + 1))
  echo "Processing $count/$total: $file"
  ragdoll update "$file"
done
```

### Bulk Metadata Updates

```bash
# Add category metadata to all PDFs in research directory
find ~/Documents/research -name "*.pdf" | while read file; do
  ragdoll update "$file" --metadata "category=research"
done

# Update metadata based on file location
for dir in ~/Documents/*/; do
  category=$(basename "$dir")
  find "$dir" -name "*.pdf" | while read file; do
    ragdoll update "$file" --metadata "category=$category"
  done
done
```

## Integration Examples

### Backup and Sync

```bash
# Backup document list
ragdoll list documents --format json > documents-backup-$(date +%Y%m%d).json

# Sync documents from backup
jq -r '.[].file_path' documents-backup.json | while read path; do
  if [ -f "$path" ]; then
    ragdoll update "$path"
  fi
done
```

### Web Scraping Integration

```bash
# Download and process web content
curl -s "https://example.com/article" | \
  html2text > temp-article.txt && \
  ragdoll update temp-article.txt \
    --metadata "source=web" \
    --metadata "url=https://example.com/article" && \
  rm temp-article.txt
```

### Git Integration

```bash
# Process files changed in git
git diff --name-only HEAD~1 HEAD | \
  grep -E '\.(pdf|docx|txt|md)$' | \
  xargs -I {} ragdoll update {}

# Add git metadata
for file in $(git ls-files '*.md'); do
  commit=$(git log -1 --format="%H" -- "$file")
  author=$(git log -1 --format="%an" -- "$file")
  ragdoll update "$file" \
    --metadata "git-commit=$commit" \
    --metadata "git-author=$author"
done
```

## Automation Scripts

### Daily Processing Script

```bash
#!/bin/bash
# daily-ragdoll-sync.sh

LOG_FILE="$HOME/.ragdoll/logs/daily-sync.log"
echo "$(date): Starting daily Ragdoll sync" >> "$LOG_FILE"

# Check system health
if ! ragdoll health >/dev/null 2>&1; then
  echo "$(date): Health check failed!" >> "$LOG_FILE"
  exit 1
fi

# Process new documents
find ~/Documents -name "*.pdf" -mtime -1 | while read file; do
  echo "$(date): Processing $file" >> "$LOG_FILE"
  ragdoll update "$file" 2>&1 | tee -a "$LOG_FILE"
done

# Clean up old logs
find ~/.ragdoll/logs -name "*.log" -mtime +7 -delete

echo "$(date): Daily sync completed" >> "$LOG_FILE"
```

### Search API Wrapper

```bash
#!/bin/bash
# ragdoll-api.sh - Simple HTTP API wrapper

PORT=${PORT:-8080}

handle_search() {
  local query=$(echo "$1" | sed 's/+/ /g')
  ragdoll search "$query" --format json
}

while true; do
  echo -e "HTTP/1.1 200 OK\nContent-Type: application/json\n" | nc -l "$PORT" -c "
    read method path protocol
    if [[ \$path =~ ^/search\\?q=(.+) ]]; then
      handle_search \${BASH_REMATCH[1]}
    else
      echo '{\"error\": \"Not found\"}'
    fi
  "
done
```

## Troubleshooting Workflows

### Diagnostic Script

```bash
#!/bin/bash
# ragdoll-diagnostics.sh

echo "=== Ragdoll CLI Diagnostics ==="
echo "Timestamp: $(date)"
echo "CLI Version: $(ragdoll --version)"
echo ""

echo "=== Configuration ==="
ragdoll config list
echo ""

echo "=== Health Check ==="
ragdoll health --verbose
echo ""

echo "=== System Stats ==="
ragdoll stats --detailed
echo ""

echo "=== Recent Log Entries ==="
tail -n 20 ~/.ragdoll/logs/ragdoll.log 2>/dev/null || echo "No log file found"
```

### Performance Benchmark

```bash
#!/bin/bash
# benchmark-ragdoll.sh

QUERIES=("artificial intelligence" "machine learning" "data science" "neural networks" "algorithms")

echo "=== Ragdoll Performance Benchmark ==="
echo "Timestamp: $(date)"
echo ""

for query in "${QUERIES[@]}"; do
  echo "Testing query: '$query'"
  time ragdoll search "$query" --limit 10 >/dev/null
  echo ""
done
```

These examples provide a comprehensive foundation for using the Ragdoll CLI effectively in various scenarios and workflows.