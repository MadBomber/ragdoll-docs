# Rails Examples

Real-world examples and implementation patterns for using ragdoll-rails in various scenarios.

## Complete Application Examples

### Knowledge Base Application

A complete knowledge base application using Ragdoll Rails.

#### Models

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :articles, dependent: :destroy
  has_many :saved_searches, dependent: :destroy
  
  enum role: { viewer: 0, editor: 1, admin: 2 }
  
  def can_upload_documents?
    editor? || admin?
  end
end

# app/models/article.rb
class Article < ApplicationRecord
  include Ragdoll::Searchable
  
  belongs_to :user
  belongs_to :category, optional: true
  has_many_attached :attachments
  
  validates :title, presence: true, length: { maximum: 255 }
  validates :content, presence: true
  validates :status, inclusion: { in: %w[draft published archived] }
  
  ragdoll_searchable do |config|
    config.content_field = :content
    config.title_field = :title
    config.metadata_fields = [:category_name, :tags, :author_name, :status]
    config.chunk_size = 800
    config.auto_process = true
    config.process_on_create = true
    config.process_on_update = true
    
    config.custom_metadata = ->(article) {
      {
        category_name: article.category&.name,
        author_name: article.user.name,
        word_count: article.content.split.size,
        reading_time: (article.content.split.size / 200.0).ceil,
        tags: article.tag_list,
        last_updated: article.updated_at.iso8601
      }
    }
  end
  
  scope :published, -> { where(status: 'published') }
  scope :by_category, ->(category) { where(category: category) }
  scope :by_user, ->(user) { where(user: user) }
  scope :recent, ->(days = 30) { where(created_at: days.days.ago..) }
  
  def category_name
    category&.name
  end
  
  def author_name
    user.name
  end
  
  def tag_list
    tags.split(',').map(&:strip) if tags.present?
  end
  
  def reading_time_minutes
    (content.split.size / 200.0).ceil
  end
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :articles, dependent: :destroy
  
  validates :name, presence: true, uniqueness: true
  validates :description, presence: true
  
  scope :with_articles, -> { joins(:articles).distinct }
  
  def article_count
    articles.published.count
  end
end

# app/models/saved_search.rb
class SavedSearch < ApplicationRecord
  belongs_to :user
  
  validates :name, presence: true
  validates :query, presence: true
  
  scope :public_searches, -> { where(is_public: true) }
  scope :by_user, ->(user) { where(user: user) }
  
  def execute
    options = filters.present? ? { filters: filters } : {}
    Article.search(query, options)
  end
end
```

#### Controllers

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :configure_permitted_parameters, if: :devise_controller?
  
  protected
  
  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
    devise_parameter_sanitizer.permit(:account_update, keys: [:name])
  end
end

# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :check_edit_permission, only: [:edit, :update, :destroy]
  
  def index
    @articles = Article.published.includes(:user, :category)
    @articles = @articles.by_category(params[:category_id]) if params[:category_id].present?
    @articles = @articles.page(params[:page]).per(20)
    
    @categories = Category.with_articles
  end
  
  def show
    @related_articles = @article.find_similar(limit: 5)
      .where.not(id: @article.id)
      .where(status: 'published')
  end
  
  def new
    @article = current_user.articles.build
    @categories = Category.all
  end
  
  def create
    @article = current_user.articles.build(article_params)
    
    if @article.save
      redirect_to @article, notice: 'Article was successfully created.'
    else
      @categories = Category.all
      render :new
    end
  end
  
  def edit
    @categories = Category.all
  end
  
  def update
    if @article.update(article_params)
      redirect_to @article, notice: 'Article was successfully updated.'
    else
      @categories = Category.all
      render :edit
    end
  end
  
  def destroy
    @article.destroy
    redirect_to articles_url, notice: 'Article was successfully deleted.'
  end
  
  private
  
  def set_article
    @article = Article.find(params[:id])
  end
  
  def article_params
    params.require(:article).permit(:title, :content, :category_id, :tags, :status, attachments: [])
  end
  
  def check_edit_permission
    redirect_to articles_path, alert: 'Access denied.' unless can_edit_article?(@article)
  end
  
  def can_edit_article?(article)
    current_user.admin? || article.user == current_user
  end
end

# app/controllers/search_controller.rb
class SearchController < ApplicationController
  before_action :set_search_params, only: [:index, :suggestions]
  
  def index
    return unless @query.present?
    
    @search_service = ArticleSearchService.new(@query, search_options)
    @results = @search_service.call
    @facets = @search_service.facets
    @suggestions = @search_service.suggestions if @results.empty?
    
    track_search_analytics
  end
  
  def suggestions
    @suggestions = ArticleSuggestionService.new(@query).call
    render json: @suggestions
  end
  
  def saved_searches
    @saved_searches = current_user.saved_searches.order(:name)
    @public_searches = SavedSearch.public_searches.includes(:user).limit(10)
  end
  
  def save_search
    @saved_search = current_user.saved_searches.build(saved_search_params)
    
    if @saved_search.save
      render json: { success: true, id: @saved_search.id }
    else
      render json: { success: false, errors: @saved_search.errors }
    end
  end
  
  private
  
  def set_search_params
    @query = params[:q]
    @filters = params[:filters] || {}
    @sort = params[:sort] || 'relevance'
    @page = params[:page] || 1
  end
  
  def search_options
    {
      filters: @filters,
      sort: @sort,
      page: @page,
      per_page: 15,
      include_facets: true,
      search_type: params[:search_type] || 'hybrid'
    }
  end
  
  def track_search_analytics
    SearchAnalytic.create!(
      user: current_user,
      query: @query,
      results_count: @results.size,
      search_type: params[:search_type] || 'hybrid',
      filters: @filters,
      response_time: @search_service.search_time
    )
  end
  
  def saved_search_params
    params.require(:saved_search).permit(:name, :query, :is_public, filters: {})
  end
end
```

#### Services

```ruby
# app/services/article_search_service.rb
class ArticleSearchService
  attr_reader :search_time
  
  def initialize(query, options = {})
    @query = query
    @options = options
    @search_time = 0
  end
  
  def call
    start_time = Time.current
    
    results = perform_search
    results = apply_filters(results) if @options[:filters].present?
    results = apply_sorting(results)
    results = paginate_results(results)
    
    @search_time = Time.current - start_time
    
    add_facets(results) if @options[:include_facets]
    
    results
  end
  
  def facets
    return {} unless @options[:include_facets]
    
    {
      'category' => category_facets,
      'author' => author_facets,
      'status' => status_facets,
      'created_at' => date_facets
    }
  end
  
  def suggestions
    return [] if @query.blank?
    
    # Use a suggestion service or implement basic suggestions
    ArticleSuggestionService.new(@query).call
  end
  
  private
  
  def perform_search
    case @options[:search_type]
    when 'semantic'
      Article.semantic_search(@query, limit: 100)
    when 'keyword'
      Article.keyword_search(@query).limit(100)
    else
      Article.search(@query, limit: 100)
    end
  end
  
  def apply_filters(results)
    filtered = results
    
    if @options[:filters][:category].present?
      category = Category.find(@options[:filters][:category])
      filtered = filtered.where(category: category)
    end
    
    if @options[:filters][:author].present?
      user = User.find(@options[:filters][:author])
      filtered = filtered.where(user: user)
    end
    
    if @options[:filters][:date_range].present?
      case @options[:filters][:date_range]
      when 'week'
        filtered = filtered.where(created_at: 1.week.ago..)
      when 'month'
        filtered = filtered.where(created_at: 1.month.ago..)
      when 'year'
        filtered = filtered.where(created_at: 1.year.ago..)
      end
    end
    
    filtered
  end
  
  def apply_sorting(results)
    case @options[:sort]
    when 'date_desc'
      results.order(created_at: :desc)
    when 'date_asc'
      results.order(created_at: :asc)
    when 'title'
      results.order(:title)
    else
      results # Keep relevance sorting from search
    end
  end
  
  def paginate_results(results)
    page = @options[:page] || 1
    per_page = @options[:per_page] || 15
    
    results.page(page).per(per_page)
  end
  
  def add_facets(results)
    # Add facet information to results
    results.define_singleton_method(:facets) { facets }
  end
  
  def category_facets
    Article.published.joins(:category)
      .group('categories.name')
      .count
      .map { |name, count| { name: name, count: count } }
  end
  
  def author_facets
    Article.published.joins(:user)
      .group('users.name')
      .count
      .map { |name, count| { name: name, count: count } }
      .sort_by { |item| -item[:count] }
      .first(10)
  end
  
  def status_facets
    Article.group(:status).count
      .map { |status, count| { name: status.humanize, value: status, count: count } }
  end
  
  def date_facets
    [
      { name: 'Past Week', value: 'week', count: Article.where(created_at: 1.week.ago..).count },
      { name: 'Past Month', value: 'month', count: Article.where(created_at: 1.month.ago..).count },
      { name: 'Past Year', value: 'year', count: Article.where(created_at: 1.year.ago..).count }
    ]
  end
end

# app/services/article_suggestion_service.rb
class ArticleSuggestionService
  def initialize(query)
    @query = query.to_s.downcase.strip
  end
  
  def call
    return [] if @query.blank? || @query.length < 3
    
    suggestions = []
    
    # Title-based suggestions
    suggestions += title_suggestions
    
    # Tag-based suggestions
    suggestions += tag_suggestions
    
    # Category-based suggestions
    suggestions += category_suggestions
    
    suggestions.uniq.first(5)
  end
  
  private
  
  def title_suggestions
    Article.published
      .where("LOWER(title) LIKE ?", "%#{@query}%")
      .limit(3)
      .pluck(:title)
  end
  
  def tag_suggestions
    # Assuming tags are stored as comma-separated strings
    Article.published
      .where("LOWER(tags) LIKE ?", "%#{@query}%")
      .pluck(:tags)
      .flat_map { |tags| tags.split(',').map(&:strip) }
      .select { |tag| tag.downcase.include?(@query) }
      .uniq
      .first(2)
  end
  
  def category_suggestions
    Category.where("LOWER(name) LIKE ?", "%#{@query}%")
      .pluck(:name)
      .first(2)
  end
end
```

#### Views

```erb
<!-- app/views/articles/index.html.erb -->
<div class="articles-index">
  <div class="page-header">
    <h1>Knowledge Base</h1>
    <div class="header-actions">
      <%= link_to "Search Articles", search_path, class: "btn btn-outline-primary" %>
      <% if current_user.can_upload_documents? %>
        <%= link_to "New Article", new_article_path, class: "btn btn-primary" %>
      <% end %>
    </div>
  </div>
  
  <div class="filters-section">
    <%= form_with url: articles_path, method: :get, local: true, class: "filter-form" do |form| %>
      <div class="filter-group">
        <%= form.select :category_id, 
            options_from_collection_for_select(@categories, :id, :name, params[:category_id]), 
            { prompt: 'All Categories' }, 
            { class: "form-select" } %>
      </div>
      <%= form.submit "Filter", class: "btn btn-outline-primary" %>
    <% end %>
  </div>
  
  <div class="articles-grid">
    <% @articles.each do |article| %>
      <div class="article-card">
        <div class="article-content">
          <h3 class="article-title">
            <%= link_to article.title, article_path(article) %>
          </h3>
          
          <p class="article-excerpt">
            <%= truncate(strip_tags(article.content), length: 150) %>
          </p>
          
          <div class="article-meta">
            <span class="author">By <%= article.author_name %></span>
            <span class="date"><%= article.created_at.strftime("%B %d, %Y") %></span>
            <% if article.category %>
              <span class="category"><%= article.category.name %></span>
            <% end %>
            <span class="reading-time"><%= article.reading_time_minutes %> min read</span>
          </div>
          
          <% if article.tag_list.present? %>
            <div class="article-tags">
              <% article.tag_list.each do |tag| %>
                <span class="tag"><%= tag %></span>
              <% end %>
            </div>
          <% end %>
        </div>
      </div>
    <% end %>
  </div>
  
  <div class="pagination-wrapper">
    <%= paginate @articles %>
  </div>
</div>
```

```erb
<!-- app/views/articles/show.html.erb -->
<div class="article-show">
  <div class="article-header">
    <div class="breadcrumb">
      <%= link_to "Articles", articles_path %>
      <% if @article.category %>
        → <%= link_to @article.category.name, articles_path(category_id: @article.category.id) %>
      <% end %>
      → <%= @article.title %>
    </div>
    
    <h1 class="article-title"><%= @article.title %></h1>
    
    <div class="article-meta">
      <div class="author-info">
        <strong><%= @article.author_name %></strong>
        <span class="publish-date">
          Published <%= @article.created_at.strftime("%B %d, %Y") %>
        </span>
        <% if @article.updated_at > @article.created_at + 1.hour %>
          <span class="update-date">
            (Updated <%= @article.updated_at.strftime("%B %d, %Y") %>)
          </span>
        <% end %>
      </div>
      
      <div class="article-stats">
        <span class="reading-time">
          <i class="bi bi-clock"></i> <%= @article.reading_time_minutes %> min read
        </span>
        <% if @article.category %>
          <span class="category">
            <i class="bi bi-tag"></i> <%= @article.category.name %>
          </span>
        <% end %>
      </div>
    </div>
    
    <% if can_edit_article?(@article) %>
      <div class="article-actions">
        <%= link_to "Edit", edit_article_path(@article), class: "btn btn-secondary" %>
        <%= link_to "Delete", article_path(@article), method: :delete,
                    confirm: "Are you sure?", class: "btn btn-outline-danger" %>
      </div>
    <% end %>
  </div>
  
  <div class="article-content">
    <%= simple_format(@article.content) %>
  </div>
  
  <% if @article.attachments.any? %>
    <div class="article-attachments">
      <h3>Attachments</h3>
      <% @article.attachments.each do |attachment| %>
        <div class="attachment">
          <%= link_to attachment.filename, rails_blob_path(attachment, disposition: "attachment") %>
          <span class="file-size">(<%= number_to_human_size(attachment.byte_size) %>)</span>
        </div>
      <% end %>
    </div>
  <% end %>
  
  <% if @article.tag_list.present? %>
    <div class="article-tags">
      <h4>Tags:</h4>
      <% @article.tag_list.each do |tag| %>
        <span class="tag"><%= tag %></span>
      <% end %>
    </div>
  <% end %>
  
  <% if @related_articles.any? %>
    <div class="related-articles">
      <h3>Related Articles</h3>
      <div class="related-grid">
        <% @related_articles.each do |article| %>
          <div class="related-card">
            <h4><%= link_to article.title, article_path(article) %></h4>
            <p><%= truncate(strip_tags(article.content), length: 100) %></p>
            <small>By <%= article.author_name %></small>
          </div>
        <% end %>
      </div>
    </div>
  <% end %>
</div>
```

```erb
<!-- app/views/search/index.html.erb -->
<div class="search-page">
  <div class="search-header">
    <h1>Search Knowledge Base</h1>
  </div>
  
  <div class="search-form">
    <%= form_with url: search_path, method: :get, local: true, 
                  data: { controller: "search", search_suggestions_url_value: suggestions_search_path } do |form| %>
      <div class="search-input-group">
        <%= form.text_field :q, value: params[:q], 
                           placeholder: "Search articles, categories, or tags...", 
                           class: "search-input form-control",
                           data: { 
                             action: "input->search#suggest",
                             search_target: "input"
                           } %>
        <%= form.submit "Search", class: "btn btn-primary" %>
      </div>
      
      <div class="search-options">
        <div class="row">
          <div class="col-md-4">
            <%= form.select :search_type, [
              ['Smart Search (Recommended)', 'hybrid'],
              ['Meaning-based Search', 'semantic'],
              ['Exact Word Search', 'keyword']
            ], { selected: params[:search_type] || 'hybrid' }, { class: "form-select" } %>
          </div>
          <div class="col-md-4">
            <%= form.select :sort, [
              ['Most Relevant', 'relevance'],
              ['Newest First', 'date_desc'],
              ['Oldest First', 'date_asc'],
              ['Alphabetical', 'title']
            ], { selected: params[:sort] || 'relevance' }, { class: "form-select" } %>
          </div>
        </div>
      </div>
    <% end %>
    
    <div class="search-suggestions" data-search-target="suggestions" style="display: none;">
      <!-- Populated via Stimulus -->
    </div>
  </div>
  
  <% if @results.present? %>
    <div class="search-results-section">
      <div class="search-meta">
        <p>Found <strong><%= pluralize(@results.total_count, 'article') %></strong> 
           <% if params[:q].present? %>for "<strong><%= params[:q] %></strong>"<% end %>
           <span class="search-time">in <%= number_with_precision(@search_service.search_time, precision: 3) %>s</span>
        </p>
      </div>
      
      <div class="search-content">
        <% if @facets.present? %>
          <div class="search-facets">
            <h4>Filter Results</h4>
            
            <% @facets.each do |facet_name, facet_data| %>
              <div class="facet-group">
                <h5><%= facet_name.humanize %></h5>
                <% facet_data.each do |item| %>
                  <div class="facet-item">
                    <%= link_to "#{item[:name]} (#{item[:count]})", 
                                search_path(q: params[:q], filters: { facet_name => item[:value] || item[:name] }),
                                class: "facet-link" %>
                  </div>
                <% end %>
              </div>
            <% end %>
          </div>
        <% end %>
        
        <div class="search-results">
          <% @results.each do |article| %>
            <div class="search-result">
              <div class="result-header">
                <h3 class="result-title">
                  <%= link_to article.title, article_path(article) %>
                </h3>
                <div class="result-meta">
                  <% if respond_to?(:similarity_score) && article.respond_to?(:similarity_score) %>
                    <span class="similarity-score">
                      <%= number_to_percentage(article.similarity_score * 100, precision: 1) %> match
                    </span>
                  <% end %>
                  <span class="author">by <%= article.author_name %></span>
                  <span class="date"><%= article.created_at.strftime("%b %d, %Y") %></span>
                  <% if article.category %>
                    <span class="category"><%= article.category.name %></span>
                  <% end %>
                </div>
              </div>
              
              <div class="result-content">
                <p class="result-snippet">
                  <%= search_result_snippet(article.content, params[:q]) %>
                </p>
              </div>
            </div>
          <% end %>
          
          <div class="search-pagination">
            <%= paginate @results %>
          </div>
        </div>
      </div>
    </div>
  <% elsif params[:q].present? %>
    <div class="no-results">
      <h3>No articles found</h3>
      <p>Try different keywords or check your spelling.</p>
      
      <% if @suggestions.present? %>
        <div class="search-suggestions">
          <p>Did you mean:</p>
          <ul>
            <% @suggestions.each do |suggestion| %>
              <li><%= link_to suggestion, search_path(q: suggestion) %></li>
            <% end %>
          </ul>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

### Document Management System

A corporate document management system with advanced features.

#### Advanced Model with Versioning

```ruby
# app/models/document.rb
class Document < ApplicationRecord
  include Ragdoll::Searchable
  
  belongs_to :user
  belongs_to :department
  has_many :document_versions, dependent: :destroy
  has_many :document_shares, dependent: :destroy
  has_many :shared_with_users, through: :document_shares, source: :user
  has_many :document_comments, dependent: :destroy
  has_one_attached :file
  
  validates :title, presence: true
  validates :visibility, inclusion: { in: %w[private department company public] }
  
  ragdoll_searchable do |config|
    config.content_field = :extracted_content
    config.title_field = :title
    config.metadata_fields = [:department_name, :document_type, :tags, :visibility]
    config.chunk_size = 1200
    config.auto_process = true
    
    config.custom_metadata = ->(doc) {
      {
        department_name: doc.department.name,
        document_type: doc.document_type,
        file_extension: doc.file_extension,
        version: doc.current_version,
        shared_count: doc.document_shares.count,
        comment_count: doc.document_comments.count,
        last_accessed: doc.last_accessed_at,
        security_level: doc.security_level
      }
    }
  end
  
  enum document_type: {
    policy: 0,
    procedure: 1,
    manual: 2,
    report: 3,
    presentation: 4,
    specification: 5,
    other: 99
  }
  
  enum security_level: {
    public: 0,
    internal: 1,
    confidential: 2,
    restricted: 3
  }
  
  scope :accessible_by, ->(user) {
    where(
      "(visibility = 'public') OR " \
      "(visibility = 'company' AND :user_id IS NOT NULL) OR " \
      "(visibility = 'department' AND department_id = :dept_id) OR " \
      "(user_id = :user_id) OR " \
      "(id IN (SELECT document_id FROM document_shares WHERE user_id = :user_id))",
      user_id: user&.id,
      dept_id: user&.department_id
    )
  }
  
  def create_version!
    self.current_version += 1
    document_versions.create!(
      version_number: current_version,
      title: title,
      content: extracted_content,
      file_data: file.attached? ? file.blob : nil,
      created_by: self.user
    )
    save!
  end
  
  def share_with!(user, permission: 'read')
    document_shares.find_or_create_by(user: user) do |share|
      share.permission = permission
    end
  end
  
  def accessible_by?(user)
    return true if user == self.user
    
    case visibility
    when 'public' then true
    when 'company' then user.present?
    when 'department' then user&.department == department
    when 'private' then false
    else false
    end || document_shares.exists?(user: user)
  end
  
  def track_access!(user)
    update!(
      last_accessed_at: Time.current,
      access_count: access_count + 1
    )
    
    DocumentAccess.create!(
      document: self,
      user: user,
      accessed_at: Time.current,
      ip_address: Current.ip_address,
      user_agent: Current.user_agent
    )
  end
end

# app/models/document_version.rb
class DocumentVersion < ApplicationRecord
  belongs_to :document
  belongs_to :created_by, class_name: 'User'
  has_one_attached :file
  
  validates :version_number, presence: true, uniqueness: { scope: :document_id }
  
  scope :ordered, -> { order(version_number: :desc) }
  
  def restore!
    document.update!(
      title: title,
      extracted_content: content,
      current_version: version_number
    )
    
    if file_data
      document.file.attach(file_data)
      document.ragdoll_process!
    end
  end
end

# app/models/document_share.rb
class DocumentShare < ApplicationRecord
  belongs_to :document
  belongs_to :user
  
  validates :permission, inclusion: { in: %w[read write admin] }
  validates :user_id, uniqueness: { scope: :document_id }
  
  scope :with_write_access, -> { where(permission: %w[write admin]) }
  scope :with_admin_access, -> { where(permission: 'admin') }
end
```

#### Advanced Search with Filters

```ruby
# app/services/advanced_document_search_service.rb
class AdvancedDocumentSearchService
  include ActionView::Helpers::DateHelper
  
  attr_reader :search_time, :total_results
  
  def initialize(query, user, options = {})
    @query = query
    @user = user
    @options = options
    @search_time = 0
    @total_results = 0
  end
  
  def call
    start_time = Time.current
    
    results = perform_search
    results = apply_security_filters(results)
    results = apply_advanced_filters(results)
    results = apply_sorting(results)
    results = paginate_results(results)
    
    @search_time = Time.current - start_time
    @total_results = results.total_count
    
    enhance_results(results)
  end
  
  def facets
    {
      'document_type' => document_type_facets,
      'department' => department_facets,
      'security_level' => security_level_facets,
      'file_type' => file_type_facets,
      'date_range' => date_range_facets,
      'author' => author_facets
    }
  end
  
  private
  
  def perform_search
    if @query.present?
      Document.search(@query, search_options)
    else
      Document.accessible_by(@user)
    end
  end
  
  def search_options
    {
      limit: 1000, # Get more results for filtering
      search_type: @options[:search_type] || 'hybrid',
      threshold: @options[:threshold] || 0.5
    }
  end
  
  def apply_security_filters(results)
    # Additional security filtering beyond basic accessibility
    filtered = results.accessible_by(@user)
    
    # Filter by security clearance
    if @user.security_clearance.present?
      max_level = security_level_mapping[@user.security_clearance]
      filtered = filtered.where(security_level: 0..max_level)
    end
    
    filtered
  end
  
  def apply_advanced_filters(results)
    filtered = results
    
    # Document type filter
    if @options[:document_type].present?
      filtered = filtered.where(document_type: @options[:document_type])
    end
    
    # Department filter
    if @options[:department_id].present?
      filtered = filtered.where(department_id: @options[:department_id])
    end
    
    # File type filter
    if @options[:file_type].present?
      filtered = filtered.joins(:file_attachment)
        .where(active_storage_blobs: { content_type: @options[:file_type] })
    end
    
    # Date range filter
    if @options[:date_range].present?
      filtered = apply_date_filter(filtered, @options[:date_range])
    end
    
    # Security level filter
    if @options[:security_level].present?
      filtered = filtered.where(security_level: @options[:security_level])
    end
    
    # Author filter
    if @options[:author_id].present?
      filtered = filtered.where(user_id: @options[:author_id])
    end
    
    # Tags filter
    if @options[:tags].present?
      tag_conditions = @options[:tags].map { "tags ILIKE ?" }
      tag_values = @options[:tags].map { |tag| "%#{tag}%" }
      filtered = filtered.where(tag_conditions.join(' AND '), *tag_values)
    end
    
    # File size filter
    if @options[:min_size].present? || @options[:max_size].present?
      filtered = apply_file_size_filter(filtered)
    end
    
    filtered
  end
  
  def apply_date_filter(results, date_range)
    case date_range
    when 'today'
      results.where(created_at: Date.current.beginning_of_day..)
    when 'week'
      results.where(created_at: 1.week.ago..)
    when 'month'
      results.where(created_at: 1.month.ago..)
    when 'quarter'
      results.where(created_at: 3.months.ago..)
    when 'year'
      results.where(created_at: 1.year.ago..)
    when 'custom'
      if @options[:start_date] && @options[:end_date]
        start_date = Date.parse(@options[:start_date])
        end_date = Date.parse(@options[:end_date])
        results.where(created_at: start_date.beginning_of_day..end_date.end_of_day)
      else
        results
      end
    else
      results
    end
  end
  
  def apply_file_size_filter(results)
    joins_clause = <<~SQL
      LEFT JOIN active_storage_attachments asa ON asa.record_id = documents.id 
        AND asa.record_type = 'Document' AND asa.name = 'file'
      LEFT JOIN active_storage_blobs asb ON asb.id = asa.blob_id
    SQL
    
    filtered = results.joins(joins_clause)
    
    if @options[:min_size].present?
      min_bytes = parse_file_size(@options[:min_size])
      filtered = filtered.where('asb.byte_size >= ?', min_bytes)
    end
    
    if @options[:max_size].present?
      max_bytes = parse_file_size(@options[:max_size])
      filtered = filtered.where('asb.byte_size <= ?', max_bytes)
    end
    
    filtered
  end
  
  def apply_sorting(results)
    case @options[:sort]
    when 'created_desc'
      results.order(created_at: :desc)
    when 'created_asc'
      results.order(created_at: :asc)
    when 'updated_desc'
      results.order(updated_at: :desc)
    when 'title_asc'
      results.order(:title)
    when 'title_desc'
      results.order(title: :desc)
    when 'size_desc'
      results.joins(:file_attachment, :file_blob).order('active_storage_blobs.byte_size DESC')
    when 'access_count'
      results.order(access_count: :desc)
    else
      results # Keep relevance ordering from search
    end
  end
  
  def paginate_results(results)
    page = @options[:page] || 1
    per_page = [@options[:per_page] || 20, 100].min
    
    results.page(page).per(per_page)
  end
  
  def enhance_results(results)
    # Add additional data to results
    results.each do |document|
      # Track that this document appeared in search results
      SearchResult.create!(
        user: @user,
        document: document,
        query: @query,
        position: results.index(document) + 1,
        similarity_score: document.respond_to?(:similarity_score) ? document.similarity_score : nil
      )
    end
    
    results
  end
  
  def document_type_facets
    accessible_documents.group(:document_type).count
      .map { |type, count| { name: type.humanize, value: type, count: count } }
  end
  
  def department_facets
    accessible_documents.joins(:department)
      .group('departments.name')
      .count
      .map { |name, count| { name: name, value: name, count: count } }
  end
  
  def security_level_facets
    accessible_documents.group(:security_level).count
      .map { |level, count| { name: level.humanize, value: level, count: count } }
  end
  
  def file_type_facets
    accessible_documents.joins(:file_attachment, :file_blob)
      .group('active_storage_blobs.content_type')
      .count
      .map { |type, count| { name: format_content_type(type), value: type, count: count } }
      .sort_by { |item| -item[:count] }
      .first(10)
  end
  
  def date_range_facets
    [
      { name: 'Today', value: 'today', count: accessible_documents.where(created_at: Date.current.beginning_of_day..).count },
      { name: 'This Week', value: 'week', count: accessible_documents.where(created_at: 1.week.ago..).count },
      { name: 'This Month', value: 'month', count: accessible_documents.where(created_at: 1.month.ago..).count },
      { name: 'This Quarter', value: 'quarter', count: accessible_documents.where(created_at: 3.months.ago..).count },
      { name: 'This Year', value: 'year', count: accessible_documents.where(created_at: 1.year.ago..).count }
    ]
  end
  
  def author_facets
    accessible_documents.joins(:user)
      .group('users.name')
      .count
      .map { |name, count| { name: name, count: count } }
      .sort_by { |item| -item[:count] }
      .first(15)
  end
  
  def accessible_documents
    @accessible_documents ||= Document.accessible_by(@user)
  end
  
  def security_level_mapping
    {
      'basic' => 0,
      'standard' => 1,
      'elevated' => 2,
      'top_secret' => 3
    }
  end
  
  def format_content_type(content_type)
    case content_type
    when 'application/pdf' then 'PDF Documents'
    when /^image\// then 'Images'
    when /word|doc/ then 'Word Documents'
    when /excel|sheet/ then 'Spreadsheets'
    when /powerpoint|presentation/ then 'Presentations'
    else content_type.humanize
    end
  end
  
  def parse_file_size(size_string)
    # Parse size strings like "10MB", "500KB", "2GB"
    size_string = size_string.to_s.upcase
    number = size_string.scan(/\d+/).first.to_f
    unit = size_string.scan(/[A-Z]+/).first
    
    case unit
    when 'KB' then number * 1024
    when 'MB' then number * 1024 * 1024
    when 'GB' then number * 1024 * 1024 * 1024
    else number
    end
  end
end
```

This comprehensive example demonstrates how to build sophisticated document management applications using ragdoll-rails with advanced search, security, versioning, and analytics features.