# Integrate Cloudflare Images for R2 Image Serving

## Problem Analysis

Currently:
- Images stored in R2
- FastAPI generates presigned URLs (1-hour expiry) via boto3
- Existing R2 image proxy worker serves images but doesn't use Cloudflare Images
- No image transformations/optimizations
- Direct R2 access (no CDN benefits)

**Benefits of Cloudflare Images:**
- On-the-fly transformations (resize, format conversion, quality)
- CDN delivery (faster than direct R2)
- Better caching
- Can serve images from R2 without migration
- Zero egress costs (already achieved with Worker, but Images adds optimization)

## Implementation Strategy

Use Cloudflare Images transformation URLs to serve images from R2. This approach:
- Keeps images in R2 (no migration needed)
- Uses Cloudflare Images as transformation/CDN layer
- Maintains existing Worker for access control
- Adds transformation capabilities via URL parameters

## Implementation Plan

### 1. Add Cloudflare Images Binding to Worker

**File**: `cloudflare-workers/r2-image-proxy/wrangler.toml`

- Add Images binding to Worker configuration
- Requires Cloudflare Images Paid subscription

**Changes**:
```toml
[images]
binding = "IMAGES"
```

### 2. Update Worker to Use Cloudflare Images

**File**: `cloudflare-workers/r2-image-proxy/src/index.ts`

- Add Images binding to Env interface
- Transform images from R2 using Cloudflare Images API
- Support transformation parameters from URL query string
- Fallback to direct R2 streaming if Images fails

**Key Changes**:
- Read image from R2 as stream
- Use `env.IMAGES.input(r2Stream)` to create transformation pipeline
- Apply transformations from query params (width, height, format, quality)
- Return transformed image

**Transformation Parameters**:
- `width`, `height` - Resize dimensions
- `format` - Output format (avif, webp, jpeg, png)
- `quality` - Quality 1-100 (default: 85)
- `fit` - Resize fit mode (contain, cover, scale-down, etc.)

### 3. Create Cloudflare Images URL Service in FastAPI

**New File**: `fastapi/services/cloudflare_images_service.py`

- Service to generate Cloudflare Images transformation URLs
- Replaces presigned URL generation for images
- Supports transformation parameters
- Falls back to presigned URLs if Images not configured

**Key Functions**:
- `generate_image_url(r2_key: str, transformations: dict) -> str`
- `get_transformation_params(request: Request) -> dict`
- Validates transformation parameters

**URL Format**:
```
https://imagedelivery.net/{account_hash}/{image_id}/{variant_name}
```

For R2 images, use transformation URLs:
```
https://{worker_url}/{dataset_id}/{document_id}/images/{filename}?width=800&format=webp
```

### 4. Update FastAPI Asset Endpoints

**File**: `fastapi/routers/dataset.py`

- Replace presigned URL generation with Cloudflare Images URLs
- Add transformation parameters from query string
- Keep presigned URL as fallback

**Changes**:
- `list_document_assets`: Generate Images URLs instead of presigned URLs
- `get_document_asset`: Generate Images URLs with transformations
- Add query parameters: `width`, `height`, `format`, `quality`

### 5. Update Frontend to Use Transformation URLs

**File**: `nextjs/src/app/(public)/datasets/[slug]/documents/[documentId]/page.tsx`

- Use Cloudflare Images URLs from API
- Add transformation parameters for thumbnails/previews
- Use responsive image sizes

**Changes**:
- AssetCard: Use smaller width for thumbnails (e.g., `width=300`)
- Dialog: Use larger width for full-size (e.g., `width=1200`)
- Add format conversion to WebP/AVIF for better compression

### 6. Add Configuration

**File**: `fastapi/config/__init__.py`

- Add Cloudflare Images configuration
- `CLOUDFLARE_IMAGES_ENABLED: bool = False`
- `CLOUDFLARE_IMAGES_WORKER_URL: str = ""`
- `CLOUDFLARE_IMAGES_DEFAULT_FORMAT: str = "webp"`
- `CLOUDFLARE_IMAGES_DEFAULT_QUALITY: int = 85`

**File**: `nextjs/.env.example`

- Add `NEXT_PUBLIC_CLOUDFLARE_IMAGES_ENABLED=false`
- Add `NEXT_PUBLIC_CLOUDFLARE_IMAGES_WORKER_URL=""`

### 7. Update Worker README

**File**: `cloudflare-workers/r2-image-proxy/README.md`

- Document Cloudflare Images integration
- Add transformation parameter examples
- Update setup instructions for Images subscription

## Transformation Examples

**Thumbnail (AssetCard)**:
```
/{datasetId}/{documentId}/images/{filename}?width=300&format=webp&quality=80
```

**Full-size (Dialog)**:
```
/{datasetId}/{documentId}/images/{filename}?width=1200&format=webp&quality=85
```

**Original (no transformations)**:
```
/{datasetId}/{documentId}/images/{filename}
```

## Migration Strategy

1. **Phase 1**: Deploy Worker with Images support (backward compatible)
2. **Phase 2**: Update FastAPI to generate Images URLs (with fallback)
3. **Phase 3**: Update frontend to use transformation URLs
4. **Phase 4**: Monitor and optimize transformation parameters

## Testing

- Test Worker with Images transformations
- Test FastAPI URL generation
- Test frontend image loading
- Test fallback to presigned URLs
- Test transformation parameters
- Verify CDN caching

## Files to Modify

1. `cloudflare-workers/r2-image-proxy/wrangler.toml` - Add Images binding
2. `cloudflare-workers/r2-image-proxy/src/index.ts` - Add Images transformations
3. `fastapi/services/cloudflare_images_service.py` - New service for URL generation
4. `fastapi/routers/dataset.py` - Update asset endpoints
5. `fastapi/config/__init__.py` - Add Images configuration
6. `nextjs/src/app/(public)/datasets/[slug]/documents/[documentId]/page.tsx` - Use transformation URLs
7. `cloudflare-workers/r2-image-proxy/README.md` - Update documentation

## Prerequisites

- Cloudflare Images Paid subscription
- Worker already deployed and working
- R2 bucket access configured

## Benefits

- **Performance**: CDN delivery, optimized formats (WebP/AVIF)
- **Cost**: Reduced bandwidth (smaller file sizes)
- **Flexibility**: On-the-fly transformations without reprocessing
- **User Experience**: Faster image loading, responsive sizes

