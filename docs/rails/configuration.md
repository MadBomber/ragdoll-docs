# Rails Configuration

Comprehensive configuration guide for ragdoll-rails, covering all aspects from basic setup to advanced customization.

## Configuration Structure

Ragdoll Rails uses a centralized configuration system accessible through `Ragdoll.configuration`:

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # Your configuration here
end
```

## Core Configuration

### LLM Provider Settings

```ruby
Ragdoll.configure do |config|
  # Provider selection
  config.llm_provider = :openai  # :openai, :anthropic, :ollama, :azure
  
  # API credentials
  config.openai_api_key = Rails.application.credentials.openai_api_key
  config.anthropic_api_key = Rails.application.credentials.anthropic_api_key
  
  # Base URLs (optional for custom endpoints)
  config.openai_base_url = "https://api.openai.com/v1"
  config.anthropic_base_url = "https://api.anthropic.com"
  
  # Model specifications
  config.default_chat_model = "gpt-4"
  config.default_embedding_model = "text-embedding-3-small"
  config.embedding_dimensions = 1536
  
  # Request settings
  config.llm_timeout = 30.seconds
  config.max_retries = 3
  config.retry_delay = 1.second
end
```

### Database Configuration

```ruby
Ragdoll.configure do |config|
  # Vector database settings
  config.vector_dimensions = 1536
  config.similarity_function = :cosine  # :cosine, :euclidean, :dot_product
  
  # Connection settings
  config.database_pool_size = 10
  config.database_timeout = 30.seconds
  
  # Indexing options
  config.create_vector_indexes = true
  config.index_type = :ivfflat  # :ivfflat, :hnsw
  config.index_lists = 100      # For IVFFlat indexes
end
```

## Document Processing

### File Handling

```ruby
Ragdoll.configure do |config|
  # Supported file types
  config.allowed_file_types = %w[
    .pdf .doc .docx .txt .md .html .rtf
    .jpg .jpeg .png .gif .webp
    .mp3 .wav .m4a .flac
  ]
  
  # File size limits
  config.max_file_size = 50.megabytes
  config.max_image_size = 10.megabytes
  config.max_audio_size = 100.megabytes
  
  # Storage settings
  config.storage_service = :local  # or :s3, :gcs, :azure
  config.preserve_original_files = true
  config.generate_thumbnails = true
  config.thumbnail_sizes = {
    small: '150x150',
    medium: '300x300',
    large: '600x600'
  }
end
```

### Content Processing

```ruby
Ragdoll.configure do |config|
  # Text processing
  config.chunk_size = 1000
  config.chunk_overlap = 200
  config.min_chunk_size = 100
  config.max_chunk_size = 2000
  
  # Content extraction
  config.extract_metadata = true
  config.extract_images = true
  config.extract_audio = false
  config.perform_ocr = true
  
  # Content cleaning
  config.sanitize_content = true
  config.remove_empty_chunks = true
  config.normalize_whitespace = true
  
  # Language detection
  config.detect_language = true
  config.default_language = 'en'
end
```

## Search Configuration

### Search Behavior

```ruby
Ragdoll.configure do |config|
  # Default search settings
  config.default_search_limit = 10
  config.max_search_limit = 100
  config.similarity_threshold = 0.7
  config.min_similarity_threshold = 0.1
  
  # Search types
  config.enable_semantic_search = true
  config.enable_keyword_search = true
  config.enable_hybrid_search = true
  config.hybrid_search_weight = 0.7  # Weight for semantic vs keyword
  
  # Result ranking
  config.boost_recent_documents = true
  config.recency_boost_factor = 0.1
  config.boost_popular_documents = true
  config.popularity_boost_factor = 0.05
end
```

### Search Features

```ruby
Ragdoll.configure do |config|
  # Advanced features
  config.enable_faceted_search = true
  config.enable_autocomplete = true
  config.enable_spell_check = true
  config.enable_query_expansion = true
  
  # Caching
  config.cache_search_results = true
  config.search_cache_duration = 1.hour
  config.cache_embeddings = true
  config.embedding_cache_duration = 24.hours
end
```

## Background Jobs

### Job Processing

```ruby
Ragdoll.configure do |config|
  # Job adapter
  config.job_adapter = :sidekiq  # :sidekiq, :resque, :delayed_job, :inline
  
  # Queue configuration
  config.processing_queue = :ragdoll
  config.embedding_queue = :ragdoll_embeddings
  config.indexing_queue = :ragdoll_indexing
  config.cleanup_queue = :ragdoll_cleanup
  
  # Processing options
  config.async_processing = true
  config.batch_processing = true
  config.batch_size = 10
  
  # Retry configuration
  config.job_max_retries = 3
  config.job_retry_delay = 30.seconds
  config.job_timeout = 5.minutes
end
```

### Performance Tuning

```ruby
Ragdoll.configure do |config|
  # Concurrency
  config.max_concurrent_jobs = 5
  config.embedding_batch_size = 50
  config.indexing_batch_size = 100
  
  # Memory management
  config.memory_limit_per_job = 512.megabytes
  config.cleanup_temp_files = true
  config.gc_after_processing = true
end
```

## Security & Access Control

### Authentication Integration

```ruby
Ragdoll.configure do |config|
  # User methods
  config.current_user_method = :current_user
  config.authenticate_user_method = :authenticate_user!
  
  # User model
  config.user_class = 'User'
  config.user_foreign_key = :user_id
  
  # Session handling
  config.require_authentication = true
  config.allow_anonymous_search = false
end
```

### Authorization

```ruby
Ragdoll.configure do |config|
  # Document access control
  config.authorize_document_access = ->(document, user) {
    return true if user&.admin?
    return false unless user
    
    case document.visibility
    when 'public' then true
    when 'private' then document.user == user
    when 'team' then document.user.team == user.team
    else false
    end
  }
  
  # Search authorization
  config.authorize_search = ->(user) {
    user&.active? && (user.premium? || user.search_credits > 0)
  }
  
  # Upload permissions
  config.authorize_upload = ->(user) {
    user&.active? && user.can_upload_documents?
  }
end
```

### Content Security

```ruby
Ragdoll.configure do |config|
  # Content filtering
  config.content_filters = [
    :remove_pii,        # Remove personally identifiable information
    :sanitize_html,     # Clean HTML content
    :check_virus,       # Virus scanning
    :validate_content   # Content validation
  ]
  
  # File validation
  config.validate_file_signatures = true
  config.scan_for_malware = true
  config.quarantine_suspicious_files = true
  
  # Data privacy
  config.anonymize_user_data = false
  config.encrypt_sensitive_content = true
  config.audit_access = true
end
```

## UI & Views Configuration

### View Settings

```ruby
Ragdoll.configure do |config|
  # Theme and styling
  config.ui_theme = :default  # :default, :dark, :minimal
  config.custom_css_path = 'ragdoll/custom'
  config.custom_js_path = 'ragdoll/custom'
  
  # Layout options
  config.layout = 'application'
  config.use_turbo = true
  config.use_stimulus = true
  
  # Search interface
  config.search_placeholder = "Search documents..."
  config.show_search_filters = true
  config.show_result_thumbnails = true
  config.results_per_page = 20
  
  # Document viewer
  config.enable_document_preview = true
  config.preview_max_size = 5.megabytes
  config.syntax_highlighting = true
end
```

### Internationalization

```ruby
Ragdoll.configure do |config|
  # Language settings
  config.default_locale = :en
  config.available_locales = [:en, :es, :fr, :de, :ja]
  config.fallback_locale = :en
  
  # UI text customization
  config.custom_translations = {
    en: {
      ragdoll: {
        search: {
          placeholder: "What are you looking for?",
          no_results: "No documents found matching your search."
        }
      }
    }
  }
end
```

## Monitoring & Logging

### Logging Configuration

```ruby
Ragdoll.configure do |config|
  # Logger settings
  config.logger = Rails.logger
  config.log_level = Rails.env.production? ? :info : :debug
  config.log_format = :json  # :json, :text
  
  # Log targets
  config.log_search_queries = true
  config.log_processing_times = true
  config.log_llm_requests = false  # Disable in production for security
  config.log_user_actions = true
  
  # Performance logging
  config.slow_query_threshold = 1.second
  config.log_slow_queries = true
  config.profile_processing = Rails.env.development?
end
```

### Metrics & Analytics

```ruby
Ragdoll.configure do |config|
  # Metrics collection
  config.collect_metrics = true
  config.metrics_backend = :prometheus  # :prometheus, :statsd, :custom
  
  # Analytics
  config.track_search_analytics = true
  config.track_user_behavior = true
  config.analytics_retention = 90.days
  
  # Health checks
  config.enable_health_checks = true
  config.health_check_path = '/health/ragdoll'
  config.include_detailed_health = Rails.env.development?
end
```

## Environment-Specific Configuration

### Development Environment

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.after_initialize do
    Ragdoll.configure do |ragdoll|
      ragdoll.log_level = :debug
      ragdoll.async_processing = false
      ragdoll.profile_processing = true
      ragdoll.enable_detailed_health = true
      ragdoll.cache_search_results = false
      ragdoll.llm_timeout = 60.seconds  # Longer timeout for debugging
    end
  end
end
```

### Production Environment

```ruby
# config/environments/production.rb
Rails.application.configure do
  config.after_initialize do
    Ragdoll.configure do |ragdoll|
      ragdoll.log_level = :warn
      ragdoll.async_processing = true
      ragdoll.cache_search_results = true
      ragdoll.collect_metrics = true
      ragdoll.audit_access = true
      ragdoll.encrypt_sensitive_content = true
      ragdoll.scan_for_malware = true
    end
  end
end
```

### Test Environment

```ruby
# config/environments/test.rb
Rails.application.configure do
  config.after_initialize do
    Ragdoll.configure do |ragdoll|
      ragdoll.job_adapter = :inline
      ragdoll.async_processing = false
      ragdoll.llm_provider = :mock
      ragdoll.cache_search_results = false
      ragdoll.log_level = :error
    end
  end
end
```

## Custom Providers

### Custom LLM Provider

```ruby
Ragdoll.configure do |config|
  config.custom_llm_providers[:my_provider] = {
    class: 'MyCustomLLMProvider',
    config: {
      api_key: ENV['MY_LLM_API_KEY'],
      base_url: 'https://api.myprovider.com'
    }
  }
end

# app/services/my_custom_llm_provider.rb
class MyCustomLLMProvider < Ragdoll::LLMProviders::Base
  def generate_embedding(text)
    # Custom implementation
  end
  
  def generate_text(prompt)
    # Custom implementation
  end
end
```

### Custom Storage Provider

```ruby
Ragdoll.configure do |config|
  config.custom_storage_providers[:my_storage] = {
    class: 'MyCustomStorageProvider',
    config: {
      endpoint: ENV['MY_STORAGE_ENDPOINT'],
      credentials: ENV['MY_STORAGE_CREDENTIALS']
    }
  }
end
```

## Configuration Validation

### Runtime Validation

```ruby
Ragdoll.configure do |config|
  # Enable validation
  config.validate_on_startup = true
  config.strict_validation = Rails.env.production?
  
  # Custom validators
  config.custom_validators = [
    ->(config) { 
      raise "Invalid chunk size" if config.chunk_size > config.max_chunk_size
    }
  ]
end
```

### Configuration Testing

```ruby
# spec/support/ragdoll_config_spec.rb
RSpec.describe "Ragdoll Configuration" do
  it "has valid configuration" do
    expect { Ragdoll.configuration.validate! }.not_to raise_error
  end
  
  it "can connect to LLM provider" do
    expect(Ragdoll::LLMService.new.health_check).to be_truthy
  end
end
```