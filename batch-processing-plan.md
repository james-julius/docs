# Batch Processing Plan for Small Documents

**Status**: Planned, not yet implemented
**Created**: 2025-12-06
**Target**: Optimize processing of bulk markdown/text file uploads

---

## Problem Statement

Current system processes each document individually in its own Modal container:
- **5000 markdown files** = 5000 containers
- **Cost**: ~$15
- **Time**: 15-20 minutes
- **Bottleneck**: Container startup overhead + separate embedding API calls

---

## Proposed Solution

Implement smart batching for small files while preserving single-document flow for large files.

### Performance Impact (5000 markdown files):
- **Before**: 5000 containers, ~15-20 minutes, ~$15 cost
- **After**: 50 containers (batches of 100), ~5-10 minutes, ~$0.50 cost
- **Savings**: 97% cost reduction, 67% faster

---

## Current Upload Flow

```
User uploads 5000 markdown files:

1. Next.js → FastAPI /bulk-upload-urls → generates 5000 R2 upload URLs
2. User uploads all 5000 files to R2 in parallel
3. R2 triggers 5000 webhooks (one per file) → Cloudflare Queue → FastAPI /webhook
4. FastAPI creates 5000 ProcessingTask records
5. FastAPI calls Modal 5000 times → process_document_modal()
6. Modal spawns 5000 containers (max 100 concurrent)
7. Each container:
   - Downloads 1 file from R2
   - Parses markdown
   - Chunks content
   - Calls embedding API (separate call per file)
   - Saves to Milvus
   - Updates task status

Result: 5000 containers, ~15-20 minutes, ~$15 cost
```

---

## Proposed Batch Flow

```
User uploads 5000 markdown files (detected as batchable):

1. Next.js → FastAPI /bulk-upload-urls → generates 5000 R2 upload URLs
2. User uploads all 5000 files to R2 in parallel
3. R2 triggers 5000 webhooks → Cloudflare Queue → FastAPI /webhook
4. FastAPI batching logic:
   - Creates 5000 ProcessingTask records
   - Accumulates files in BatchQueue
   - Groups files by type and size → 50 batches of 100 files
   - Calls Modal 50 times → process_document_batch_modal()

5. Modal spawns 50 batch containers (can run in parallel)
6. Each batch container:
   - Downloads 100 files from R2 in parallel
   - Parses all 100 markdown files sequentially
   - Chunks all content
   - **Batches embedding calls** (100 files × 5 chunks = 500 chunks per API call)
   - Saves all to Milvus in bulk
   - Updates all 100 task statuses

Result: 50 containers, ~5-10 minutes, ~$0.50 cost
```

---

## Key Batching Insertion Point

**File**: `fastapi/routers/webhooks.py`
**Line**: 321 (in R2 upload webhook handler)

### Current Code:
```python
# Queue document for processing via Modal
document.status = DocumentStatus.PROCESSING

result = await modal_client.process_document(
    dataset_id=event.datasetId,
    object_key=object_key,
    task_id=str(document.id),  # ONE document at a time
    ...
)
```

### Proposed Batching Logic:
```python
from services.batch_optimizer import BatchOptimizer, FileInfo

# Create file info
file_info = FileInfo(
    filename=document.file_name,
    size=document.file_size or 0,
    extension=os.path.splitext(document.file_name)[1],
    object_key=object_key,
    task_id=str(document.id)
)

# Add to batch queue
batch_queue.add_document(dataset_id, file_info)

# Check if batch is ready
pending_batch = batch_queue.get_pending_batch(dataset_id)

if BatchOptimizer.should_batch_documents(pending_batch):
    # Process as batch
    result = await modal_client.process_document_batch(
        dataset_id=dataset_id,
        documents=[{"object_key": d.object_key, "task_id": d.task_id} for d in pending_batch],
    )
else:
    # Process individually (existing flow)
    result = await modal_client.process_document(...)
```

---

## Implementation Components

### 1. ✅ Batch Detection Logic (DONE)

**File**: `fastapi/services/batch_optimizer.py`

Already created with:
- `should_batch_documents()` - determines if files should be batched
- `create_batches()` - splits large batches into optimal sizes
- `estimate_processing_time()` - cost/time estimation
- `estimate_cost()` - cost analysis

**Batching Criteria**:
- Minimum 50 files
- Maximum 100 files per batch
- All files < 100KB
- All files same extension (same processing logic)

---

### 2. Add Batch Queue Service (NEW)

**File**: `fastapi/services/batch_queue.py`

```python
from typing import Dict, List
from datetime import datetime
from services.batch_optimizer import FileInfo

class BatchQueue:
    """In-memory queue for accumulating documents for batch processing."""

    def __init__(self):
        self._queues: Dict[str, List[FileInfo]] = {}  # dataset_id -> files
        self._timers: Dict[str, datetime] = {}  # dataset_id -> first_added_time

    def add_document(self, dataset_id: str, file_info: FileInfo):
        """Add document to batch queue."""
        if dataset_id not in self._queues:
            self._queues[dataset_id] = []
            self._timers[dataset_id] = datetime.utcnow()

        self._queues[dataset_id].append(file_info)

    def get_pending_batch(self, dataset_id: str) -> List[FileInfo]:
        """Get pending batch for dataset if ready."""
        if dataset_id not in self._queues:
            return []

        files = self._queues[dataset_id]
        time_elapsed = (datetime.utcnow() - self._timers[dataset_id]).seconds

        # Trigger batch if:
        # 1. Reached batch size (100 files)
        # 2. Timeout expired (5 seconds since first file)
        if len(files) >= 100 or time_elapsed >= 5:
            return self._queues.pop(dataset_id)

        return []

# Global singleton
_batch_queue = None

def get_batch_queue() -> BatchQueue:
    global _batch_queue
    if _batch_queue is None:
        _batch_queue = BatchQueue()
    return _batch_queue
```

---

### 3. Update Webhook Handler

**File**: `fastapi/routers/webhooks.py` (Line 288-436)

Modify the Modal backend section to use batching:

```python
from services.batch_queue import get_batch_queue
from services.batch_optimizer import BatchOptimizer, FileInfo

# After updating document status to UPLOADED

# Initialize batch queue (singleton)
batch_queue = get_batch_queue()

# Add to batch
file_info = FileInfo(
    filename=document.file_name,
    size=document.file_size or 0,
    extension=os.path.splitext(document.file_name)[1],
    object_key=object_key,
    task_id=str(document.id)
)

batch_queue.add_document(str(dataset_id), file_info)

# Check if batch ready
pending_batch = batch_queue.get_pending_batch(str(dataset_id))

if pending_batch and BatchOptimizer.should_batch_documents(pending_batch):
    # Mark all as PROCESSING
    for file in pending_batch:
        doc = session.get(Document, uuid.UUID(file.task_id))
        doc.status = DocumentStatus.PROCESSING
    session.commit()

    # Process as batch
    result = await modal_client.process_document_batch(
        dataset_id=str(dataset_id),
        documents=[{
            "object_key": f.object_key,
            "task_id": f.task_id
        } for f in pending_batch],
    )
elif not pending_batch:
    # Still accumulating, return success (batch will process later)
    return {"status": "queued_for_batch", "document_id": str(document.id)}
else:
    # Not enough for batch, process individually
    document.status = DocumentStatus.PROCESSING
    session.commit()

    result = await modal_client.process_document(...)  # Existing flow
```

---

### 4. Add Batch Modal Client Method

**File**: `fastapi/services/modal_client.py`

```python
async def process_document_batch(
    self,
    dataset_id: str,
    documents: List[Dict[str, str]],  # [{"object_key": ..., "task_id": ...}]
) -> Dict[str, Any]:
    """Trigger Modal batch document processing."""

    payload = {
        "dataset_id": dataset_id,
        "documents": documents,  # Multiple documents
        "batch_size": len(documents),
    }

    response = await self.client.post(
        f"{self.modal_webhook_url}/batch",  # New batch endpoint
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=30.0,  # Just trigger, don't wait for processing
    )

    return response.json()
```

---

### 5. Add Batch Modal Function

**File**: `modal/modal_app.py`

```python
@app.function(
    image=image,
    cpu=8.0,  # More CPU for batch processing
    memory=8192,  # 8GB for processing multiple files
    timeout=3600,  # 1 hour for large batches
    secrets=[...],
    max_containers=20,  # Limit concurrent batches
)
@modal.web_endpoint(method="POST", label="process-document-batch")
async def process_document_batch_webhook(request: Dict) -> Dict:
    """Process multiple documents in a single container."""

    dataset_id = request["dataset_id"]
    documents = request["documents"]  # List of {object_key, task_id}

    logger.info(f"📦 Batch processing {len(documents)} documents")

    # Download all files from R2 in parallel
    from services.storage import get_r2_client
    r2 = get_r2_client()

    files = await asyncio.gather(*[
        r2.download_file(doc["object_key"])
        for doc in documents
    ])

    # Process all files
    all_chunks = []
    for idx, (file_bytes, doc) in enumerate(zip(files, documents)):
        try:
            # Parse document
            parsed = await parse_document(file_bytes, doc["object_key"].split("/")[-1])

            # Chunk document
            chunks = await chunk_document(parsed, dataset_id, doc["object_key"])

            # Add task_id to each chunk for tracking
            for chunk in chunks:
                chunk["document_id"] = doc["task_id"]

            all_chunks.extend(chunks)

        except Exception as e:
            logger.error(f"Failed to process {doc['object_key']}: {e}")
            # Update individual document status to FAILED
            await update_document_status(doc["task_id"], "failed", error=str(e))

    # Batch embed ALL chunks (big performance win!)
    if all_chunks:
        from services.embeddings import batch_embed_chunks
        embedded_chunks = await batch_embed_chunks(all_chunks)  # 1 API call for 500+ chunks!

        # Save to Milvus in bulk
        await save_chunks_to_milvus(embedded_chunks)

    # Update all document statuses to READY
    for doc in documents:
        await update_document_status(doc["task_id"], "ready")

    return {
        "status": "success",
        "documents_processed": len(documents),
        "total_chunks": len(all_chunks)
    }
```

---

### 6. Preserve Single-Document Path

No changes needed to existing `process_document_modal()` - it continues to work for:
- Large files (>100KB)
- Mixed file types in same upload
- Small batches (<50 files)
- Any edge cases where batching doesn't apply

---

## Batching Strategy

- **Batch size**: 50-100 files per container
- **Grouping**: By file type (all .md, all .txt, etc.)
- **Embedding optimization**: Batch 5000 chunks → 50 API calls instead of 250
- **Processing**: Sequential within batch (simple, no parallel overhead)
- **Triggers**:
  - Size trigger: 100 files accumulated
  - Time trigger: 5 seconds since first file

---

## Implementation Checklist

- [ ] Create `BatchQueue` service (`fastapi/services/batch_queue.py`)
- [ ] Update webhook handler (`fastapi/routers/webhooks.py:321`)
- [ ] Add `process_document_batch()` to Modal client (`fastapi/services/modal_client.py`)
- [ ] Add `process_document_batch_webhook()` Modal function (`modal/modal_app.py`)
- [ ] Implement batch error handling
- [ ] Add comprehensive tests:
  - [ ] Test batch processing with 100 markdown files
  - [ ] Test fallback to individual processing
  - [ ] Test batch failure handling
  - [ ] End-to-end integration test
- [ ] Deploy and validate with real batch upload

**Estimated Effort**: 3-5 days

---

## Critical Files to Review Before Implementation

- `modal/modal_app.py` - Modal functions
- `fastapi/routers/dataset.py` - Upload endpoints
- `fastapi/routers/webhooks.py` - R2 webhook handler
- `fastapi/services/modal_client.py` - Modal webhooks
- `modal/services/embeddings.py` - Embedding batching

---

## Complexity Assessment

**Where We're Adding Complexity:**
- BatchQueue service (medium - ~100 lines)
- Webhook batching logic (medium - ~50 lines)
- Modal batch function (large - ~100 lines)
- Batch Modal client method (small - ~20 lines)

**Where We're Maintaining Simplicity:**
- Keep single-document path unchanged
- Batching only for obvious cases (many small files)
- No complex scheduling or queueing
- Fail fast on batch errors (no retry logic)

**Verdict**: Moderate complexity increase for significant performance gains on regular batch workloads.

---

## Related Work

**Phase 1 (Legacy Formats)**: ✅ COMPLETED
- LibreOffice conversion for .doc, .rtf, .odt, .xls, .ppt
- All tests passing (6 passed, 1 skipped for local LibreOffice)
- Files modified:
  - `modal/modal_config.py` - Added LibreOffice dependencies
  - `modal/services/parsing/legacy_converter.py` - Conversion logic
  - `modal/services/parsing/document_parsers.py` - Parser routing
  - `modal/tests/test_legacy_converter.py` - Test suite
