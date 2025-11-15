# Azure RAG Chat Application

A Retrieval Augmented Generation (RAG) chat application built with Azure OpenAI and Azure AI Search.

## Features

- **Vector Search**: Semantic search using embeddings for intelligent document retrieval
- **RAG Pattern**: Combines retrieval with GPT-4o for accurate, data-grounded responses
- **Azure Integration**: Fully integrated with Azure OpenAI and Azure AI Search

## Quick Start

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/azure-rag-chat-app.git
   cd azure-rag-chat-app
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Configure environment**
   ```bash
   cp .env.example .env
   # Edit .env with your Azure credentials
   ```

4. **Run the application**
   ```bash
   python rag-app.py
   ```

## Setup Guide

See [SETUP-GUIDE.md](SETUP-GUIDE.md) for complete step-by-step instructions on:
- Setting up Azure resources
- Deploying models
- Creating vector search indexes
- Understanding the RAG pattern

## Architecture

```
User Query → Embedding Model → Azure AI Search → Relevant Docs → GPT-4o → Response
```

## Requirements

- Python 3.8+
- Azure subscription
- Azure OpenAI resource
- Azure AI Search service

## Learn More

- [Azure OpenAI Documentation](https://learn.microsoft.com/azure/ai-services/openai/)
- [Azure AI Search Documentation](https://learn.microsoft.com/azure/search/)
- [RAG Pattern Overview](https://learn.microsoft.com/azure/ai-services/openai/concepts/use-your-data)

## License

MIT
