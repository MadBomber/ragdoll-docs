# Rails Controllers

The ragdoll-rails gem provides a comprehensive set of controllers that handle document management, search functionality, and administrative tasks. These controllers follow Rails conventions and can be easily customized or extended.

## Controller Overview

The Ragdoll Rails controllers are organized into logical groups:

- **Document Management**: Upload, view, edit, and delete documents
- **Search**: Perform searches and manage search interfaces  
- **Content**: Handle content chunks and embeddings
- **Admin**: Administrative functions and monitoring
- **API**: RESTful API endpoints

## Core Controllers

### Ragdoll::DocumentsController

The main controller for document operations.

#### Actions

```ruby
class Ragdoll::DocumentsController < Ragdoll::ApplicationController
  before_action :authenticate_user!, except: [:show] if Ragdoll.configuration.require_authentication
  before_action :set_document, only: [:show, :edit, :update, :destroy, :download, :reprocess]
  before_action :authorize_document!, only: [:show, :edit, :update, :destroy]
  
  # GET /documents
  def index
    @documents = documents_scope
      .includes(:user, :contents)
      .page(params[:page])
      .per(params[:per_page] || 20)
    
    @documents = @documents.where(status: params[:status]) if params[:status].present?
    @documents = @documents.where(content_type: params[:content_type]) if params[:content_type].present?
    
    respond_to do |format|
      format.html
      format.json { render json: @documents }
      format.csv { send_data documents_csv, filename: "documents-#{Date.current}.csv" }
    end
  end
  
  # GET /documents/1
  def show
    @content_preview = @document.content_preview(1000)
    @related_documents = @document.find_similar(limit: 5)
    
    respond_to do |format|
      format.html
      format.json { render json: @document, include: [:contents, :embeddings] }
      format.pdf { render_pdf }
    end
  end
  
  # GET /documents/new
  def new
    @document = documents_scope.build
  end
  
  # GET /documents/1/edit
  def edit
  end
  
  # POST /documents
  def create
    @document = documents_scope.build(document_params)
    @document.user = current_user if respond_to?(:current_user)
    
    respond_to do |format|
      if @document.save
        enqueue_processing if should_auto_process?
        format.html { redirect_to @document, notice: 'Document was successfully created.' }
        format.json { render json: @document, status: :created }
      else
        format.html { render :new }
        format.json { render json: @document.errors, status: :unprocessable_entity }
      end
    end
  end
  
  # PATCH/PUT /documents/1
  def update
    respond_to do |format|
      if @document.update(document_params)
        enqueue_reprocessing if file_changed?
        format.html { redirect_to @document, notice: 'Document was successfully updated.' }
        format.json { render json: @document }
      else
        format.html { render :edit }
        format.json { render json: @document.errors, status: :unprocessable_entity }
      end
    end
  end
  
  # DELETE /documents/1
  def destroy
    @document.destroy
    respond_to do |format|
      format.html { redirect_to documents_url, notice: 'Document was successfully deleted.' }
      format.json { head :no_content }
    end
  end
  
  # GET /documents/1/download
  def download
    authorize_download!
    redirect_to rails_blob_path(@document.file, disposition: "attachment")
  end
  
  # POST /documents/1/reprocess
  def reprocess
    authorize_reprocess!
    Ragdoll::ProcessDocumentJob.perform_later(@document)
    redirect_to @document, notice: 'Document reprocessing has been queued.'
  end
  
  private
  
  def documents_scope
    if respond_to?(:current_user) && current_user
      Ragdoll::Document.accessible_by(current_user)
    else
      Ragdoll::Document.public_documents
    end
  end
  
  def set_document
    @document = documents_scope.find(params[:id])
  end
  
  def document_params
    params.require(:document).permit(:title, :description, :file, :metadata, images: [])
  end
  
  def authorize_document!
    return if Ragdoll.configuration.authorize_document_access.call(@document, current_user)
    
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Access denied.' }
      format.json { render json: { error: 'Access denied' }, status: :forbidden }
    end
  end
  
  def should_auto_process?
    Ragdoll.configuration.auto_process_documents && @document.file.attached?
  end
  
  def enqueue_processing
    Ragdoll::ProcessDocumentJob.perform_later(@document)
  end
  
  def enqueue_reprocessing
    return unless @document.file.attached?
    
    @document.update!(status: 'pending')
    Ragdoll::ProcessDocumentJob.perform_later(@document)
  end
  
  def file_changed?
    @document.saved_change_to_attribute?(:file) || @document.file.attachment.changed?
  end
  
  def render_pdf
    # Custom PDF rendering logic
    pdf = Ragdoll::PdfRenderer.new(@document).render
    send_data pdf, filename: "#{@document.title}.pdf", type: 'application/pdf'
  end
  
  def documents_csv
    Ragdoll::CsvExporter.new(@documents).export
  end
end
```

#### Custom Actions

```ruby
class Ragdoll::DocumentsController < Ragdoll::ApplicationController
  # GET /documents/bulk_upload
  def bulk_upload
    @upload_session = Ragdoll::UploadSession.new
  end
  
  # POST /documents/bulk_create
  def bulk_create
    @upload_session = Ragdoll::UploadSession.new(bulk_upload_params)
    
    if @upload_session.valid?
      @upload_session.process_files
      redirect_to documents_path, notice: "#{@upload_session.file_count} files uploaded successfully."
    else
      render :bulk_upload
    end
  end
  
  # GET /documents/stats
  def stats
    @stats = {
      total_documents: documents_scope.count,
      processed_documents: documents_scope.processed.count,
      total_size: documents_scope.sum(:file_size),
      recent_uploads: documents_scope.recent(7).count,
      top_content_types: documents_scope.group(:content_type).count.sort_by(&:last).reverse.first(5)
    }
    
    respond_to do |format|
      format.html
      format.json { render json: @stats }
    end
  end
  
  private
  
  def bulk_upload_params
    params.require(:upload_session).permit(files: [], metadata: {})
  end
end
```

### Ragdoll::SearchController

Handles search functionality and interfaces.

```ruby
class Ragdoll::SearchController < Ragdoll::ApplicationController
  before_action :authenticate_user!, if: :authentication_required?
  before_action :authorize_search!, if: :authorization_required?
  before_action :set_search_params, only: [:index, :suggestions, :facets]
  
  # GET /search
  def index
    @search_service = Ragdoll::SearchService.new(@query, search_options)
    @results = @search_service.call
    @facets = @search_service.facets if params[:include_facets]
    @suggestions = @search_service.suggestions if @results.empty?
    
    # Track search analytics
    track_search_event if Ragdoll.configuration.track_search_analytics
    
    respond_to do |format|
      format.html
      format.json { render json: search_response }
      format.turbo_stream { render :index }
    end
  end
  
  # GET /search/suggestions
  def suggestions
    @suggestions = Ragdoll::SuggestionService.new(@query).call
    
    respond_to do |format|
      format.json { render json: @suggestions }
    end
  end
  
  # GET /search/facets
  def facets
    @facets = Ragdoll::FacetService.new(@query, facet_options).call
    
    respond_to do |format|
      format.json { render json: @facets }
      format.html { render partial: 'facets', locals: { facets: @facets } }
    end
  end
  
  # POST /search/save
  def save
    @saved_search = current_user.saved_searches.build(saved_search_params)
    
    if @saved_search.save
      render json: @saved_search, status: :created
    else
      render json: @saved_search.errors, status: :unprocessable_entity
    end
  end
  
  private
  
  def set_search_params
    @query = params[:q] || params[:query]
    @filters = params[:filters] || {}
    @sort = params[:sort] || 'relevance'
    @page = params[:page] || 1
    @per_page = [params[:per_page].to_i, 50].min.positive? || 10
  end
  
  def search_options
    {
      filters: @filters,
      sort: @sort,
      page: @page,
      per_page: @per_page,
      include_facets: params[:include_facets],
      search_type: params[:search_type] || 'hybrid',
      threshold: params[:threshold]&.to_f || Ragdoll.configuration.similarity_threshold,
      user: current_user
    }
  end
  
  def facet_options
    {
      facet_fields: params[:facet_fields] || ['content_type', 'user', 'created_at'],
      facet_limit: params[:facet_limit] || 10
    }
  end
  
  def search_response
    {
      query: @query,
      results: @results.map { |result| serialize_search_result(result) },
      total_count: @results.total_count,
      page: @page,
      per_page: @per_page,
      facets: @facets,
      suggestions: @suggestions,
      search_time: @search_service.search_time
    }
  end
  
  def serialize_search_result(result)
    {
      id: result.id,
      title: result.title,
      description: result.description,
      content_preview: result.content_preview,
      similarity_score: result.similarity_score,
      url: document_path(result),
      metadata: result.metadata,
      user: result.user&.name,
      created_at: result.created_at
    }
  end
  
  def track_search_event
    Ragdoll::SearchAnalytics.track(
      query: @query,
      user: current_user,
      results_count: @results.size,
      search_type: params[:search_type],
      response_time: @search_service.search_time
    )
  end
  
  def authentication_required?
    !Ragdoll.configuration.allow_anonymous_search
  end
  
  def authorization_required?
    Ragdoll.configuration.authorize_search.present?
  end
  
  def authorize_search!
    return if Ragdoll.configuration.authorize_search.call(current_user)
    
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Search access denied.' }
      format.json { render json: { error: 'Search access denied' }, status: :forbidden }
    end
  end
  
  def saved_search_params
    params.require(:saved_search).permit(:name, :query, :filters, :is_public)
  end
end
```

### Ragdoll::Admin::AdminController

Base controller for administrative functions.

```ruby
class Ragdoll::Admin::AdminController < Ragdoll::ApplicationController
  before_action :authenticate_admin!
  layout 'ragdoll/admin'
  
  protected
  
  def authenticate_admin!
    return if current_user&.admin?
    
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Admin access required.' }
      format.json { render json: { error: 'Admin access required' }, status: :forbidden }
    end
  end
end

class Ragdoll::Admin::DashboardController < Ragdoll::Admin::AdminController
  # GET /admin/dashboard
  def index
    @stats = {
      total_documents: Ragdoll::Document.count,
      processed_documents: Ragdoll::Document.processed.count,
      pending_documents: Ragdoll::Document.pending.count,
      failed_documents: Ragdoll::Document.failed.count,
      total_storage: Ragdoll::Document.sum(:file_size),
      active_users: active_users_count,
      recent_searches: recent_searches_count,
      system_health: system_health_check
    }
    
    @recent_uploads = Ragdoll::Document.recent(7).limit(10)
    @processing_queue_size = processing_queue_size
    @error_documents = Ragdoll::Document.failed.limit(5)
  end
  
  # GET /admin/system_info
  def system_info
    @system_info = {
      ragdoll_version: Ragdoll::VERSION,
      rails_version: Rails.version,
      ruby_version: RUBY_VERSION,
      database_version: database_version,
      redis_version: redis_version,
      background_job_adapter: Ragdoll.configuration.job_adapter,
      llm_provider: Ragdoll.configuration.llm_provider,
      storage_service: Ragdoll.configuration.storage_service
    }
    
    respond_to do |format|
      format.html
      format.json { render json: @system_info }
    end
  end
  
  private
  
  def active_users_count
    # Implementation depends on your user model
    User.joins(:documents).where(documents: { created_at: 1.week.ago.. }).distinct.count
  end
  
  def recent_searches_count
    # Implementation depends on search analytics
    Ragdoll::SearchAnalytics.where(created_at: 1.day.ago..).count
  end
  
  def system_health_check
    {
      database: database_healthy?,
      llm_provider: llm_provider_healthy?,
      background_jobs: background_jobs_healthy?,
      storage: storage_healthy?
    }
  end
  
  def processing_queue_size
    case Ragdoll.configuration.job_adapter
    when :sidekiq
      Sidekiq::Queue.new(Ragdoll.configuration.processing_queue).size
    when :resque
      Resque.size(Ragdoll.configuration.processing_queue)
    else
      0
    end
  end
  
  def database_healthy?
    ActiveRecord::Base.connection.active?
  rescue
    false
  end
  
  def llm_provider_healthy?
    Ragdoll::LLMService.new.health_check
  rescue
    false
  end
  
  def background_jobs_healthy?
    case Ragdoll.configuration.job_adapter
    when :sidekiq
      Sidekiq.redis(&:ping) == 'PONG'
    else
      true
    end
  rescue
    false
  end
  
  def storage_healthy?
    ActiveStorage::Blob.service.exist?('health_check')
  rescue
    false
  end
end
```

## API Controllers

### Ragdoll::Api::V1::BaseController

Base API controller with common functionality.

```ruby
class Ragdoll::Api::V1::BaseController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods
  
  before_action :authenticate_api_user!
  before_action :set_default_format
  
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :render_unprocessable_entity
  rescue_from Ragdoll::AuthorizationError, with: :render_forbidden
  
  protected
  
  def authenticate_api_user!
    authenticate_or_request_with_http_token do |token, options|
      @current_api_user = User.find_by(api_token: token)
    end
  end
  
  def current_api_user
    @current_api_user
  end
  
  def set_default_format
    request.format = :json unless params[:format]
  end
  
  def render_success(data = nil, message = nil, status = :ok)
    response = { success: true }
    response[:message] = message if message
    response[:data] = data if data
    render json: response, status: status
  end
  
  def render_error(message, status = :bad_request, errors = nil)
    response = { success: false, message: message }
    response[:errors] = errors if errors
    render json: response, status: status
  end
  
  def render_not_found(exception)
    render_error("Record not found", :not_found)
  end
  
  def render_unprocessable_entity(exception)
    render_error("Validation failed", :unprocessable_entity, exception.record.errors)
  end
  
  def render_forbidden(exception)
    render_error("Access denied", :forbidden)
  end
end
```

### Ragdoll::Api::V1::DocumentsController

RESTful API for document operations.

```ruby
class Ragdoll::Api::V1::DocumentsController < Ragdoll::Api::V1::BaseController
  before_action :set_document, only: [:show, :update, :destroy, :reprocess]
  before_action :authorize_document!, only: [:show, :update, :destroy]
  
  # GET /api/v1/documents
  def index
    @documents = documents_scope
      .includes(:user, :contents)
      .page(params[:page])
      .per(params[:per_page] || 20)
    
    apply_filters
    
    render json: {
      documents: @documents.map { |doc| serialize_document(doc) },
      pagination: pagination_meta(@documents)
    }
  end
  
  # GET /api/v1/documents/:id
  def show
    render json: {
      document: serialize_document(@document, include_content: true)
    }
  end
  
  # POST /api/v1/documents
  def create
    @document = documents_scope.build(document_params)
    @document.user = current_api_user
    
    if @document.save
      enqueue_processing if should_auto_process?
      render_success(serialize_document(@document), "Document created successfully", :created)
    else
      render_error("Document creation failed", :unprocessable_entity, @document.errors)
    end
  end
  
  # PATCH /api/v1/documents/:id
  def update
    if @document.update(document_params)
      enqueue_reprocessing if file_changed?
      render_success(serialize_document(@document), "Document updated successfully")
    else
      render_error("Document update failed", :unprocessable_entity, @document.errors)
    end
  end
  
  # DELETE /api/v1/documents/:id
  def destroy
    @document.destroy
    render_success(nil, "Document deleted successfully")
  end
  
  # POST /api/v1/documents/:id/reprocess
  def reprocess
    Ragdoll::ProcessDocumentJob.perform_later(@document)
    render_success(nil, "Document reprocessing queued")
  end
  
  # POST /api/v1/documents/bulk_upload
  def bulk_upload
    files = params[:files] || []
    metadata = params[:metadata] || {}
    
    results = []
    errors = []
    
    files.each_with_index do |file, index|
      document = documents_scope.build(
        title: file.original_filename,
        file: file,
        metadata: metadata,
        user: current_api_user
      )
      
      if document.save
        enqueue_processing if should_auto_process?
        results << serialize_document(document)
      else
        errors << { index: index, filename: file.original_filename, errors: document.errors }
      end
    end
    
    render json: {
      success: errors.empty?,
      uploaded: results.size,
      failed: errors.size,
      documents: results,
      errors: errors
    }
  end
  
  private
  
  def documents_scope
    Ragdoll::Document.accessible_by(current_api_user)
  end
  
  def set_document
    @document = documents_scope.find(params[:id])
  end
  
  def document_params
    params.require(:document).permit(:title, :description, :file, metadata: {})
  end
  
  def apply_filters
    @documents = @documents.where(status: params[:status]) if params[:status].present?
    @documents = @documents.where(content_type: params[:content_type]) if params[:content_type].present?
    @documents = @documents.where('created_at >= ?', params[:created_after]) if params[:created_after].present?
    @documents = @documents.where('created_at <= ?', params[:created_before]) if params[:created_before].present?
  end
  
  def serialize_document(document, include_content: false)
    result = {
      id: document.id,
      title: document.title,
      description: document.description,
      content_type: document.content_type,
      file_size: document.file_size,
      status: document.status,
      metadata: document.metadata,
      created_at: document.created_at,
      updated_at: document.updated_at,
      user: {
        id: document.user&.id,
        name: document.user&.name
      },
      urls: {
        self: api_v1_document_url(document),
        download: document_url(document) + '/download'
      }
    }
    
    if include_content
      result[:content] = {
        preview: document.content_preview,
        chunks_count: document.contents.count,
        embeddings_count: document.embeddings.count
      }
    end
    
    result
  end
  
  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      total_pages: collection.total_pages,
      total_count: collection.total_count,
      per_page: collection.limit_value
    }
  end
  
  def authorize_document!
    return if Ragdoll.configuration.authorize_document_access.call(@document, current_api_user)
    
    raise Ragdoll::AuthorizationError, "Access denied to document #{@document.id}"
  end
  
  def should_auto_process?
    Ragdoll.configuration.auto_process_documents && @document.file.attached?
  end
  
  def enqueue_processing
    Ragdoll::ProcessDocumentJob.perform_later(@document)
  end
  
  def enqueue_reprocessing
    return unless @document.file.attached?
    
    @document.update!(status: 'pending')
    Ragdoll::ProcessDocumentJob.perform_later(@document)
  end
  
  def file_changed?
    @document.saved_change_to_attribute?(:file) || @document.file.attachment.changed?
  end
end
```

## Controller Customization

### Inheriting from Ragdoll Controllers

```ruby
class DocumentsController < Ragdoll::DocumentsController
  before_action :set_company_scope
  
  private
  
  def documents_scope
    current_user.company.documents
  end
  
  def set_company_scope
    @company = current_user.company
  end
  
  def document_params
    super.merge(company_id: @company.id)
  end
end
```

### Custom Authorization

```ruby
class Ragdoll::DocumentsController < Ragdoll::ApplicationController
  include Pundit::Authorization
  
  def show
    authorize @document
    # ... rest of action
  end
  
  def create
    @document = documents_scope.build(document_params)
    authorize @document
    # ... rest of action
  end
end

# app/policies/ragdoll/document_policy.rb
class Ragdoll::DocumentPolicy < ApplicationPolicy
  def show?
    user.admin? || record.user == user || record.public?
  end
  
  def create?
    user.present? && user.can_upload_documents?
  end
  
  def update?
    user.admin? || record.user == user
  end
  
  def destroy?
    user.admin? || record.user == user
  end
end
```

### Adding Custom Actions

```ruby
class Ragdoll::DocumentsController < Ragdoll::ApplicationController
  # GET /documents/:id/preview
  def preview
    @document = documents_scope.find(params[:id])
    authorize_document!
    
    @preview_content = Ragdoll::PreviewService.new(@document).generate
    
    respond_to do |format|
      format.html { render layout: false }
      format.json { render json: { preview: @preview_content } }
    end
  end
  
  # POST /documents/:id/share
  def share
    @document = documents_scope.find(params[:id])
    authorize_document!
    
    @share_link = Ragdoll::ShareLinkService.new(@document, current_user).create
    
    respond_to do |format|
      format.json { render json: { share_url: @share_link.url } }
    end
  end
  
  # POST /documents/:id/favorite
  def favorite
    @document = documents_scope.find(params[:id])
    current_user.favorite(@document)
    
    respond_to do |format|
      format.json { render json: { favorited: true } }
    end
  end
end
```

This comprehensive controller documentation provides everything you need to understand, use, and customize the Ragdoll Rails controllers for your specific needs.