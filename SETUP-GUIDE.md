# Azure AI RAG Application - Complete Setup Guide

This guide shows you how to build a **Retrieval Augmented Generation (RAG)** application using Azure AI services and CLI commands.

## 📚 What is RAG?

**RAG (Retrieval Augmented Generation)** combines:
1. **Retrieval**: Search through your data to find relevant information
2. **Augmentation**: Add that information to the AI's context
3. **Generation**: AI generates an answer based on YOUR data

**Why use RAG?**
- AI can answer questions about YOUR specific data
- More accurate than relying on AI's general knowledge
- Data stays in your control

## 🏗️ Architecture Overview

```
User Query → Vector Embedding → Azure AI Search → Relevant Docs → GPT-4o → Answer
```

**Key Components:**
1. **Azure OpenAI** - GPT-4o for chat, text-embedding-3-small for vectors
2. **Azure AI Search** - Stores documents with vector embeddings
3. **Python App** - Orchestrates the RAG pattern

---

## 🚀 Setup Steps

### Step 1: Login to Azure

```bash
az login
az account set --subscription "YOUR_SUBSCRIPTION_NAME"
```

**What this does:** Authenticates you and sets your working subscription.

---

### Step 2: Deploy Required Models

#### Deploy GPT-4o (Chat Model)
```bash
az cognitiveservices account deployment create \
  --name YOUR_RESOURCE_NAME \
  --resource-group azuraifw \
  --deployment-name gpt-4o \
  --model-name gpt-4o \
  --model-version "2024-08-06" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name "GlobalStandard"
```

**What this does:** Deploys GPT-4o for generating intelligent responses.

#### Verify Embedding Model Deployment
```bash
az cognitiveservices account deployment list \
  --name YOUR_RESOURCE_NAME \
  --resource-group azuraifw \
  --query "[].{name:name, model:properties.model.name}" \
  -o table
```

**What this does:** Lists your deployments. You should see `text-embedding-3-small`.

**💡 Why embeddings?** They convert text into numbers (vectors) that capture semantic meaning, enabling semantic search.

---

### Step 3: Prepare Your Data

#### Download Sample Data
```bash
curl -L -o brochures.zip https://github.com/MicrosoftLearning/mslearn-ai-studio/raw/main/data/brochures.zip
powershell -Command "Expand-Archive -Path brochures.zip -DestinationPath brochures -Force"
```

#### Upload to Azure Storage
```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name YOUR_STORAGE_ACCOUNT \
  --resource-group azuraifw \
  --query "[0].value" -o tsv)

# Upload all PDFs
az storage blob upload-batch \
  --destination brochures \
  --source brochures \
  --account-name YOUR_STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY" \
  --overwrite
```

**What this does:** Stores your PDF brochures in Azure Blob Storage.

---

### Step 4: Create Search Index with Vector Support

#### Get Azure AI Search Admin Key
```bash
SEARCH_KEY=$(az search admin-key show \
  --service-name YOUR_SEARCH_SERVICE \
  --resource-group azuraifw \
  --query "primaryKey" -o tsv)
```

#### Create Vector Index
```bash
curl -X POST "https://YOUR_SEARCH_SERVICE.search.windows.net/indexes?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: $SEARCH_KEY" \
  -d '{
    "name": "brochures-index",
    "fields": [
      {"name": "id", "type": "Edm.String", "key": true, "searchable": false},
      {"name": "content", "type": "Edm.String", "searchable": true},
      {"name": "title", "type": "Edm.String", "searchable": true},
      {"name": "contentVector", "type": "Collection(Edm.Single)", "searchable": true, "dimensions": 1536, "vectorSearchProfile": "vector-profile"}
    ],
    "vectorSearch": {
      "algorithms": [{"name": "vector-algorithm", "kind": "hnsw"}],
      "profiles": [{"name": "vector-profile", "algorithm": "vector-algorithm"}]
    }
  }'
```

**What this does:** Creates a search index that supports:
- **Full-text search** (content, title fields)
- **Vector search** (contentVector field with 1536 dimensions for text-embedding-3-small)
- **HNSW algorithm** for fast approximate nearest neighbor search

**💡 Key Concepts:**
- **Collection(Edm.Single)**: Array of floating-point numbers (the embedding vector)
- **dimensions: 1536**: Size of text-embedding-3-small embeddings
- **HNSW**: Hierarchical Navigable Small World - fast vector search algorithm

---

### Step 5: Generate Embeddings and Upload Documents

This Python script generates embeddings for each document and uploads them:

```python
from openai import AzureOpenAI
import requests

# Initialize Azure OpenAI client
client = AzureOpenAI(
    api_version="2024-08-01-preview",
    azure_endpoint="https://YOUR_RESOURCE_NAME.cognitiveservices.azure.com/",
    api_key="YOUR_KEY"
)

# Your documents
documents = [
    {
        "id": "1",
        "title": "New York Travel Guide",
        "content": "New York City offers amazing attractions..."
    }
]

# Generate embeddings and upload
for doc in documents:
    # Get embedding from Azure OpenAI
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=doc["content"]
    )
    embedding = response.data[0].embedding  # 1536-dimensional vector

    # Prepare document with vector
    doc["contentVector"] = embedding

    # Upload to Azure Search (see upload-with-embeddings.py for complete code)
```

**What this does:**
1. Takes text content
2. Converts it to a 1536-dimensional vector using Azure OpenAI
3. Stores both text and vector in Azure AI Search

**💡 Why store both?**
- **Text**: For displaying results to users
- **Vector**: For semantic search (finding similar meanings)

---

## 🎯 How the RAG App Works

### Configuration (.env file)
```env
OPEN_AI_ENDPOINT=https://YOUR_RESOURCE_NAME.cognitiveservices.azure.com/
OPEN_AI_KEY=your_key_here
CHAT_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small
SEARCH_ENDPOINT=https://YOUR_SEARCH_SERVICE.search.windows.net
SEARCH_KEY=your_search_key_here
INDEX_NAME=brochures-index
```

### The RAG Pattern in Code

```python
# 1. User asks a question
user_query = "Where can I stay in New York?"

# 2. RAG parameters tell Azure OpenAI to search your index
rag_params = {
    "data_sources": [{
        "type": "azure_search",
        "parameters": {
            "endpoint": search_url,
            "index_name": "brochures-index",
            "query_type": "vector",  # Use semantic search!
            "embedding_dependency": {
                "deployment_name": "text-embedding-3-small"
            }
        }
    }]
}

# 3. Azure OpenAI:
#    a. Converts query to vector
#    b. Searches your index
#    c. Finds relevant documents
#    d. Uses them to generate answer
response = chat_client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_query}],
    extra_body=rag_params
)
```

**What happens behind the scenes:**
1. Your query → Embedding model → Vector (1536 numbers)
2. Vector search in Azure AI Search → Finds similar document vectors
3. Retrieved documents → Added to GPT-4o context
4. GPT-4o → Generates answer based on YOUR data

---

## 🔍 Vector Search vs Keyword Search

### Keyword Search (Simple)
```
Query: "Where to stay?"
Searches for: exact words "where", "to", "stay"
```

### Vector Search (Semantic)
```
Query: "Where to stay?"
Converts to: [0.123, -0.456, 0.789, ...]  (1536 numbers)
Finds: Documents about "hotels", "accommodation", "lodging"
         (even without exact word match!)
```

**Vector search is smarter** - it understands meaning, not just keywords!

---

## 📊 Useful Commands

### Check Index Status
```bash
curl -X GET "https://YOUR_SEARCH_SERVICE.search.windows.net/indexes/brochures-index/stats?api-version=2024-07-01" \
  -H "api-key: $SEARCH_KEY"
```

### List All Indexes
```bash
curl -X GET "https://YOUR_SEARCH_SERVICE.search.windows.net/indexes?api-version=2024-07-01" \
  -H "api-key: $SEARCH_KEY"
```

### Check Deployments
```bash
az cognitiveservices account deployment list \
  --name YOUR_RESOURCE_NAME \
  --resource-group azuraifw \
  --query "[].{Name:name, Model:properties.model.name, Status:properties.provisioningState}" \
  -o table
```

---

## 🎓 Key Learning Points

### 1. **Embeddings are Key**
- Text → Numbers (vectors) that capture meaning
- Similar meanings → Similar vectors
- Enables semantic search

### 2. **RAG Solves Data Freshness**
- AI models have knowledge cutoff dates
- RAG lets AI use YOUR latest data
- No need to retrain models

### 3. **Azure AI Search Powers It**
- Stores documents with vectors
- Fast vector similarity search
- Combines with Azure OpenAI seamlessly

### 4. **API Versions Matter**
- Use `2024-08-01-preview` or later for RAG features
- Different models need different parameters
- Always check documentation for compatibility

### 5. **Embedding Model Dimensions**
- text-embedding-3-small: **1536 dimensions**
- text-embedding-3-large: **3072 dimensions**
- Index dimensions must match model!

---

## 🚨 Common Issues & Solutions

### Issue: "DeploymentNotFound"
**Solution:** Check deployment name matches exactly:
```bash
az cognitiveservices account deployment list --name YOUR_RESOURCE_NAME --resource-group azuraifw
```

### Issue: "Collection(Edm.Double) vs Collection(Edm.Single)"
**Solution:** Always use **Edm.Single** for vector fields in Azure AI Search.

### Issue: Vector search not working
**Solution:** Ensure:
1. Index has vector field with correct dimensions
2. Documents have embeddings uploaded
3. `query_type` is set to `"vector"`
4. Embedding model deployment name is correct

---

## 🎉 Success Criteria

Your RAG app is working when:
1. ✅ You can ask questions about your data
2. ✅ Answers include information from your documents
3. ✅ Semantic search works (finds related concepts, not just keywords)
4. ✅ Responses cite sources from your index

---

## 📚 Further Learning

- **Azure AI Search Documentation**: https://learn.microsoft.com/azure/search/
- **Azure OpenAI RAG Pattern**: https://learn.microsoft.com/azure/ai-services/openai/concepts/use-your-data
- **Vector Search Concepts**: https://learn.microsoft.com/azure/search/vector-search-overview
- **Embedding Models**: https://platform.openai.com/docs/guides/embeddings

---

**Built with:** Azure CLI, Azure OpenAI, Azure AI Search, Python
**Pattern:** Retrieval Augmented Generation (RAG)
**Date:** November 2025
