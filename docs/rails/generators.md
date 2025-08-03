# Rails Generators

The ragdoll-rails gem provides a comprehensive set of Rails generators to help you quickly set up and customize your document management system. These generators follow Rails conventions and can be easily customized.

## Installation Generator

### ragdoll:install

The main installation generator sets up the basic Ragdoll configuration in your Rails application.

```bash
rails generate ragdoll:install
```

#### What it creates:

- Configuration initializer (`config/initializers/ragdoll.rb`)
- Database migrations for core models
- Routes configuration
- Basic stylesheet and JavaScript imports
- Sample configuration files

#### Generated Files:

```ruby
# config/initializers/ragdoll.rb
Ragdoll.configure do |config|
  # LLM Provider configuration
  config.llm_provider = :openai
  config.openai_api_key = Rails.application.credentials.openai_api_key
  
  # Default models
  config.default_embedding_model = "text-embedding-3-small"
  config.default_chat_model = "gpt-4"
  
  # Processing settings
  config.chunk_size = 1000
  config.chunk_overlap = 200
  config.auto_process_documents = true
  
  # Authentication (customize for your app)
  config.current_user_method = :current_user
  config.authenticate_user_method = :authenticate_user!
end
```

```ruby
# config/routes.rb (added to existing routes)
Rails.application.routes.draw do
  mount Ragdoll::Engine => '/ragdoll'
  # ... your existing routes
end
```

#### Options:

```bash
# Skip database migrations
rails generate ragdoll:install --skip-migrations

# Skip routes configuration
rails generate ragdoll:install --skip-routes

# Specify a different mount path
rails generate ragdoll:install --mount-path=/documents

# Skip sample configuration
rails generate ragdoll:install --skip-config
```

## Model Generators

### ragdoll:model

Generates a model that integrates with Ragdoll search functionality.

```bash
rails generate ragdoll:model Article title:string content:text author:string
```

#### Generated Model:

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  include Ragdoll::Searchable
  
  ragdoll_searchable do |config|
    config.content_field = :content
    config.title_field = :title
    config.metadata_fields = [:author]
    config.chunk_size = 1000
    config.auto_process = true
  end
  
  validates :title, presence: true
  validates :content, presence: true
end
```

#### Generated Migration:

```ruby
# db/migrate/create_articles.rb
class CreateArticles < ActiveRecord::Migration[7.0]
  def change
    create_table :articles, id: :uuid do |t|
      t.string :title, null: false
      t.text :content
      t.string :author
      t.json :metadata, default: {}
      t.tsvector :search_vector
      t.references :user, type: :uuid, foreign_key: true

      t.timestamps
    end
    
    add_index :articles, :search_vector, using: :gin
    add_index :articles, :author
  end
end
```

#### Options:

```bash
# Specify content field
rails generate ragdoll:model Post title:string body:text --content-field=body

# Skip user association
rails generate ragdoll:model Document title:string --skip-user

# Custom chunk size
rails generate ragdoll:model Article content:text --chunk-size=800

# Skip search configuration
rails generate ragdoll:model BasicModel name:string --skip-search
```

### ragdoll:searchable

Adds Ragdoll search functionality to an existing model.

```bash
rails generate ragdoll:searchable Article
```

#### Generated Changes:

```ruby
# app/models/article.rb (modified)
class Article < ApplicationRecord
  include Ragdoll::Searchable
  
  ragdoll_searchable do |config|
    config.content_field = :content  # Detected automatically
    config.title_field = :title      # Detected automatically
    config.auto_process = true
  end
  
  # ... existing code
end
```

#### Generated Migration:

```ruby
# db/migrate/add_ragdoll_to_articles.rb
class AddRagdollToArticles < ActiveRecord::Migration[7.0]
  def change
    add_column :articles, :metadata, :json, default: {}
    add_column :articles, :search_vector, :tsvector
    add_column :articles, :ragdoll_status, :string, default: 'pending'
    add_column :articles, :ragdoll_processed_at, :datetime
    
    add_index :articles, :search_vector, using: :gin
    add_index :articles, :ragdoll_status
  end
end
```

## Controller Generators

### ragdoll:controller

Generates a controller with Ragdoll document management functionality.

```bash
rails generate ragdoll:controller Documents
```

#### Generated Controller:

```ruby
# app/controllers/documents_controller.rb
class DocumentsController < ApplicationController
  include Ragdoll::ControllerHelpers
  
  before_action :authenticate_user!
  before_action :set_document, only: [:show, :edit, :update, :destroy]
  
  # GET /documents
  def index
    @documents = current_user.documents
      .includes(:ragdoll_contents)
      .page(params[:page])
      .per(20)
      
    @documents = @documents.where(ragdoll_status: params[:status]) if params[:status].present?
  end
  
  # GET /documents/1
  def show
    @related_documents = @document.find_similar(limit: 5)
  end
  
  # GET /documents/new
  def new
    @document = current_user.documents.build
  end
  
  # POST /documents
  def create
    @document = current_user.documents.build(document_params)
    
    if @document.save
      redirect_to @document, notice: 'Document was successfully created.'
    else
      render :new
    end
  end
  
  # PATCH/PUT /documents/1
  def update
    if @document.update(document_params)
      redirect_to @document, notice: 'Document was successfully updated.'
    else
      render :edit
    end
  end
  
  # DELETE /documents/1
  def destroy
    @document.destroy
    redirect_to documents_url, notice: 'Document was successfully deleted.'
  end
  
  private
  
  def set_document
    @document = current_user.documents.find(params[:id])
  end
  
  def document_params
    params.require(:document).permit(:title, :content, :file, metadata: {})
  end
end
```

#### Generated Views:

The generator creates a complete set of views:

- `app/views/documents/index.html.erb`
- `app/views/documents/show.html.erb`
- `app/views/documents/new.html.erb`
- `app/views/documents/edit.html.erb`
- `app/views/documents/_form.html.erb`

#### Options:

```bash
# Generate API controller instead
rails generate ragdoll:controller Documents --api

# Skip view generation
rails generate ragdoll:controller Documents --skip-views

# Generate with search functionality
rails generate ragdoll:controller Documents --with-search

# Custom parent controller
rails generate ragdoll:controller Documents --parent=AdminController
```

### ragdoll:search_controller

Generates a controller specifically for search functionality.

```bash
rails generate ragdoll:search_controller Search
```

#### Generated Controller:

```ruby
# app/controllers/search_controller.rb
class SearchController < ApplicationController
  include Ragdoll::SearchHelpers
  
  # GET /search
  def index
    return unless params[:q].present?
    
    @query = params[:q]
    @search_results = perform_search(@query, search_options)
    @facets = @search_results.facets if params[:include_facets]
    
    track_search_analytics if respond_to?(:track_search_analytics)
  end
  
  # GET /search/suggestions
  def suggestions
    @suggestions = Ragdoll::SuggestionService.new(params[:q]).call
    render json: @suggestions
  end
  
  # GET /search/autocomplete
  def autocomplete
    @completions = Ragdoll::AutocompleteService.new(params[:q]).call
    render json: @completions
  end
  
  private
  
  def search_options
    {
      limit: params[:limit] || 20,
      page: params[:page] || 1,
      filters: params[:filters] || {},
      search_type: params[:search_type] || 'hybrid',
      include_facets: params[:include_facets]
    }
  end
  
  def perform_search(query, options = {})
    case params[:model]
    when 'Document'
      Ragdoll::Document.search(query, options)
    else
      # Search across all searchable models
      Ragdoll::GlobalSearchService.new(query, options).call
    end
  end
end
```

## View Generators

### ragdoll:views

Copies Ragdoll views to your application for customization.

```bash
rails generate ragdoll:views
```

#### Copies all views:

```
app/views/ragdoll/
├── documents/
│   ├── index.html.erb
│   ├── show.html.erb
│   ├── new.html.erb
│   ├── edit.html.erb
│   └── _form.html.erb
├── search/
│   ├── index.html.erb
│   ├── _search_form.html.erb
│   ├── _search_results.html.erb
│   └── _facets.html.erb
└── shared/
    ├── _document_card.html.erb
    ├── _upload_form.html.erb
    └── _pagination.html.erb
```

#### Options:

```bash
# Copy only specific views
rails generate ragdoll:views --only=documents

# Copy to custom directory
rails generate ragdoll:views --path=admin/ragdoll
```

### ragdoll:layout

Generates a layout file optimized for Ragdoll functionality.

```bash
rails generate ragdoll:layout
```

#### Generated Layout:

```erb
<!-- app/views/layouts/ragdoll.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title>Document Management</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    
    <%= stylesheet_link_tag "ragdoll/application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
    <%= javascript_include_tag "ragdoll/application", "data-turbo-track": "reload" %>
  </head>

  <body>
    <nav class="ragdoll-navbar">
      <div class="container">
        <%= link_to "Documents", ragdoll.documents_path, class: "navbar-brand" %>
        
        <div class="navbar-nav">
          <%= link_to "Upload", ragdoll.new_document_path, class: "nav-link" %>
          <%= link_to "Search", ragdoll.search_path, class: "nav-link" %>
          <% if user_signed_in? %>
            <%= link_to "My Documents", ragdoll.documents_path(user: current_user), class: "nav-link" %>
          <% end %>
        </div>
      </div>
    </nav>
    
    <main class="ragdoll-main">
      <div class="container">
        <% if notice %>
          <div class="alert alert-success alert-dismissible">
            <%= notice %>
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
          </div>
        <% end %>
        
        <% if alert %>
          <div class="alert alert-danger alert-dismissible">
            <%= alert %>
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
          </div>
        <% end %>
        
        <%= yield %>
      </div>
    </main>
  </body>
</html>
```

## Configuration Generators

### ragdoll:config

Generates advanced configuration files.

```bash
rails generate ragdoll:config
```

#### Generated Files:

```ruby
# config/ragdoll.yml
default: &default
  llm:
    provider: openai
    models:
      embedding: "text-embedding-3-small"
      chat: "gpt-4"
  
  processing:
    chunk_size: 1000
    chunk_overlap: 200
    auto_process: true
    background_jobs: true
  
  search:
    default_limit: 20
    similarity_threshold: 0.7
    enable_facets: true

development:
  <<: *default
  llm:
    provider: openai
    api_key: <%= ENV['OPENAI_API_KEY'] %>

test:
  <<: *default
  llm:
    provider: mock

production:
  <<: *default
  llm:
    provider: openai
    api_key: <%= Rails.application.credentials.openai_api_key %>
```

```ruby
# lib/tasks/ragdoll.rake
namespace :ragdoll do
  desc "Process all pending documents"
  task process_pending: :environment do
    Ragdoll::Document.pending.find_each do |document|
      Ragdoll::ProcessDocumentJob.perform_later(document)
    end
  end
  
  desc "Reindex all documents"
  task reindex: :environment do
    Ragdoll::Document.processed.find_each do |document|
      Ragdoll::IndexDocumentJob.perform_later(document)
    end
  end
  
  desc "Clean up failed documents"
  task cleanup_failed: :environment do
    failed_documents = Ragdoll::Document.failed.where('updated_at < ?', 24.hours.ago)
    puts "Cleaning up #{failed_documents.count} failed documents"
    failed_documents.destroy_all
  end
  
  desc "Generate search analytics report"
  task analytics_report: :environment do
    report = Ragdoll::AnalyticsService.new.generate_report
    puts report
  end
end
```

### ragdoll:credentials

Sets up encrypted credentials for LLM providers.

```bash
rails generate ragdoll:credentials
```

#### Adds to credentials:

```yaml
# config/credentials.yml.enc (decrypted view)
openai:
  api_key: sk-your-openai-key-here

anthropic:
  api_key: sk-ant-your-anthropic-key-here

azure:
  api_key: your-azure-key
  endpoint: https://your-resource.openai.azure.com/
```

## Migration Generators

### ragdoll:migration

Generates custom migrations for Ragdoll extensions.

```bash
rails generate ragdoll:migration add_custom_fields_to_documents category:string priority:integer
```

#### Generated Migration:

```ruby
# db/migrate/add_custom_fields_to_ragdoll_documents.rb
class AddCustomFieldsToRagdollDocuments < ActiveRecord::Migration[7.0]
  def change
    add_column :ragdoll_documents, :category, :string
    add_column :ragdoll_documents, :priority, :integer, default: 0
    
    add_index :ragdoll_documents, :category
    add_index :ragdoll_documents, :priority
  end
end
```

### ragdoll:indexes

Generates optimized database indexes for better search performance.

```bash
rails generate ragdoll:indexes
```

#### Generated Migration:

```ruby
# db/migrate/add_ragdoll_performance_indexes.rb
class AddRagdollPerformanceIndexes < ActiveRecord::Migration[7.0]
  def change
    # Vector similarity indexes
    execute <<-SQL
      CREATE INDEX CONCURRENTLY IF NOT EXISTS ragdoll_embeddings_cosine_idx 
      ON ragdoll_embeddings USING ivfflat (embedding vector_cosine_ops) 
      WITH (lists = 100);
    SQL
    
    # Composite indexes for common queries
    add_index :ragdoll_documents, [:user_id, :status, :created_at]
    add_index :ragdoll_documents, [:content_type, :file_size]
    add_index :ragdoll_contents, [:document_id, :chunk_index]
    
    # Full-text search indexes
    execute <<-SQL
      CREATE INDEX CONCURRENTLY IF NOT EXISTS ragdoll_documents_search_idx 
      ON ragdoll_documents USING gin(search_vector);
    SQL
  end
end
```

## Custom Generator Templates

You can customize generator templates by copying them to your application:

```bash
# Copy all generator templates
mkdir -p lib/templates/ragdoll
cp -r $(bundle show ragdoll-rails)/lib/generators/ragdoll/templates/* lib/templates/ragdoll/
```

### Custom Model Template:

```ruby
# lib/templates/ragdoll/model/model.rb.tt
class <%= class_name %> < ApplicationRecord
  include Ragdoll::Searchable
  
  # Associations
  belongs_to :user, optional: true
  
  # Validations
<% attributes.each do |attribute| -%>
  <% if attribute.required? -%>
  validates :<%= attribute.name %>, presence: true
  <% end -%>
<% end -%>
  
  # Ragdoll configuration
  ragdoll_searchable do |config|
    config.content_field = :<%= content_field || 'content' %>
    config.title_field = :<%= title_field || 'title' %>
    config.metadata_fields = [<%= metadata_fields.map { |f| ":#{f}" }.join(', ') %>]
    config.chunk_size = <%= chunk_size || 1000 %>
    config.auto_process = <%= auto_process.nil? ? true : auto_process %>
  end
  
  # Scopes
  scope :by_user, ->(user) { where(user: user) }
  scope :recent, ->(days = 7) { where(created_at: days.days.ago..) }
  
  # Instance methods
  def display_name
    <%= title_field || 'title' %> || "Untitled <%= class_name %>"
  end
end
```

## Generator Usage Examples

### Complete Setup Workflow:

```bash
# 1. Install Ragdoll
rails generate ragdoll:install

# 2. Generate a document model
rails generate ragdoll:model Document title:string content:text category:string

# 3. Generate search functionality
rails generate ragdoll:search_controller Search

# 4. Copy views for customization
rails generate ragdoll:views --only=documents,search

# 5. Add performance indexes
rails generate ragdoll:indexes

# 6. Run migrations
rails db:migrate

# 7. Set up credentials
rails generate ragdoll:credentials
EDITOR="code --wait" rails credentials:edit
```

### Adding Search to Existing Model:

```bash
# Add Ragdoll search to existing Article model
rails generate ragdoll:searchable Article

# Generate controller with search
rails generate ragdoll:controller Articles --with-search

# Run the migration
rails db:migrate

# Process existing articles
rails ragdoll:process_pending
```

This comprehensive generator documentation provides everything you need to quickly set up and customize Ragdoll Rails in your application using the provided generators.