# DMS
## Sample Document Management System (DMS) Architecture  

### 1. Core components  

| Component | Role | Typical technology choices |
|-----------|------|-----------------------------|
| **Web front‑end** | User interface for upload, search, versioning, permissions | React / Angular SPA, Bootstrap UI |
| **API layer** | REST/GraphQL endpoints that the front‑end and external apps call | Node.js (Express), Python (FastAPI), Java (Spring Boot) |
| **File storage** | Holds the binary files | S3‑compatible object store, Azure Blob, on‑premise NAS (via SFTP/FTPS) |
| **Metadata database** | Stores document attributes, version history, ACLs | PostgreSQL, MySQL, or a document DB like MongoDB |
| **Search engine** | Full‑text indexing and faceted search | Elasticsearch or OpenSearch |
| **Authentication & Authorization** | Central identity, role‑based access control | OAuth2 / OpenID Connect (Keycloak, Auth0) |
| **Background workers** | Asynchronous tasks: thumbnail generation, OCR, virus scanning | Celery (Python), Sidekiq (Ruby), BullMQ (Node) |
| **Audit & logging** | Immutable record of actions for compliance | ELK stack, Splunk, or CloudWatch Logs |

### 2. Data flow example (upload)  

1. **User** selects a file in the web UI.  
2. UI sends a **POST /documents** request with metadata (title, tags) to the API.  
3. API creates a **document record** in the metadata DB and generates a **pre‑signed URL** for the storage service.  
4. UI uploads the binary directly to the storage service using the pre‑signed URL (bypasses the API for large files).  
5. Storage service triggers a **notification** (e.g., S3 Event) that a new object exists.  
6. A **background worker** picks up the event, runs:  
   - Virus scan (ClamAV)  
   - OCR (Tesseract) for PDF/text extraction  
   - Thumbnail creation (ImageMagick)  
   - Indexes extracted text in Elasticsearch.  
7. Worker updates the document record with **status = “ready”** and links to generated assets.  

### 3. Sample database schema (PostgreSQL)  

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY,
    title           TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at      TIMESTAMP WITH TIME ZONE,
    owner_id        UUID NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    storage_key     TEXT NOT NULL,
    status          TEXT CHECK (status IN ('uploading','processing','ready','error')) DEFAULT 'uploading'
);

CREATE TABLE document_versions (
    id              UUID PRIMARY KEY,
    document_id     UUID REFERENCES documents(id) ON DELETE CASCADE,
    version_number  INT NOT NULL,
    storage_key     TEXT NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT now(),
    checksum        TEXT
);

CREATE TABLE document_tags (
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    tag         TEXT,
    PRIMARY KEY (document_id, tag)
);
```

### 4. Security considerations  

- **Transport encryption:** All UI‑API traffic over HTTPS; storage access via signed URLs or TLS.  
- **At‑rest encryption:** Enable bucket‑level encryption (AES‑256) and DB encryption.  
- **Access control:** ACLs stored per‑document; API checks user’s roles/permissions before serving or modifying a file.  
- **Data loss prevention:** Replicate storage across zones; regular DB backups; immutable audit logs.  

### 5. Minimal viable product (MVP) stack  

- **Front‑end:** React + Material‑UI  
- **API:** FastAPI (Python)  
- **Storage:** MinIO (self‑hosted S3‑compatible)  
- **DB:** PostgreSQL  
- **Search:** Elasticsearch Docker image  
- **Auth:** Keycloak (OpenID Connect)  
- **Workers:** Celery + Redis broker  

This combination can be spun up with Docker Compose for rapid prototyping, then migrated to managed services (AWS S3, RDS, Elastic Cloud) for production.
