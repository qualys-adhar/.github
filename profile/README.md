# Book Recommendation System - Architecture

## 🏗️ System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (React/Vue)                     │
│                  (Book Upload, Search, Recs)                │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────┐
│              Node.js Backend (TypeScript)                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ API Routes                                           │   │
│  │  • POST /analyze (PDF upload & analysis)             │   │
│  │  • POST /ml/embed (Embed & index book)               │   │
│  │  • POST /ml/recommend (Get recommendations)          │   │
│  │  • GET /ml/stats (Indexing progress)                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Services                                             │   │
│  │  • book.service.ts (Embedding operations)            │   │
│  │  • recommendation.service.ts (Similarity search)     │   │
│  │  • ml.client.ts (ML service communication)           │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Storage                                              │   │
│  │  • MongoDB (metadata, embeddings)                    │   │
│  │  • AWS S3 (PDF files)                                │   │
│  └──────────────────────────────────────────────────────┘   │
└────────┬────────────────────────────────────┬────────────────┘
         │                                    │
         ↓                                    ↓
┌──────────────────────┐      ┌──────────────────────────────┐
│    AWS S3            │      │  FastAPI ML Service (Python) │
│  • PDF storage       │      │  ┌──────────────────────────┐│
│  • Signed URLs       │      │  │ Encoder                  ││
│  • Download links    │      │  │ • DistilBERT/MiniLM     ││
└──────────────────────┘      │  │ • 384-dim embeddings    ││
                              │  └──────────────────────────┘│
         ↓                     │  ┌──────────────────────────┐│
┌──────────────────────┐       │  │ FAISS Index              ││
│    MongoDB           │       │  │ • Inner Product (IVF)   ││
│  ┌────────────────┐  │       │  │ • Fast similarity search ││
│  │ Books          │  │       │  │ • GPU/CPU optimized     ││
│  │ ├─ title       │  │       │  └──────────────────────────┘│
│  │ ├─ author      │  │       │  ┌──────────────────────────┐│
│  │ ├─ embedding[] │  │ (read)│  │ Training (Contrastive)   ││
│  │ ├─ genre       │  │◄──────  │ • Self-supervised        ││
│  │ ├─ indexedAt   │  │       │  │ • Learn from book pairs ││
│  │ └─ s3Key       │  │       │  └──────────────────────────┘│
│  └────────────────┘  │       │                             │
└──────────────────────┘       │  ┌──────────────────────────┐│
                               │  │ REST API                 ││
                               │  │ • /embed                 ││
                               │  │ • /search                ││
                               │  │ • /similarity            ││
                               │  │ • /health                ││
                               │  └──────────────────────────┘│
                               └──────────────────────────────┘
```

## 📊 Data Flow

### 1. Book Upload & Embedding

```
User Upload PDF
    ↓
Node Backend
    ├─ Extract Text (pdf.js)
    ├─ Save to MongoDB
    ├─ Upload to S3
    │
    └─→ POST /ml/embed
            ↓
        ML Service
            ├─ Encode text → 384-dim vector
            ├─ Store embedding in memory
            │
            └─→ Add to FAISS index
                    ↓
        ML Backend (Response)
            ├─ Save embedding to MongoDB
            └─ Update indexedAt timestamp
```

### 2. Book Recommendation Search

```
User Requests Similar Books
    ↓
Node Backend
    └─→ POST /ml/recommend
            ↓
        ML Service
            ├─ Encode query text
            │
            └─→ Search FAISS index
                    ├─ Find k nearest neighbors
                    └─ Return book IDs + scores
                        ↓
        Node Backend
            ├─ Fetch metadata from MongoDB
            ├─ Combine with similarity scores
            │
            └─→ Return to Frontend
                (title, author, genre, similarity)
```

### 3. Text Analysis (Existing)

```
PDF Upload
    ↓
Extract Text (page by page)
    ↓
Word Frequency Analysis
    ├─ Tokenize
    ├─ Remove stopwords
    └─ Count occurrences
        ↓
Save Top Words to MongoDB
    ├─ Word list
    ├─ Frequencies
    └─ Total word count
```

## 🔗 Integration Points

### Node.js ↔ ML Service Communication

**Protocol:** HTTP/JSON
**Client:** Axios
**Base URL:** `process.env.ML_SERVICE_URL` (default: `http://localhost:8000`)

```typescript
// src/ml/ml.client.ts
class MLClient {
  async embed(text: string): Promise<number[]>;
  async search(query: string, k: number): Promise<SearchResult[]>;
  async similarity(textA: string, textB: string): Promise<number>;
  async indexAdd(
    bookId: string,
    text: string,
    metadata?: any,
  ): Promise<boolean>;
  async health(): Promise<MLServiceHealth>;
}
```

**Error Handling:**

- Timeout: 30 seconds
- Retry: Application level (not built-in)
- Fallback: Graceful degradation on ML service down

## 🗄️ Database Schema

### MongoDB Collections

#### `books` Collection

```typescript
{
  _id: ObjectId,

  // Book metadata
  title: String,
  author: String,
  genre: String, // NEW: for recommendations
  s3Key: String,

  // Text analysis (existing)
  topWords: [
    { word: String, count: Number },
    ...
  ],
  totalWords: Number,
  analysisProgress: Number,

  // ML integration (NEW)
  embedding: [Number], // 384-dimensional vector
  indexedAt: Date,     // When added to FAISS

  // Timestamps
  createdAt: Date,
  updatedAt: Date,
}
```

### Indexes

```
// Existing
- (title, author) UNIQUE

// NEW
- (indexedAt, genre)
- (embedding) [for vector searches if using similarity]
```

## 🎯 Component Responsibilities

### Frontend

- Display books
- Upload PDFs
- Show recommendations
- Search interface

### Node.js Backend

- **Orchestration:** Route requests to correct service
- **Data Management:** MongoDB CRUD
- **File Handling:** S3 uploads/downloads
- **ML Orchestration:** Call ML service APIs
- **API Gateway:** REST endpoints for frontend

### ML Service (FastAPI)

- **Text Encoding:** Convert text to 384-dim vectors
- **FAISS Indexing:** Store vectors, enable fast search
- **Similarity Computation:** Compare texts
- **Model Management:** Train and save models
- **Inference:** Serve embeddings and search

### MongoDB

- Persist book metadata
- Store embeddings for backup
- Track indexing progress
- Audit trail

### AWS S3

- Store PDFs
- Generate download links
- Backup documents

## 🔐 Security Considerations

### API Security

- ✅ CORS enabled (from Hono)
- ✅ Environment variables for secrets
- ⚠️ TODO: Authentication (JWT)
- ⚠️ TODO: Rate limiting

### Data Protection

- ✅ S3 signed URLs (time-limited)
- ✅ MongoDB credentials in .env
- ⚠️ TODO: Encryption at rest
- ⚠️ TODO: Encryption in transit (HTTPS)

### ML Service Security

- ✅ Internal network (Docker)
- ⚠️ TODO: API key authentication
- ⚠️ TODO: Input validation

## 📈 Scalability

### Current Design

- **Backend:** Single Node.js instance (can scale horizontally)
- **ML Service:** Single Python instance (can use GPU)
- **Database:** MongoDB Atlas (managed scaling)
- **Storage:** S3 (unlimited scale)

### Scaling Strategy

#### Horizontal Scaling (Multiple Instances)

```yaml
# Use load balancer (Nginx, AWS ALB)
Load Balancer
├─ Backend Instance 1
├─ Backend Instance 2
├─ Backend Instance 3
└─ (all sharing MongoDB + S3)
```

#### ML Service Scaling

```
- Single instance: Up to 10k books (slow)
- Multiple instances: 10k-100k books (with cache)
- Distributed FAISS: 100k+ books (needs architecture change)
```

#### Database Scaling

```
- MongoDB Atlas auto-scaling
- Sharding if >100M books
- Replication for HA
```

## 🧪 Testing Strategy

### Unit Tests

```typescript
// Services
-embedBook() -
  getRecommendations() -
  filterByThreshold() -
  // Controllers
  embedBookHandler() -
  recommendationsHandler();
```

### Integration Tests

```bash
# ML Service availability
POST /ml/health

# End-to-end flow
1. Upload PDF
2. Embed
3. Search recommendations
4. Verify results
```

### Load Tests

```
- 1000 concurrent embeds
- 10000 concurrent searches
- FAISS index with 100k books
```

## 🚀 Deployment

### Development

```bash
# Terminal 1: Node backend
cd backend && pnpm dev

# Terminal 2: ML service
cd ml && python -m uvicorn src.api.app:app --reload

# Terminal 3: MongoDB
mongod

# Terminal 4: Tests
bash test-ml-integration.sh
```

### Production

```bash
# Docker Compose
docker-compose up -d

# Kubernetes (future)
kubectl apply -f k8s/
```

## 📊 Performance Targets

| Operation               | Target | Current |
| ----------------------- | ------ | ------- |
| Embed book (2000 words) | <5s    | ?       |
| Search 10k books        | <100ms | ?       |
| Return recommendations  | <500ms | ?       |
| Health check            | <100ms | ?       |

## 🔄 Future Enhancements

1. **Async Processing**
   - Queue embeddings (Bull, Kafka)
   - Process in background
   - Progress tracking

2. **Caching**
   - Redis for embeddings
   - Cache recommendations
   - Warm FAISS index

3. **Advanced Features**
   - Real-time collaboration
   - Recommendation explanations
   - A/B testing
   - Personalized recommendations

4. **Model Improvements**
   - Fine-tune on your books
   - Multi-language support
   - Domain-specific training

5. **Monitoring**
   - Prometheus metrics
   - ELK stack logging
   - Distributed tracing

---

**Architecture Version:** 2.0
**Last Updated:** Jan 2025
**Status:** Implementation Complete, Testing In Progress
