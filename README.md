# Decentralized-Academic-Citation-Knowledge-Graph-Engine
A hackathon project that builds an automated knowledge graph from academic research papers, detecting cross-disciplinary connections, collaboration networks, and redund# 🧠 Decentralized Academic Citation & Knowledge Graph Engine

A hackathon project that builds an automated knowledge graph from academic research papers, detecting cross-disciplinary connections, collaboration networks, and redundant studies.

## 🚀 Quick Start

### Option 1: Local Development (Fastest for Hackathon)

```bash
# Clone/setup
git clone <repo>
cd knowledge-graph-engine

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export GCP_PROJECT_ID="your-project"
export ALLOYDB_CLUSTER="your-cluster"
export ALLOYDB_INSTANCE="your-instance"

# Run locally
python main.py
```

Visit `http://localhost:8080` and check `/health` endpoint.

### Option 2: Deploy to Cloud Run (Production)

```bash
# Follow DEPLOYMENT.md for full setup
# Quick version:
gcloud run deploy knowledge-graph-engine \
  --source . \
  --region us-central1 \
  --set-env-vars=GCP_PROJECT_ID=$PROJECT_ID
```

## 📋 Features

### ✅ Core Features (MVP)
- **PDF Upload & Processing** - Ingest research papers directly
- **Automated Entity Extraction** - Extract authors, papers, datasets using Vertex AI
- **Knowledge Graph Building** - Store relationships in AlloyDB with pgvector
- **Collaboration Detection** - Find researchers working together
- **Semantic Search** - Find similar papers using embeddings

### 🎯 Hackathon Differentiators
1. **Cross-Disciplinary Discovery** - Find papers from different departments that share methodologies
2. **Redundancy Detection** - Identify similar studies before they're published
3. **Hidden Collaboration Networks** - Surface connections between distant research groups
4. **Real-Time Graph Updates** - Add new papers and instantly see new connections

## 🏗️ Architecture

```
Client (React Dashboard)
    ↓
Cloud Run (API Service)
    ↓
    ├── Vertex AI (Entity Extraction + Embeddings)
    ├── Cloud Storage (PDF Management)
    └── AlloyDB with pgvector (Graph Database)
```

## 📦 API Endpoints

### Upload Paper
```bash
POST /upload-paper
Content-Type: multipart/form-data

Parameters:
- file: PDF file
- title: Paper title (optional)
- department: Department (optional)
- authors: Comma-separated author names (optional)

Response:
{
  "status": "success",
  "title": "My Research",
  "entities_extracted": 42,
  "relationships_extracted": 128
}
```

### Search Collaborations
```bash
GET /search-collaborations?author=John%20Doe

Response:
{
  "author": "John Doe",
  "collaborators": [
    {"author": "Jane Smith", "papers_together": 3},
    {"author": "Bob Johnson", "papers_together": 1}
  ]
}
```

### Find Similar Papers
```bash
GET /find-similar-papers?paper_title=Deep%20Learning%20Applications

Response:
{
  "query_paper": "Deep Learning Applications",
  "similar_papers": [
    {"title": "Neural Networks...", "similarity_score": 0.89},
    {"title": "AI Methods...", "similarity_score": 0.76}
  ]
}
```

### Health Check
```bash
GET /health

Response:
{"status": "healthy"}
```

## 💾 Database Schema

### Papers Table
```sql
CREATE TABLE papers (
  id SERIAL PRIMARY KEY,
  title VARCHAR(500) NOT NULL UNIQUE,
  abstract TEXT,
  department VARCHAR(200),
  keywords TEXT[],
  research_area VARCHAR(200),
  embedding vector(768),          -- pgvector embedding
  doi VARCHAR(100),
  ingested_at TIMESTAMP,
  updated_at TIMESTAMP
);

CREATE INDEX idx_papers_embedding ON papers 
  USING ivfflat(embedding vector_cosine_ops);
```

### Authors Table
```sql
CREATE TABLE authors (
  id SERIAL PRIMARY KEY,
  name VARCHAR(300) NOT NULL UNIQUE,
  affiliation VARCHAR(300),
  email VARCHAR(200)
);
```

### Relationships Table
```sql
CREATE TABLE relationships (
  id SERIAL PRIMARY KEY,
  source_entity VARCHAR(500),
  target_entity VARCHAR(500),
  relationship_type VARCHAR(100),    -- 'cites', 'collaborates', 'uses_dataset'
  context TEXT,
  created_at TIMESTAMP
);
```

## 🔧 Advanced Features

### 1. Detect Research Redundancy

```python
# Find papers with >85% similarity (likely redundant)
SELECT p1.title, p2.title, 
       1 - (p1.embedding <=> p2.embedding) as similarity
FROM papers p1, papers p2
WHERE p1.id < p2.id
AND 1 - (p1.embedding <=> p2.embedding) > 0.85
ORDER BY similarity DESC;
```

### 2. Cross-Disciplinary Collaboration Network

```python
# Find researchers collaborating across departments
SELECT DISTINCT a1.affiliation, a2.affiliation, COUNT(*) as collaboration_count
FROM paper_authors pa1
JOIN authors a1 ON pa1.author_id = a1.id
JOIN paper_authors pa2 ON pa1.paper_id = pa2.paper_id
JOIN authors a2 ON pa2.author_id = a2.id
WHERE a1.affiliation != a2.affiliation
GROUP BY a1.affiliation, a2.affiliation
ORDER BY collaboration_count DESC;
```

### 3. Citation Network Analysis

```python
# Find most cited papers
SELECT target_entity, COUNT(*) as citation_count
FROM relationships
WHERE relationship_type = 'cites'
GROUP BY target_entity
ORDER BY citation_count DESC
LIMIT 20;
```

### 4. Researcher Impact Score

```python
# Calculate h-index style metric
SELECT author, 
       COUNT(DISTINCT paper_id) as papers,
       AVG(citation_count) as avg_citations
FROM (
  SELECT a.name as author, pa.paper_id,
         COUNT(*) as citation_count
  FROM authors a
  JOIN paper_authors pa ON a.id = pa.author_id
  JOIN relationships r ON r.source_entity = (SELECT title FROM papers WHERE id = pa.paper_id)
  WHERE r.relationship_type = 'cites'
  GROUP BY a.name, pa.paper_id
) grouped
GROUP BY author
ORDER BY papers DESC;
```

## 🎨 Frontend Usage

The React dashboard (`frontend.jsx`) provides:
- 📄 **Paper Upload** - Drag & drop PDF upload
- 🔍 **Author Search** - Find collaborations by author name
- 📊 **Similarity Search** - Find related papers
- 📈 **Real-time Results** - Instant graph updates

### Deploy Frontend

```bash
# Create React app
npx create-react-app kg-frontend

# Copy frontend.jsx to src/components/
cp frontend.jsx src/components/KnowledgeGraphDashboard.jsx

# Update App.jsx
import KnowledgeGraphDashboard from './components/KnowledgeGraphDashboard';
export default function App() {
  return <KnowledgeGraphDashboard />;
}

# Deploy to Vercel/Netlify
npm run build
vercel deploy
```

## 📊 Performance Benchmarks

| Operation | Latency | Notes |
|-----------|---------|-------|
| PDF Upload & Extract | 5-30s | Depends on PDF size |
| Entity Extraction | 2-5s | Uses Vertex AI |
| Embedding Generation | 1-3s | Parallel processing |
| Similarity Search (100K papers) | <500ms | With pgvector index |
| Collaboration Query | <100ms | Simple join |

## 🔐 Security Features

- **IAM Authentication** - Cloud Run requires authentication
- **Row-Level Security** - AlloyDB with PostgreSQL RLS
- **Encrypted PDFs** - Cloud Storage with server-side encryption
- **VPC Isolation** - AlloyDB in private VPC
- **Audit Logging** - Cloud Audit Logs for all operations

## 💰 Cost Estimation

**Monthly costs (estimated):**
- Cloud Run: $5-20
- AlloyDB (small): $50-100
- Vertex AI: $10-50
- Cloud Storage: $5-10
- **Total: ~$70-180/month**

**Cost optimization:**
- Use committed discounts for AlloyDB
- Batch paper processing during off-hours
- Archive old papers to Cloud Archive Storage

## 🧪 Testing

```bash
# Test API locally
curl http://localhost:8080/health

# Test paper upload
curl -X POST http://localhost:8080/upload-paper \
  -F "file=@sample_paper.pdf" \
  -F "title=My Paper" \
  -F "department=CS"

# Test search
curl "http://localhost:8080/search-collaborations?author=John%20Doe"
```

## 📚 Example Use Cases

### 1. Prevent Duplicate Research
A biology department wants to start a study on "Gene editing in cancer cells". The system finds 3 similar papers in progress in other departments and suggests collaboration instead.

### 2. Discover Hidden Connections
Computer Science and Medical students using similar machine learning methodologies are matched for a joint thesis project.

### 3. Cross-Disciplinary Innovation
Economics department researchers using network analysis discover they can partner with Computer Science's graph algorithms experts.

## 🚀 Scaling Strategy

### Phase 1 (MVP - 1000 papers)
- Single Cloud Run instance
- AlloyDB small instance
- In-memory caching

### Phase 2 (10K papers)
- Load balancing across Cloud Run instances
- ReadyToServeAlloyDB instances (read replicas)
- Redis caching layer

### Phase 3 (100K+ papers)
- AlloyDB Enterprise
- Vertex AI AI search service
- Graph database optimization
- CDN for static assets

## 🤝 Contributing

To extend this project:

1. **Add More Entity Types** - Modify the extraction prompt in `main.py`
2. **Custom Visualizations** - Add graph visualization using D3.js or vis.js
3. **Advanced Analytics** - Implement community detection in the graph
4. **Multi-format Support** - Add arXiv API integration, Semantic Scholar API
5. **Export Features** - BibTeX, JSON, CSV exports

## 📝 Example Prompts for Entity Extraction

### Biology Paper
```
Extract: protein targets, disease areas, experimental methods, datasets used
```

### Computer Science Paper
```
Extract: algorithms, complexity analysis, datasets, benchmarks, programming languages
```

### Economics Paper
```
Extract: economic models, theories, datasets, time periods, geographic regions
```

## 🐛 Troubleshooting

### "Connection refused to AlloyDB"
- Check VPC connector is active
- Verify IAM permissions are set
- Check AlloyDB instance is running

### "PDF extraction failed"
- Ensure PDF is not encrypted
- Try PDF with text (not image-based scans)
- Check file size <100MB

### "Embedding generation timeout"
- Reduce batch size
- Check Vertex AI API quota
- Increase Cloud Run timeout

## 📖 Resources

- [AlloyDB Documentation](https://cloud.google.com/alloydb/docs)
- [Vertex AI Embeddings](https://cloud.google.com/vertex-ai/docs/generative-ai/embeddings)
- [pgvector Extension](https://github.com/pgvector/pgvector)
- [Cloud Run Deployment](https://cloud.google.com/run/docs)

## 🎯 Hackathon Tips

1. **Start with MVP** - Get basic upload and search working first
2. **Demo with sample papers** - Prepare 5-10 test PDFs
3. **Focus on one strength** - Collaboration detection or redundancy detection
4. **Visual demos** - Show the graph visualization
5. **Real use case** - Demo with actual university papers

## 📄 License

MIT License - feel free to modify and use for your hackathon!

---

**Built with ❤️ for PromptWars Hackathon**
ant studies.
