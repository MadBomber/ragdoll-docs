# Views & Helpers

The ragdoll-rails gem provides a comprehensive set of view templates, components, and helper methods to quickly build document management and search interfaces in your Rails application.

## View Templates

### Document Views

#### Document Index (`documents/index.html.erb`)

```erb
<div class="ragdoll-documents-index">
  <div class="documents-header">
    <h1>Documents</h1>
    <div class="documents-actions">
      <%= link_to "Upload Document", new_document_path, class: "btn btn-primary" %>
      <%= link_to "Bulk Upload", bulk_upload_documents_path, class: "btn btn-secondary" %>
    </div>
  </div>

  <div class="documents-filters">
    <%= form_with url: documents_path, method: :get, local: true, class: "filter-form" do |form| %>
      <div class="filter-group">
        <%= form.select :status, options_for_select([
          ['All Statuses', ''],
          ['Processed', 'processed'],
          ['Processing', 'processing'],
          ['Pending', 'pending'],
          ['Failed', 'failed']
        ], params[:status]), {}, { class: "form-select" } %>
      </div>
      
      <div class="filter-group">
        <%= form.select :content_type, content_type_options, {}, { class: "form-select" } %>
      </div>
      
      <div class="filter-group">
        <%= form.submit "Filter", class: "btn btn-outline-primary" %>
      </div>
    <% end %>
  </div>

  <div class="documents-grid">
    <% @documents.each do |document| %>
      <%= render 'document_card', document: document %>
    <% end %>
  </div>

  <div class="documents-pagination">
    <%= paginate @documents %>
  </div>
</div>
```

#### Document Card (`documents/_document_card.html.erb`)

```erb
<div class="document-card" data-document-id="<%= document.id %>">
  <div class="document-thumbnail">
    <% if document.file.attached? && document.file.representable? %>
      <%= image_tag document.file.representation(resize_to_limit: [200, 150]), 
                    alt: document.title, class: "thumbnail-image" %>
    <% else %>
      <div class="file-type-icon">
        <%= ragdoll_file_icon(document.content_type) %>
      </div>
    <% end %>
  </div>
  
  <div class="document-content">
    <h3 class="document-title">
      <%= link_to document.title, document_path(document) %>
    </h3>
    
    <p class="document-description">
      <%= truncate(document.description, length: 100) %>
    </p>
    
    <div class="document-meta">
      <span class="file-size"><%= human_file_size(document.file_size) %></span>
      <span class="upload-date"><%= time_ago_in_words(document.created_at) %> ago</span>
      <span class="status-badge status-<%= document.status %>">
        <%= document.status.humanize %>
      </span>
    </div>
    
    <div class="document-actions">
      <%= link_to "View", document_path(document), class: "btn btn-sm btn-outline-primary" %>
      <% if can_edit_document?(document) %>
        <%= link_to "Edit", edit_document_path(document), class: "btn btn-sm btn-outline-secondary" %>
      <% end %>
      <% if can_delete_document?(document) %>
        <%= link_to "Delete", document_path(document), method: :delete,
                    confirm: "Are you sure?", class: "btn btn-sm btn-outline-danger" %>
      <% end %>
    </div>
  </div>
</div>
```

#### Document Show (`documents/show.html.erb`)

```erb
<div class="ragdoll-document-show">
  <div class="document-header">
    <div class="document-info">
      <h1><%= @document.title %></h1>
      <p class="document-description"><%= @document.description %></p>
      
      <div class="document-metadata">
        <div class="meta-item">
          <strong>File Type:</strong> <%= @document.content_type %>
        </div>
        <div class="meta-item">
          <strong>File Size:</strong> <%= human_file_size(@document.file_size) %>
        </div>
        <div class="meta-item">
          <strong>Status:</strong> 
          <span class="status-badge status-<%= @document.status %>">
            <%= @document.status.humanize %>
          </span>
        </div>
        <div class="meta-item">
          <strong>Uploaded:</strong> <%= @document.created_at.strftime("%B %d, %Y at %I:%M %p") %>
        </div>
        <% if @document.user %>
          <div class="meta-item">
            <strong>Uploaded by:</strong> <%= @document.user.name %>
          </div>
        <% end %>
      </div>
    </div>
    
    <div class="document-actions">
      <%= link_to "Download", document_download_path(@document), 
                  class: "btn btn-primary" %>
      <% if can_edit_document?(@document) %>
        <%= link_to "Edit", edit_document_path(@document), 
                    class: "btn btn-secondary" %>
      <% end %>
      <% if can_reprocess_document?(@document) %>
        <%= link_to "Reprocess", reprocess_document_path(@document), 
                    method: :post, class: "btn btn-outline-secondary" %>
      <% end %>
    </div>
  </div>
  
  <div class="document-content">
    <div class="content-preview">
      <h3>Content Preview</h3>
      <div class="preview-text">
        <%= simple_format(@content_preview) %>
      </div>
    </div>
    
    <% if @document.metadata.any? %>
      <div class="document-custom-metadata">
        <h3>Metadata</h3>
        <dl class="metadata-list">
          <% @document.metadata.each do |key, value| %>
            <dt><%= key.humanize %></dt>
            <dd><%= value %></dd>
          <% end %>
        </dl>
      </div>
    <% end %>
  </div>
  
  <% if @related_documents.any? %>
    <div class="related-documents">
      <h3>Related Documents</h3>
      <div class="related-grid">
        <% @related_documents.each do |doc| %>
          <%= render 'document_card', document: doc %>
        <% end %>
      </div>
    </div>
  <% end %>
</div>
```

### Search Views

#### Search Interface (`search/index.html.erb`)

```erb
<div class="ragdoll-search">
  <div class="search-header">
    <h1>Search Documents</h1>
  </div>
  
  <div class="search-form">
    <%= form_with url: search_path, method: :get, local: true, class: "search-form-wrapper" do |form| %>
      <div class="search-input-group">
        <%= form.text_field :q, value: params[:q], 
                           placeholder: "Search documents...", 
                           class: "search-input",
                           data: { 
                             action: "input->search#suggest",
                             search_target: "input"
                           } %>
        <%= form.submit "Search", class: "search-btn" %>
      </div>
      
      <div class="search-options">
        <%= form.select :search_type, [
          ['Hybrid Search', 'hybrid'],
          ['Semantic Search', 'semantic'],
          ['Keyword Search', 'keyword']
        ], { selected: params[:search_type] || 'hybrid' }, { class: "search-type-select" } %>
        
        <%= form.check_box :include_facets, checked: params[:include_facets] %>
        <%= form.label :include_facets, "Show filters" %>
      </div>
    <% end %>
  </div>
  
  <div class="search-suggestions" data-search-target="suggestions" style="display: none;">
    <!-- Populated via Stimulus -->
  </div>
  
  <div class="search-results-container">
    <% if @results.present? %>
      <div class="search-meta">
        <p>Found <%= pluralize(@results.total_count, 'result') %> 
           <% if params[:q].present? %>for "<%= params[:q] %>"<% end %>
           in <%= number_with_precision(@search_service.search_time, precision: 3) %>s
        </p>
      </div>
      
      <div class="search-content">
        <% if @facets.present? %>
          <div class="search-facets">
            <%= render 'search_facets', facets: @facets %>
          </div>
        <% end %>
        
        <div class="search-results">
          <% @results.each do |result| %>
            <%= render 'search_result', result: result %>
          <% end %>
          
          <div class="search-pagination">
            <%= paginate @results %>
          </div>
        </div>
      </div>
    <% elsif params[:q].present? %>
      <div class="no-results">
        <h3>No results found</h3>
        <p>Try adjusting your search terms or filters.</p>
        
        <% if @suggestions.present? %>
          <div class="search-suggestions">
            <p>Did you mean:</p>
            <ul>
              <% @suggestions.each do |suggestion| %>
                <li>
                  <%= link_to suggestion, search_path(q: suggestion) %>
                </li>
              <% end %>
            </ul>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
</div>
```

#### Search Result (`search/_search_result.html.erb`)

```erb
<div class="search-result" data-result-id="<%= result.id %>">
  <div class="result-header">
    <h3 class="result-title">
      <%= link_to result.title, document_path(result) %>
    </h3>
    <div class="result-meta">
      <span class="similarity-score">
        <%= number_to_percentage(result.similarity_score * 100, precision: 1) %> match
      </span>
      <span class="file-type"><%= result.content_type %></span>
      <span class="upload-date"><%= result.created_at.strftime("%b %d, %Y") %></span>
    </div>
  </div>
  
  <div class="result-content">
    <p class="result-description"><%= result.description %></p>
    <div class="result-preview">
      <%= highlight(result.content_preview, params[:q]) %>
    </div>
  </div>
  
  <div class="result-actions">
    <%= link_to "View Document", document_path(result), class: "btn btn-sm btn-primary" %>
    <%= link_to "Download", document_download_path(result), class: "btn btn-sm btn-outline-secondary" %>
  </div>
</div>
```

## View Helpers

### Ragdoll::ApplicationHelper

```ruby
module Ragdoll::ApplicationHelper
  # File type and size helpers
  def ragdoll_file_icon(content_type)
    icon_class = case content_type
                when /^image\// then "bi-file-earmark-image"
                when /pdf/ then "bi-file-earmark-pdf"
                when /word|doc/ then "bi-file-earmark-word"
                when /excel|sheet/ then "bi-file-earmark-excel"
                when /powerpoint|presentation/ then "bi-file-earmark-ppt"
                when /^text\// then "bi-file-earmark-text"
                when /^audio\// then "bi-file-earmark-music"
                when /^video\// then "bi-file-earmark-play"
                else "bi-file-earmark"
                end
    
    content_tag :i, "", class: "#{icon_class} file-icon"
  end
  
  def human_file_size(size)
    return "0 bytes" if size.nil? || size.zero?
    
    number_to_human_size(size, precision: 1)
  end
  
  def content_type_options
    types = Ragdoll::Document.distinct.pluck(:content_type).compact.sort
    options = [['All Types', '']]
    
    types.each do |type|
      display_name = case type
                    when /^image\// then "Images"
                    when /pdf/ then "PDFs"  
                    when /word|doc/ then "Word Documents"
                    when /excel|sheet/ then "Spreadsheets"
                    when /powerpoint|presentation/ then "Presentations"
                    when /^text\// then "Text Files"
                    else type.humanize
                    end
      options << [display_name, type]
    end
    
    options_for_select(options.uniq, params[:content_type])
  end
  
  # Status helpers
  def status_badge(status)
    css_class = case status
               when 'processed' then 'badge-success'
               when 'processing' then 'badge-warning'
               when 'pending' then 'badge-secondary'
               when 'failed' then 'badge-danger'
               else 'badge-light'
               end
    
    content_tag :span, status.humanize, class: "badge #{css_class}"
  end
  
  def processing_progress(document)
    return unless document.processing?
    
    progress = document.processing_progress || 0
    
    content_tag :div, class: "progress" do
      content_tag :div, "", 
                  class: "progress-bar", 
                  style: "width: #{progress}%",
                  data: { 
                    document_id: document.id,
                    progress: progress 
                  }
    end
  end
  
  # Authorization helpers
  def can_edit_document?(document)
    return false unless current_user
    
    Ragdoll.configuration.authorize_document_access&.call(document, current_user) ||
      document.user == current_user ||
      current_user.admin?
  end
  
  def can_delete_document?(document)
    can_edit_document?(document)
  end
  
  def can_reprocess_document?(document)
    can_edit_document?(document) && document.processed?
  end
  
  # Search helpers
  def highlight_search_terms(text, query)
    return text if query.blank?
    
    terms = query.split(/\s+/).reject(&:blank?)
    highlighted = text
    
    terms.each do |term|
      highlighted = highlighted.gsub(
        /#{Regexp.escape(term)}/i,
        '<mark>\0</mark>'
      )
    end
    
    highlighted.html_safe
  end
  
  def search_result_snippet(content, query, length: 200)
    return truncate(content, length: length) if query.blank?
    
    # Find the best snippet containing search terms
    terms = query.split(/\s+/).reject(&:blank?)
    term_positions = []
    
    terms.each do |term|
      pos = content.downcase.index(term.downcase)
      term_positions << pos if pos
    end
    
    if term_positions.any?
      start_pos = [term_positions.min - 50, 0].max
      snippet = content[start_pos, length]
      "..." + snippet + "..."
    else
      truncate(content, length: length)
    end
  end
  
  # Metadata helpers
  def format_metadata_value(value)
    case value
    when Date, DateTime, Time
      value.strftime("%B %d, %Y")
    when TrueClass, FalseClass
      value ? "Yes" : "No"
    when Array
      value.join(", ")
    when Hash
      value.to_json
    else
      value.to_s
    end
  end
  
  def metadata_icon(key)
    icon_class = case key.to_s.downcase
                when /author|creator|owner/ then "bi-person"
                when /date|time|created|updated/ then "bi-calendar"
                when /category|tag|type/ then "bi-tag"
                when /size|length|count/ then "bi-rulers"
                when /location|place|geo/ then "bi-geo-alt"
                when /url|link|website/ then "bi-link"
                when /email|mail/ then "bi-envelope"
                when /phone|tel/ then "bi-telephone"
                else "bi-info-circle"
                end
    
    content_tag :i, "", class: "#{icon_class} metadata-icon"
  end
end
```

### Ragdoll::SearchHelper

```ruby
module Ragdoll::SearchHelper
  def search_form(options = {})
    defaults = {
      url: search_path,
      method: :get,
      local: true,
      class: "ragdoll-search-form"
    }
    
    form_with **defaults.merge(options) do |form|
      render 'ragdoll/search/search_form', form: form
    end
  end
  
  def search_filters(facets)
    return unless facets.present?
    
    content_tag :div, class: "search-filters" do
      facets.map do |facet_name, facet_data|
        render 'ragdoll/search/filter_group', 
               facet_name: facet_name, 
               facet_data: facet_data
      end.join.html_safe
    end
  end
  
  def similarity_score_badge(score)
    percentage = (score * 100).round(1)
    
    css_class = case percentage
               when 90..100 then 'badge-success'
               when 70..89 then 'badge-warning'
               when 50..69 then 'badge-secondary'
               else 'badge-light'
               end
    
    content_tag :span, "#{percentage}%", class: "badge #{css_class}"
  end
  
  def search_time_display(time_in_seconds)
    if time_in_seconds < 1
      "#{(time_in_seconds * 1000).round}ms"
    else
      "#{time_in_seconds.round(2)}s"
    end
  end
  
  def search_suggestions(suggestions)
    return unless suggestions.present?
    
    content_tag :div, class: "search-suggestions" do
      content_tag :p, "Did you mean:" do
        suggestions.map do |suggestion|
          link_to suggestion, search_path(q: suggestion), class: "suggestion-link"
        end.join(" · ").html_safe
      end
    end
  end
  
  def facet_filter_link(facet_name, facet_value, count)
    current_filters = params[:filters] || {}
    new_filters = current_filters.dup
    
    if new_filters[facet_name] == facet_value
      new_filters.delete(facet_name)
      css_class = "facet-link active"
    else
      new_filters[facet_name] = facet_value
      css_class = "facet-link"
    end
    
    link_to search_path(q: params[:q], filters: new_filters), class: css_class do
      "#{facet_value} (#{count})"
    end
  end
end
```

## Stimulus Controllers

### Search Controller (`search_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "suggestions", "results"]
  static values = { 
    suggestionsUrl: String,
    debounceDelay: { type: Number, default: 300 }
  }
  
  connect() {
    this.timeout = null
  }
  
  suggest() {
    clearTimeout(this.timeout)
    
    this.timeout = setTimeout(() => {
      const query = this.inputTarget.value.trim()
      
      if (query.length > 2) {
        this.fetchSuggestions(query)
      } else {
        this.hideSuggestions()
      }
    }, this.debounceDelayValue)
  }
  
  async fetchSuggestions(query) {
    try {
      const response = await fetch(`${this.suggestionsUrlValue}?q=${encodeURIComponent(query)}`)
      const suggestions = await response.json()
      
      this.displaySuggestions(suggestions)
    } catch (error) {
      console.error('Failed to fetch suggestions:', error)
    }
  }
  
  displaySuggestions(suggestions) {
    if (suggestions.length === 0) {
      this.hideSuggestions()
      return
    }
    
    const suggestionsList = suggestions.map(suggestion => 
      `<li class="suggestion-item" data-action="click->search#selectSuggestion">${suggestion}</li>`
    ).join('')
    
    this.suggestionsTarget.innerHTML = `<ul class="suggestions-list">${suggestionsList}</ul>`
    this.suggestionsTarget.style.display = 'block'
  }
  
  hideSuggestions() {
    this.suggestionsTarget.style.display = 'none'
  }
  
  selectSuggestion(event) {
    const suggestion = event.target.textContent
    this.inputTarget.value = suggestion
    this.hideSuggestions()
    this.submitSearch()
  }
  
  submitSearch() {
    this.element.querySelector('form').submit()
  }
  
  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

### Upload Controller (`upload_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "preview", "progress", "errors"]
  static values = { 
    maxFileSize: Number,
    allowedTypes: Array,
    uploadUrl: String
  }
  
  preview() {
    const files = this.inputTarget.files
    this.clearErrors()
    
    if (files.length === 0) {
      this.clearPreview()
      return
    }
    
    this.validateFiles(files)
    this.displayPreview(files)
  }
  
  validateFiles(files) {
    let hasErrors = false
    
    Array.from(files).forEach(file => {
      if (file.size > this.maxFileSizeValue) {
        this.showError(`File "${file.name}" is too large. Maximum size is ${this.formatFileSize(this.maxFileSizeValue)}.`)
        hasErrors = true
      }
      
      if (!this.allowedTypesValue.includes(file.type)) {
        this.showError(`File type "${file.type}" is not allowed for "${file.name}".`)
        hasErrors = true
      }
    })
    
    return !hasErrors
  }
  
  displayPreview(files) {
    const previews = Array.from(files).map(file => {
      return `
        <div class="file-preview" data-filename="${file.name}">
          <div class="file-info">
            <span class="filename">${file.name}</span>
            <span class="filesize">${this.formatFileSize(file.size)}</span>
          </div>
          <div class="file-actions">
            <button type="button" class="remove-file" data-action="click->upload#removeFile">×</button>
          </div>
        </div>
      `
    }).join('')
    
    this.previewTarget.innerHTML = previews
  }
  
  removeFile(event) {
    const preview = event.target.closest('.file-preview')
    const filename = preview.dataset.filename
    
    // Remove from file input (requires recreating the input)
    const dt = new DataTransfer()
    const files = this.inputTarget.files
    
    Array.from(files).forEach(file => {
      if (file.name !== filename) {
        dt.items.add(file)
      }
    })
    
    this.inputTarget.files = dt.files
    preview.remove()
    
    if (this.inputTarget.files.length === 0) {
      this.clearPreview()
    }
  }
  
  async upload(event) {
    event.preventDefault()
    
    const files = this.inputTarget.files
    if (files.length === 0) return
    
    if (!this.validateFiles(files)) return
    
    this.showProgress()
    
    try {
      for (let i = 0; i < files.length; i++) {
        await this.uploadFile(files[i], i + 1, files.length)
      }
      
      this.onUploadSuccess()
    } catch (error) {
      this.showError('Upload failed: ' + error.message)
    } finally {
      this.hideProgress()
    }
  }
  
  async uploadFile(file, index, total) {
    const formData = new FormData()
    formData.append('document[file]', file)
    formData.append('document[title]', file.name)
    
    const response = await fetch(this.uploadUrlValue, {
      method: 'POST',
      body: formData,
      headers: {
        'X-CSRF-Token': document.querySelector('[name="csrf-token"]').content
      }
    })
    
    if (!response.ok) {
      throw new Error(`Failed to upload ${file.name}`)
    }
    
    this.updateProgress(index, total)
  }
  
  showProgress() {
    this.progressTarget.style.display = 'block'
    this.updateProgress(0, 1)
  }
  
  updateProgress(current, total) {
    const percentage = (current / total) * 100
    const progressBar = this.progressTarget.querySelector('.progress-bar')
    if (progressBar) {
      progressBar.style.width = `${percentage}%`
      progressBar.textContent = `${current} / ${total} files`
    }
  }
  
  hideProgress() {
    this.progressTarget.style.display = 'none'
  }
  
  onUploadSuccess() {
    this.clearPreview()
    this.inputTarget.value = ''
    // Optionally redirect or reload
    window.location.reload()
  }
  
  showError(message) {
    const errorDiv = document.createElement('div')
    errorDiv.className = 'alert alert-danger'
    errorDiv.textContent = message
    this.errorsTarget.appendChild(errorDiv)
  }
  
  clearErrors() {
    this.errorsTarget.innerHTML = ''
  }
  
  clearPreview() {
    this.previewTarget.innerHTML = ''
  }
  
  formatFileSize(bytes) {
    if (bytes === 0) return '0 Bytes'
    
    const k = 1024
    const sizes = ['Bytes', 'KB', 'MB', 'GB']
    const i = Math.floor(Math.log(bytes) / Math.log(k))
    
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
  }
}
```

## View Components

### Document Card Component

```ruby
# app/components/ragdoll/document_card_component.rb
class Ragdoll::DocumentCardComponent < ViewComponent::Base
  def initialize(document:, show_actions: true, size: :medium)
    @document = document
    @show_actions = show_actions
    @size = size
  end
  
  private
  
  attr_reader :document, :show_actions, :size
  
  def card_classes
    base_classes = ["ragdoll-document-card"]
    base_classes << "card-#{size}"
    base_classes << "card-#{document.status}"
    base_classes.join(" ")
  end
  
  def thumbnail_url
    if document.file.attached? && document.file.representable?
      rails_representation_url(document.file.representation(resize_to_limit: thumbnail_size))
    else
      nil
    end
  end
  
  def thumbnail_size
    case size
    when :small then [100, 75]
    when :large then [400, 300]
    else [200, 150]
    end
  end
end
```

```erb
<!-- app/components/ragdoll/document_card_component.html.erb -->
<div class="<%= card_classes %>" data-document-id="<%= document.id %>">
  <div class="card-thumbnail">
    <% if thumbnail_url %>
      <%= image_tag thumbnail_url, alt: document.title, class: "thumbnail-image" %>
    <% else %>
      <div class="file-type-icon">
        <%= ragdoll_file_icon(document.content_type) %>
      </div>
    <% end %>
    
    <div class="card-overlay">
      <%= status_badge(document.status) %>
    </div>
  </div>
  
  <div class="card-content">
    <h3 class="card-title">
      <%= link_to document.title, document_path(document) %>
    </h3>
    
    <% if document.description.present? %>
      <p class="card-description">
        <%= truncate(document.description, length: description_length) %>
      </p>
    <% end %>
    
    <div class="card-meta">
      <span class="file-size"><%= human_file_size(document.file_size) %></span>
      <span class="upload-date"><%= time_ago_in_words(document.created_at) %> ago</span>
    </div>
    
    <% if show_actions %>
      <div class="card-actions">
        <%= link_to "View", document_path(document), class: "btn btn-sm btn-primary" %>
        <% if can_edit_document?(document) %>
          <%= link_to "Edit", edit_document_path(document), class: "btn btn-sm btn-secondary" %>
        <% end %>
      </div>
    <% end %>
  </div>
</div>
```

This comprehensive view and helper documentation provides everything you need to customize and extend the Ragdoll Rails UI components for your specific application needs.