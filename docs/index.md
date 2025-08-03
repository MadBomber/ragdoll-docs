---
hide:
  - toc
---

<div class="hero-banner">
  <h1>🎪 Ragdoll Documentation</h1>
  <p>Advanced multi-modal document intelligence platform built on ActiveRecord with PostgreSQL + pgvector</p>
  <div class="quick-start-buttons">
    <a href="getting-started/quick-start/" class="btn-primary">
      🚀 Quick Start
    </a>
    <a href="getting-started/installation/" class="btn-secondary">
      📦 Installation
    </a>
    <a href="development/architecture/" class="btn-secondary">
      🏗️ Architecture
    </a>
  </div>
</div>

<div class="feature-grid">
  <div class="feature-card">
    <h3><span class="feature-icon">🎯</span> Multi-Modal First</h3>
    <p>Designed from the ground up to handle text, image, and audio content as first-class citizens through a sophisticated polymorphic architecture.</p>
  </div>

  <div class="feature-card">
    <h3><span class="feature-icon">🏗️</span> Production Ready</h3>
    <p>Enterprise-grade features including 7 LLM providers, PostgreSQL + pgvector, background processing, and comprehensive error handling.</p>
  </div>

  <div class="feature-card">
    <h3><span class="feature-icon">⚡</span> Performance Optimized</h3>
    <p>Hardware-accelerated vector operations, intelligent indexing, and connection pooling for scalable deployments.</p>
  </div>

  <div class="feature-card">
    <h3><span class="feature-icon">📊</span> Advanced Analytics</h3>
    <p>Sophisticated ranking algorithms, usage tracking, and performance monitoring with smart search result optimization.</p>
  </div>

  <div class="feature-card">
    <h3><span class="feature-icon">🔧</span> Extensible</h3>
    <p>Clear patterns for adding new content types, document processors, embedding providers, and search algorithms.</p>
  </div>

  <div class="feature-card">
    <h3><span class="feature-icon">🛡️</span> Secure</h3>
    <p>Production security best practices, API key management, file validation, and comprehensive audit logging.</p>
  </div>
</div>

## 📚 Documentation Overview

### Getting Started

- **[Quick Start Guide](getting-started/quick-start.md)** - Get up and running with Ragdoll in minutes
- **[Installation & Setup](getting-started/installation.md)** - Complete installation and environment setup
- **[Configuration Guide](getting-started/configuration.md)** - Comprehensive configuration system documentation

### Core Architecture

- **[Architecture Overview](development/architecture.md)** - System design and component relationships
- **[Multi-Modal Support](user-guide/multi-modal.md)** - Text, image, and audio content handling
- **[Database Schema](operations/database-schema.md)** - Polymorphic multi-modal database design
- **[Background Processing](operations/background-processing.md)** - ActiveJob integration and async operations

### Features & Capabilities

- **[Document Processing](user-guide/document-processing.md)** - File parsing, metadata extraction, and content analysis
- **[Search & Analytics](user-guide/search-analytics.md)** - Advanced semantic search with usage analytics
- **[Embedding System](user-guide/embedding-system.md)** - Vector generation and similarity search
- **[File Upload System](user-guide/file-uploads.md)** - Shrine-based production file handling

### API Documentation

- **[Client API Reference](api-reference/api-client.md)** - High-level client interface methods
- **[Models Reference](api-reference/api-models.md)** - ActiveRecord models and relationships
- **[Services Reference](api-reference/api-services.md)** - Business logic and processing services
- **[Jobs Reference](api-reference/api-jobs.md)** - Background job system

### Deployment & Operations

- **[Production Deployment](deployment/deployment.md)** - Production setup with PostgreSQL + pgvector
- **[Performance Tuning](deployment/performance.md)** - Optimization strategies and monitoring
- **[Monitoring & Analytics](operations/monitoring.md)** - Usage tracking and system health
- **[Troubleshooting](operations/troubleshooting.md)** - Common issues and solutions

### Advanced Topics

- **[LLM Integration](reference/llm-integration.md)** - Multiple provider support and configuration
- **[Metadata Schemas](reference/metadata-schemas.md)** - Structured content analysis and validation
- **[Extending the System](development/extending.md)** - Adding new content types and processors
- **[Security Considerations](deployment/security.md)** - Production security best practices

### Development

- **[Development Setup](development/development.md)** - Setting up development environment
- **[Testing Guide](development/testing.md)** - Running tests and coverage analysis
- **[Contributing](development/contributing.md)** - Guidelines for contributing to the project

## 🚀 What Makes Ragdoll Special

Ragdoll is not just a "simple RAG library" - it's a **production-ready document intelligence platform** with enterprise-grade features:

### 🎯 **Multi-Modal First**
Unlike most RAG systems that retrofit multi-modal support, Ragdoll was designed from the ground up to handle text, image, and audio content as first-class citizens through a sophisticated polymorphic architecture.

### 🏗️ **Sophisticated Architecture**

- **Dual Metadata Design**: Separates LLM-generated content analysis from system file properties
- **Polymorphic Database Schema**: Unified search across all content types
- **Background Processing**: Complete ActiveJob integration for scalable operations
- **Production File Handling**: Shrine-based upload system with validation

### 📊 **Advanced Analytics**

- **Usage Tracking**: Sophisticated ranking algorithms based on frequency and recency
- **Performance Monitoring**: Built-in analytics for search patterns and system health
- **Smart Ranking**: Combines similarity scores with usage analytics for better results

### 🔧 **Enterprise Features**

- **7 LLM Providers**: OpenAI, Anthropic, Google, Azure, Ollama, HuggingFace, OpenRouter
- **Production Database Support**: PostgreSQL + pgvector
- **Comprehensive Error Handling**: Custom exception hierarchy with detailed logging
- **Health Monitoring**: System diagnostics and status reporting

### ⚡ **Performance Optimized**

- **pgvector Integration**: Hardware-accelerated vector operations
- **Intelligent Indexing**: Optimized database indexes for fast search
- **Background Processing**: Non-blocking document processing
- **Connection Pooling**: Scalable database connections

## 📖 Documentation Philosophy

This documentation is **implementation-driven** - every feature documented here is fully implemented and tested. We believe in accurate documentation that matches the actual capabilities of the system.

### What You'll Find Here:

- ✅ **Accurate Examples**: All code examples are tested and working
- ✅ **Production-Ready Guidance**: Real-world deployment and optimization advice
- ✅ **Complete Feature Coverage**: Documentation for all implemented features
- ✅ **Advanced Use Cases**: Enterprise scenarios and complex integrations

### What You Won't Find:

- ❌ **Vapor Features**: We don't document features that don't exist
- ❌ **Oversimplified Examples**: Our examples reflect real-world complexity
- ❌ **Marketing Fluff**: Technical accuracy over marketing copy

## 🤝 Getting Help

### Documentation Issues
If you find any discrepancies between the documentation and actual implementation, please file an issue. We maintain strict accuracy standards.

### Feature Requests
Ragdoll has many undocumented capabilities. Before requesting a feature, check if it already exists by reviewing the complete documentation.

### Support Channels

- **GitHub Issues**: Bug reports and feature requests
- **Documentation**: Comprehensive guides and references
- **Code Examples**: Working examples for all major features

## 🎯 Quick Navigation

**New to Ragdoll?** Start with:

1. [Quick Start Guide](getting-started/quick-start.md) - Basic usage in 5 minutes
2. [Architecture Overview](development/architecture.md) - Understand the system design
3. [Multi-Modal Support](user-guide/multi-modal.md) - See what makes us different

**Ready for Production?** Focus on:

1. [Production Deployment](deployment/deployment.md) - PostgreSQL setup
2. [Configuration Guide](getting-started/configuration.md) - Enterprise configuration
3. [Performance Tuning](deployment/performance.md) - Optimization strategies

**Integrating with Existing Systems?** Review:

1. [API Reference](api-reference/api-client.md) - Client interface methods
2. [LLM Integration](reference/llm-integration.md) - Provider configuration
3. [Security Considerations](deployment/security.md) - Production security

---

*This documentation is intended to reflect the actual implementation of Ragdoll v{{ extra.version }} and should be updated with each release to maintain accuracy.*
