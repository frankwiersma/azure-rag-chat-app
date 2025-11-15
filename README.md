# Azure RAG Chat Application 🤖

A production-ready **Retrieval Augmented Generation (RAG)** chat application built with Azure OpenAI and Azure AI Search, featuring semantic vector search for intelligent document retrieval.

![Python](https://img.shields.io/badge/python-3.8+-blue.svg)
![Azure](https://img.shields.io/badge/Azure-OpenAI-0078D4.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)

---

## 📚 What is RAG?

**RAG (Retrieval Augmented Generation)** is a pattern that makes AI smarter by combining:

1. **Retrieval** - Search through YOUR data to find relevant information
2. **Augmentation** - Add that context to the AI's prompt
3. **Generation** - AI generates accurate answers based on YOUR data

### Why Use RAG?

✅ **Accurate** - AI answers from YOUR specific data, not general knowledge
✅ **Current** - Always uses latest data without retraining models
✅ **Secure** - Your data stays in your control
✅ **Traceable** - Responses cite source documents

---

## 🏗️ Architecture

```
┌─────────────┐
│ User Query  │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│ Embedding Model  │ Converts query to 1536-dimensional vector
└────────┬─────────┘
         │
         ▼
┌─────────────────────┐
│ Azure AI Search     │ Vector similarity search
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Retrieved Documents │
│ + Original Query    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ GPT-4o              │ Generates answer using retrieved context
└────────┬────────────┘
         │
         ▼
    ┌────────┐
    │ Answer │
    └────────┘
```

### Key Components

| Component | Purpose | Model/Service |
|-----------|---------|---------------|
| **Chat Model** | Generate intelligent responses | GPT-4o |
| **Embedding Model** | Convert text to vectors | text-embedding-3-small (1536 dims) |
| **Vector Store** | Store & search document embeddings | Azure AI Search |
| **Orchestrator** | RAG pattern implementation | Python App |

---

## 🔍 Vector Search vs Keyword Search

### Traditional Keyword Search
```
Query: "Where to stay?"
Matches: Documents containing exact words "where", "stay"
```

### Vector Search (What This App Uses!)
```
Query: "Where to stay?"
   ↓ (embedding)
Vector: [0.123, -0.456, 0.789, ... ] (1536 numbers)
   ↓ (similarity search)
Finds: Documents about "hotels", "accommodation", "lodging"
       (Even without exact keywords!)
```

**Why Vector Search is Better:**
- 🧠 Understands **meaning**, not just words
- 🔎 Finds **semantic matches** ("hotel" matches "accommodation")
- 🌍 Works across **languages** (similar concepts have similar vectors)
- 📊 **Quantifiable** similarity (cosine distance between vectors)

---

## 🎯 How It Works

### The RAG Flow

```python
# 1. User asks a question
user_query = "Where can I stay in New York?"

# 2. App configuration tells Azure OpenAI to use your data
rag_params = {
    "data_sources": [{
        "type": "azure_search",
        "parameters": {
            "endpoint": "https://your-search.search.windows.net",
            "index_name": "brochures-index",
            "query_type": "vector",  # 🎯 Semantic search!
            "embedding_dependency": {
                "deployment_name": "text-embedding-3-small"
            }
        }
    }]
}

# 3. Behind the scenes:
#    a. Query → Embedding model → Vector [1536 numbers]
#    b. Vector search in Azure AI Search → Find similar doc vectors
#    c. Retrieved docs → Added to GPT-4o context
#    d. GPT-4o → Generates answer using YOUR data

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_query}],
    extra_body=rag_params  # 🔑 This enables RAG!
)
```

### What Happens Step-by-Step

1. **Query Vectorization**: Your question becomes a 1536-dimensional vector
2. **Similarity Search**: Azure AI Search finds documents with similar vectors (cosine similarity)
3. **Context Retrieval**: Most relevant documents are retrieved
4. **Augmented Prompt**: Documents are added to GPT-4o's context
5. **Smart Generation**: GPT-4o generates answer grounded in YOUR data

---

## 🚀 Quick Start

### Prerequisites

- Python 3.8+
- Azure subscription
- Azure OpenAI resource (with GPT-4o and text-embedding-3-small deployed)
- Azure AI Search service

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/frankwiersma/azure-rag-chat-app.git
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

### Example Usage

```
Enter the prompt (or type 'quit' to exit): Where can I stay in New York?Assistant: Based on the available information, you can stay at the Contoso Hotel 
in Times Square, with rates starting from $200 per night. The hotel offers 
convenient access to major attractions like the Statue of Liberty and Central 
Park.

Enter the prompt (or type 'quit' to exit): quit
```

---

## 🎓 Key Concepts

### Embeddings Explained

**Embeddings** convert text into numerical vectors that capture semantic meaning:

```python
Text: "New York hotel"
   ↓ (text-embedding-3-small)
Vector: [0.123, -0.456, 0.789, ..., 0.234]  # 1536 numbers
```

**Why this matters:**
- Similar meanings = Similar vectors
- "hotel" and "accommodation" have nearby vectors
- Enables semantic search (not just keyword matching)

### Vector Dimensions

- **text-embedding-3-small**: 1536 dimensions
- **text-embedding-3-large**: 3072 dimensions
- Your index dimensions **must match** your embedding model!

### HNSW Algorithm

**Hierarchical Navigable Small World** - Fast approximate nearest neighbor search

- Builds a graph structure of document vectors
- Searches hierarchically (coarse → fine)
- Much faster than brute-force search for large datasets

---

## 📊 Project Structure

```
azure-rag-chat-app/
├── .gitignore              # Protects sensitive files
├── .env.example            # Configuration template
├── README.md               # You are here!
├── SETUP-GUIDE.md          # Detailed CLI commands & setup
├── rag-app.py              # Main RAG application
└── requirements.txt        # Python dependencies
```

---

## ⚙️ Configuration

### Environment Variables

```env
# Azure OpenAI
OPEN_AI_ENDPOINT=https://your-resource.cognitiveservices.azure.com/
OPEN_AI_KEY=your_api_key_here

# Models
CHAT_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small

# Azure AI Search
SEARCH_ENDPOINT=https://your-search.search.windows.net
SEARCH_KEY=your_search_key_here
INDEX_NAME=brochures-index
```

### Index Schema

```json
{
  "name": "brochures-index",
  "fields": [
    {
      "name": "contentVector",
      "type": "Collection(Edm.Single)",
      "dimensions": 1536,
      "vectorSearchProfile": "vector-profile"
    }
  ]
}
```

**Important**: Use `Edm.Single` (not `Edm.Double`) for vector fields!

---

## 🔧 Setup

See [SETUP-GUIDE.md](SETUP-GUIDE.md) for detailed instructions on:

- Deploying Azure OpenAI models (GPT-4o, embeddings)
- Creating Azure AI Search index with vector fields
- Uploading documents with embeddings
- Complete Azure CLI commands

---

## 🚨 Troubleshooting

### Common Issues

**Issue**: "DeploymentNotFound"
```bash
# Check your model deployments
az cognitiveservices account deployment list \
  --name YOUR_RESOURCE_NAME \
  --resource-group YOUR_RG
```

**Issue**: "Index not found"
```bash
# List all indexes
curl -X GET "https://YOUR_SEARCH.search.windows.net/indexes?api-version=2024-07-01" \
  -H "api-key: YOUR_KEY"
```

**Issue**: Vector search not working
- ✅ Verify index has vector field with correct dimensions (1536)
- ✅ Ensure documents have embeddings uploaded
- ✅ Check `query_type` is set to `"vector"`
- ✅ Verify embedding deployment name matches

---

## 🎉 Success Criteria

Your RAG app is working when:

1. ✅ You can ask questions about your data
2. ✅ Answers include specific information from your documents
3. ✅ Semantic search works (finds related concepts, not just keywords)
4. ✅ The app can answer "What attractions are in Dubai?" by finding hotel/tour info

---

## 📚 Learn More

### Azure Documentation
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/)
- [Azure AI Search](https://learn.microsoft.com/azure/search/)
- [Vector Search in Azure AI Search](https://learn.microsoft.com/azure/search/vector-search-overview)

### RAG Pattern
- [Use Your Own Data with Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/concepts/use-your-data)
- [RAG Pattern Best Practices](https://learn.microsoft.com/azure/ai-services/openai/how-to/use-your-data-securely)

### Embeddings
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [Text Embedding Models](https://learn.microsoft.com/azure/ai-services/openai/concepts/models#embeddings-models)

---

## 🤝 Contributing

Contributions welcome! Please feel free to submit a Pull Request.

---

## 📝 License

MIT License - See LICENSE file for details

---

## 🙏 Acknowledgments

Built as part of the Azure AI Foundry learning path. Special thanks to the Azure AI team for excellent documentation and examples.

---

**Built with:** Azure CLI, Azure OpenAI, Azure AI Search, Python  
**Pattern:** Retrieval Augmented Generation (RAG)  
**Author:** Frank Wiersma  
**Date:** November 2025

