# Image-Chunk Linking Fix

## Problem

When searching datasets with multimodal content (PDFs with images), all search results were displaying the **same image** regardless of which chunk was retrieved. For example, searching for "Rosemary", "Bay Leaves", and "Sage" in a cookbook would show the same herb image for all three results.

## Root Cause

The issue had two parts:

### 1. Ingestion Gap (Primary Issue)
During document ingestion in `multimodal_processor.py`:
- Images are extracted from PDFs and uploaded to R2 storage
- Image markdown references (`![alt](url)`) are replaced with `[IMAGE]` placeholders in text
- **Problem**: Text chunks containing `[IMAGE]` placeholders were NOT being linked to their corresponding image R2 keys
- Only dedicated "image chunks" (containing OCR/vision data) had `image_r2_key` in metadata

### 2. Retrieval Fallback (Secondary Issue)
In `routers/mcp.py`, there was fallback code that attempted to fix missing images:
```python
# OLD CODE - Problematic
if chunk_type == "text" and not has_image_r2_key:
    # Fetch first image from DocumentAssets
    asset = session.query(DocumentAsset).filter(...).first()
    if asset:
        metadata["image_url"] = generate_presigned_url(asset.r2_key)
```
This caused **every** text chunk from the same document to get the **same first image**.

## Solution

### Part 1: Fix at Ingestion Time (`modal/services/multimodal_processor.py`)

Modified `create_multimodal_chunks()` to properly link text chunks with their embedded images:

```python
# 1. Build a mapping of image positions in the original content
image_positions = []
for img_data in images:
    r2_key = img_data.get("r2_key", "")
    if not r2_key:
        continue

    # Find image markdown in content by matching R2 filename
    r2_filename = r2_key.split("/")[-1]
    pattern = rf'!\[[^\]]*\]\([^)]*{re.escape(r2_filename)}[^)]*\)'
    for match in re.finditer(pattern, content):
        image_positions.append({
            "start": match.start(),
            "r2_key": r2_key,
            "page": img_data.get("page", 0),
            "image_index": img_data.get("index", 0),
            ...
        })

# 2. Sort by position and create FIFO queue
image_positions.sort(key=lambda x: x["start"])
available_images = list(image_positions)

# 3. For each text chunk, assign images based on [IMAGE] placeholder count
for chunk in chunks:
    image_count = chunk.text.count("[IMAGE]")
    if image_count > 0 and available_images:
        chunk_images = []
        for _ in range(min(image_count, len(available_images))):
            chunk_images.append(available_images.pop(0))

        # Store first image as primary in metadata
        first_img = chunk_images[0]
        metadata.update({
            "image_r2_key": first_img["r2_key"],
            "page": first_img["page"],
            "image_index": first_img["image_index"],
            "embedded_image_count": len(chunk_images),
        })
```

### Part 2: Remove Fallback Code (`fastapi/routers/mcp.py`)

Removed the fallback that was fetching the first image for all text chunks:

```python
# NEW CODE - No fallback, rely on proper ingestion
else:
    # Text chunks without image_r2_key don't get images attached
    # (prevents incorrect image associations)
    pass
```

## Data Flow After Fix

```
PDF Document
    ↓
PyMuPDF4LLM extracts content + images
    ↓
Images uploaded to R2: {dataset_id}/{document_id}/images/{page}_{index}.png
    ↓
create_multimodal_chunks():
    1. Maps image positions in original markdown
    2. Replaces images with [IMAGE] placeholders
    3. Chunks text semantically
    4. For each chunk with [IMAGE], assigns correct image_r2_key
    ↓
Chunks stored in Milvus with proper image_r2_key metadata
    ↓
At retrieval: presigned URL generated for the correct image
```

## Files Changed

| File | Change |
|------|--------|
| `modal/services/multimodal_processor.py` | Added image position tracking and FIFO assignment to text chunks |
| `fastapi/routers/mcp.py` | Removed fallback code that was assigning wrong images |

## Testing

- All FastAPI tests pass (689 passed)
- All Modal tests pass (120 passed)

## Migration Note

**Existing documents need to be re-ingested** to benefit from this fix. Chunks already in Milvus don't have the updated `image_r2_key` metadata. Options:

1. Delete and re-upload affected documents
2. Run a migration script to re-process documents
3. New documents will automatically have correct metadata
