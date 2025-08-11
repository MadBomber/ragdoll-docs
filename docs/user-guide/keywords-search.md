# Keywords Search

Ragdoll provides powerful keywords-based search functionality that can be used standalone or combined with semantic search for highly precise document retrieval. The keywords system leverages PostgreSQL array operations and GIN indexing for exceptional performance.

## Overview

The keywords search system is inspired by the `find_matching_entries.rb` algorithm but optimized for PostgreSQL arrays. It provides two main search modes:

- **Overlap Search** (`&&`): Find documents containing any of the specified keywords
- **Contains All Search** (`@>`): Find documents containing all specified keywords

## Basic Usage

### Simple Keywords Search

```ruby
# Find documents containing any of these keywords
results = Ragdoll::Document.search_by_keywords(['machine', 'learning', 'ai'])

# Results are automatically sorted by match count (most matches first)
results.each do |doc|
  puts "#{doc.title}: #{doc.keywords_match_count} keyword matches"
end
```

### Exact Keywords Search

```ruby
# Find documents that contain ALL specified keywords
results = Ragdoll::Document.search_by_keywords_all(['ruby', 'programming'])

# Results are sorted by focus (fewer total keywords = more focused)
results.each do |doc|
  puts "#{doc.title}: #{doc.total_keywords_count} total keywords"
end
```

## Advanced Features

### Case Insensitive Search

Keywords are automatically normalized to lowercase for consistent matching:

```ruby
# These all match the same documents:
results1 = Ragdoll::Document.search_by_keywords(['Python'])
results2 = Ragdoll::Document.search_by_keywords(['PYTHON']) 
results3 = Ragdoll::Document.search_by_keywords(['python'])
```

### Input Normalization

The system accepts various input formats:

```ruby
# Array of strings (recommended)
results = Ragdoll::Document.search_by_keywords(['web', 'development'])

# Single string
results = Ragdoll::Document.search_by_keywords('python')

# Mixed case array
results = Ragdoll::Document.search_by_keywords(['Machine', 'LEARNING', 'ai'])
```

### Search Options

```ruby
# Custom result limit
results = Ragdoll::Document.search_by_keywords(['ai'], limit: 50)

# Empty/invalid inputs return no results
results = Ragdoll::Document.search_by_keywords([])          # Empty array
results = Ragdoll::Document.search_by_keywords(nil)        # Nil input
results = Ragdoll::Document.search_by_keywords(['', ' '])  # Empty strings
```

## Integration with Semantic Search

Keywords search integrates seamlessly with Ragdoll's semantic search system:

```ruby
# Combined semantic + keywords search
search_engine = Ragdoll::SearchEngine.new(embedding_service)

results = search_engine.search_similar_content(
  'artificial intelligence applications',
  keywords: ['ai', 'machine learning', 'neural networks'],
  limit: 10
)

# The search will:
# 1. Generate embeddings for the query
# 2. Filter results to documents matching keywords
# 3. Rank by semantic similarity within keyword matches
```

## Performance Optimization

### Database Indexing

Ragdoll automatically creates a GIN index on the keywords column for optimal performance:

```sql
-- This index is created automatically via migration
CREATE INDEX index_ragdoll_documents_on_keywords_gin 
ON ragdoll_documents USING GIN (keywords);
```

### Search Performance

The GIN index provides excellent performance characteristics:

- **O(log n) lookup time** for keyword overlap queries
- **Efficient array operations** using PostgreSQL's native array support
- **Fast intersection calculations** for multiple keyword searches

### Performance Comparison

```ruby
# Without GIN index: O(n) full table scan
# With GIN index: O(log n) + O(m) where m = matching documents

# Example query plans:
# Without index: Seq Scan on ragdoll_documents (cost=0.00..1000.00)
# With index:    Bitmap Index Scan on idx_keywords_gin (cost=0.00..10.00)
```

## Query Examples

### Common Use Cases

```ruby
# Technology documents
tech_docs = Ragdoll::Document.search_by_keywords([
  'javascript', 'react', 'nodejs', 'typescript'
])

# Research papers
research = Ragdoll::Document.search_by_keywords([
  'research', 'study', 'analysis', 'methodology'
])

# Programming tutorials
tutorials = Ragdoll::Document.search_by_keywords([
  'tutorial', 'guide', 'howto', 'example'
])

# Find focused documents (exact match)
python_specific = Ragdoll::Document.search_by_keywords_all([
  'python', 'programming'
])
```

### Complex Queries

```ruby
# Multi-step search process
step1 = Ragdoll::Document.search_by_keywords(['ai', 'machine learning'])
step2 = step1.where(status: 'processed')
step3 = step2.where('created_at > ?', 1.month.ago)
final_results = step3.limit(20)

# Combining with other ActiveRecord methods
recent_ai_docs = Ragdoll::Document
  .search_by_keywords(['artificial intelligence'])
  .where(document_type: 'text')
  .where('file_modified_at > ?', 1.week.ago)
  .order(:created_at)
```

## Search Result Analysis

### Understanding Match Scores

```ruby
results = Ragdoll::Document.search_by_keywords(['web', 'api', 'rest'])

results.each do |doc|
  puts "Document: #{doc.title}"
  puts "Keywords: #{doc.keywords.join(', ')}"
  puts "Matches: #{doc.keywords_match_count}/3"
  puts "Total Keywords: #{doc.total_keywords_count}"
  puts "---"
end
```

### Result Ordering

**Overlap Search** (`search_by_keywords`):
1. **Primary**: Match count (descending) - documents with more keyword matches rank higher
2. **Secondary**: Creation date (descending) - newer documents break ties

**Exact Search** (`search_by_keywords_all`):
1. **Primary**: Total keyword count (ascending) - more focused documents rank higher
2. **Secondary**: Creation date (descending) - newer documents break ties

## Error Handling

The keywords search system gracefully handles edge cases:

```ruby
# Empty inputs
Ragdoll::Document.search_by_keywords([])     # Returns empty relation
Ragdoll::Document.search_by_keywords(nil)    # Returns empty relation
Ragdoll::Document.search_by_keywords('')     # Returns empty relation

# Invalid inputs
Ragdoll::Document.search_by_keywords(['', ' ', nil])  # Filters out empty values

# No matches found
results = Ragdoll::Document.search_by_keywords(['nonexistent'])
# Returns empty ActiveRecord::Relation, not an error
```

## Best Practices

### 1. Use Appropriate Search Method

```ruby
# Use overlap search for broad discovery
broad_results = Ragdoll::Document.search_by_keywords(['ai', 'data', 'analysis'])

# Use exact search for precision
focused_results = Ragdoll::Document.search_by_keywords_all(['python', 'tutorial'])
```

### 2. Optimize Keyword Design

```ruby
# Good: Specific, focused keywords
keywords = ['machine-learning', 'classification', 'scikit-learn']

# Avoid: Too generic or common words
keywords = ['the', 'and', 'with']  # Not useful for search
```

### 3. Combine with Semantic Search

```ruby
# Best of both worlds: semantic understanding + keyword precision
results = search_engine.search_similar_content(
  'How to build neural networks',
  keywords: ['tensorflow', 'keras', 'python'],
  limit: 15
)
```

### 4. Monitor Performance

```ruby
# Log search performance for optimization
start_time = Time.current
results = Ragdoll::Document.search_by_keywords(['optimization'])
execution_time = (Time.current - start_time) * 1000

puts "Keywords search took #{execution_time}ms for #{results.count} results"
```

## Algorithm Details

The keywords search implementation closely follows the `find_matching_entries.rb` algorithm but leverages PostgreSQL's native array capabilities:

### Original Ruby Algorithm
```ruby
# find_matching_entries.rb approach
match_count = (entry.map(&:downcase) & data).size
matches[match_count] << { entry: entry, index: index }
```

### PostgreSQL Optimization
```sql
-- Ragdoll's PostgreSQL approach
SELECT *, cardinality(keywords & ARRAY['keyword1', 'keyword2']::text[]) as keywords_match_count
FROM ragdoll_documents
WHERE keywords && ARRAY['keyword1', 'keyword2']::text[]
ORDER BY keywords_match_count DESC
```

### Benefits of PostgreSQL Implementation

1. **Server-side Processing**: Computation happens in the database
2. **Index Utilization**: GIN indexes accelerate array operations
3. **Memory Efficiency**: No need to load all data into Ruby memory
4. **Scalability**: Performance remains consistent with large datasets
5. **Concurrent Access**: Multiple searches can run simultaneously

## Migration Guide

If you're upgrading from a custom keyword search implementation, here's how to migrate:

### From Manual Array Queries

```ruby
# Before: Manual array overlap
where("'machine' = ANY(keywords) OR 'learning' = ANY(keywords)")

# After: Use search_by_keywords
search_by_keywords(['machine', 'learning'])
```

### From ILIKE Pattern Matching

```ruby
# Before: Slow pattern matching
where("keywords ILIKE ? OR keywords ILIKE ?", '%machine%', '%learning%')

# After: Fast array operations
search_by_keywords(['machine', 'learning'])
```

### Adding the GIN Index

If upgrading an existing installation, run the migration:

```bash
# Run the keywords index migration
rails db:migrate
```

## Troubleshooting

### Common Issues

**Issue**: Slow keyword searches
**Solution**: Ensure the GIN index exists with `\d+ ragdoll_documents` in psql

**Issue**: Case sensitivity problems  
**Solution**: Keywords are automatically normalized - check your stored keywords format

**Issue**: No results found
**Solution**: Verify keywords are stored as arrays, not strings in the database

### Query Analysis

```sql
-- Analyze query performance
EXPLAIN ANALYZE 
SELECT *, cardinality(keywords & ARRAY['python']::text[]) as match_count
FROM ragdoll_documents 
WHERE keywords && ARRAY['python']::text[]
ORDER BY match_count DESC 
LIMIT 20;
```

### Debug Information

```ruby
# Enable debug logging
ENV["RAGDOLL_DEBUG"] = "true"

# Check stored keywords format
doc = Ragdoll::Document.first
puts doc.keywords.class           # Should be Array
puts doc.keywords.inspect         # Check the actual values
```

## Future Enhancements

The keywords search system is designed for extensibility:

- **Weighted Keywords**: Support for keyword importance scoring
- **Synonym Expansion**: Automatic expansion of related terms
- **Fuzzy Matching**: Approximate keyword matching
- **Analytics Integration**: Keyword search tracking and optimization
- **Custom Operators**: Support for additional PostgreSQL array operators

For the latest features and updates, check the [changelog](../../changelog.md) and [API reference](../api-reference/api-models.md).