# Rails Installation

This guide walks you through installing and setting up ragdoll-rails in your Ruby on Rails application.

## Prerequisites

- Ruby 3.2.0 or higher
- Rails 7.0 or higher
- PostgreSQL with pgvector extension
- Redis (for background jobs)

## Installation Steps

### 1. Add to Gemfile

```ruby
# Gemfile
gem 'ragdoll-rails'

# Optional: Background job processing
gem 'sidekiq' # or 'resque'
```

### 2. Bundle Install

```bash
bundle install
```

### 3. Run the Generator

The initializer generator sets up the basic configuration:

```bash
rails generate ragdoll:init
```

This creates:
- Configuration initializer file
- Setup instructions

### 4. Install Migrations

Copy the database migrations from the ragdoll engine:

```bash
rails ragdoll:install:migrations
```

This copies:
- Database migrations for documents, embeddings, and content tables
- PostgreSQL extension setup

### 5. Configure Database

Ensure PostgreSQL with pgvector is set up:

```bash
# Install pgvector extension
psql your_database -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### 6. Run Migrations

```bash
rails db:migrate
```

### 7. Configure LLM Provider

Edit the generated configuration:

```ruby
# config/initializers/ragdoll_config.rb
Ragdoll.configure do |config|
  config.llm_provider = :openai
  config.openai_api_key = Rails.application.credentials.openai_api_key
  config.default_embedding_model = "text-embedding-3-small"
  config.default_chat_model = "gpt-4"
end
```

### 8. Mount the Engine

Add to your routes file:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount Ragdoll::Engine => '/ragdoll'
  # Your other routes...
end
```

## Configuration Options

### Basic Configuration

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # LLM Provider Settings
  config.llm_provider = :openai
  config.openai_api_key = Rails.application.credentials.openai_api_key
  config.openai_base_url = "https://api.openai.com/v1" # Optional
  
  # Model Settings
  config.default_embedding_model = "text-embedding-3-small"
  config.default_chat_model = "gpt-4"
  config.embedding_dimensions = 1536
  
  # Processing Settings
  config.chunk_size = 1000
  config.chunk_overlap = 200
  config.max_file_size = 50.megabytes
  
  # Search Settings
  config.default_search_limit = 10
  config.similarity_threshold = 0.7
  
  # Background Job Settings
  config.processing_queue = :ragdoll
  config.job_adapter = :sidekiq # or :resque, :delayed_job
end
```

### Environment-Specific Configuration

```ruby
# config/environments/development.rb
config.after_initialize do
  Ragdoll.configure do |ragdoll|
    ragdoll.log_level = :debug
    ragdoll.async_processing = false # Process immediately in development
  end
end

# config/environments/production.rb
config.after_initialize do
  Ragdoll.configure do |ragdoll|
    ragdoll.log_level = :info
    ragdoll.async_processing = true
    ragdoll.processing_queue = :ragdoll_production
  end
end
```

## Background Job Setup

### Sidekiq Setup

```ruby
# Gemfile
gem 'sidekiq'

# config/application.rb
config.active_job.queue_adapter = :sidekiq

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

### Queue Configuration

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  config.processing_queue = :ragdoll
  config.embedding_queue = :ragdoll_embeddings
  config.indexing_queue = :ragdoll_indexing
end
```

## Asset Pipeline Setup

### Sprockets (Rails 6/7)

```javascript
// app/assets/javascripts/application.js
//= require ragdoll/application

// app/assets/stylesheets/application.css
*= require ragdoll/application
```

### Webpacker/Shakapacker

```javascript
// app/javascript/packs/application.js
import 'ragdoll/application'

// app/assets/stylesheets/application.scss
@import 'ragdoll/application';
```

## Database Configuration

### PostgreSQL with pgvector

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development
  # Ensure pgvector extension is available
  
production:
  <<: *default
  database: myapp_production
  url: <%= ENV['DATABASE_URL'] %>
```

### Migration Customization

```ruby
# Override default migration if needed
class CustomRagdollSetup < ActiveRecord::Migration[7.0]
  def change
    # Enable pgvector extension
    enable_extension 'vector'
    
    # Run standard Ragdoll migrations
    Ragdoll::InstallGenerator.new.create_migrations
    
    # Add custom indexes or modifications
    add_index :ragdoll_documents, [:user_id, :created_at]
  end
end
```

## Security Configuration

### Strong Parameters

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # Permitted file types
  config.allowed_file_types = %w[.pdf .doc .docx .txt .md .html]
  
  # Maximum file size
  config.max_file_size = 50.megabytes
  
  # Content security
  config.sanitize_content = true
  config.extract_metadata = true
end
```

### Authentication Integration

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # Integrate with your authentication system
  config.current_user_method = :current_user
  config.authenticate_user_method = :authenticate_user!
  
  # Authorization
  config.authorize_document_access = ->(document, user) {
    document.user == user || user.admin?
  }
end
```

## File Storage Configuration

### Active Storage Setup

```ruby
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

production:
  service: S3
  access_key_id: <%= Rails.application.credentials.aws[:access_key_id] %>
  secret_access_key: <%= Rails.application.credentials.aws[:secret_access_key] %>
  region: us-east-1
  bucket: your-bucket-name

# config/environments/production.rb
config.active_storage.variant_processor = :mini_magick
```

### Storage Configuration

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # File storage options
  config.storage_service = :local # or :s3, :gcs, etc.
  config.preserve_original_files = true
  config.generate_thumbnails = true
  config.thumbnail_sizes = { small: '150x150', medium: '300x300', large: '600x600' }
end
```

## Verification

### Test the Installation

```ruby
# In Rails console
rails console

# Test configuration
Ragdoll.configuration.llm_provider
# => :openai

# Test model creation
doc = Ragdoll::Document.create!(
  title: "Test Document",
  content: "This is a test document for verification."
)

# Test search
Ragdoll::Document.search("test")
```

### Health Check

Create a health check endpoint:

```ruby
# config/routes.rb
get '/health/ragdoll', to: 'health#ragdoll'

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def ragdoll
    checks = {
      database: check_database,
      llm_provider: check_llm_provider,
      background_jobs: check_background_jobs
    }
    
    render json: { status: checks.values.all? ? 'healthy' : 'unhealthy', checks: checks }
  end
  
  private
  
  def check_database
    Ragdoll::Document.connection.active?
  rescue
    false
  end
  
  def check_llm_provider
    Ragdoll::LLMService.new.health_check
  rescue
    false
  end
  
  def check_background_jobs
    Sidekiq.redis(&:ping) == 'PONG'
  rescue
    false
  end
end
```

## Troubleshooting

### Common Issues

#### pgvector Extension Missing

```bash
# Install on Ubuntu/Debian
sudo apt install postgresql-14-pgvector

# Install on macOS with Homebrew
brew install pgvector

# Enable in database
psql your_database -c "CREATE EXTENSION vector;"
```

#### Asset Compilation Issues

```bash
# Precompile assets
rails assets:precompile

# Clear asset cache
rails assets:clobber
```

#### Background Job Issues

```bash
# Check Sidekiq status
bundle exec sidekiq

# Monitor queues
bundle exec sidekiq-web
```

### Configuration Validation

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # ... your configuration
  
  # Validate configuration on startup
  config.validate_on_startup = true
end
```

### Logging

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  config.logger = Rails.logger
  config.log_level = Rails.env.production? ? :info : :debug
end
```

## Next Steps

- [Configuration](configuration.md) - Detailed configuration options
- [Models](models.md) - Learn about the ActiveRecord models
- [Controllers](controllers.md) - Understand the provided controllers
- [Examples](examples.md) - See implementation examples