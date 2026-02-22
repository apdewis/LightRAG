# RAGAnything Integration Plan for LightRAG Server

## Overview

This plan integrates [RAGAnything](https://github.com/HKUDS/RAG-Anything) multimodal capabilities into the existing LightRAG API server, enabling image/table/equation processing alongside the existing text-only RAG pipeline. The integration is **opt-in** via environment variables, ensuring zero breakage of existing functionality.

---

## Architecture Summary

### Current Flow

```mermaid
flowchart LR
    A[Client Upload] --> B[/documents/upload]
    B --> C{File Type Check}
    C -->|Supported| D[pipeline_enqueue_file]
    D --> E[Text Extraction]
    E --> F[LightRAG.ainsert]
    C -->|Unsupported| G[HTTP 400]
    
    H[Client Query] --> I[/query]
    I --> J[rag.aquery_llm]
    J --> K[Response]
```

### Proposed Flow

```mermaid
flowchart LR
    A[Client Upload] --> B[/documents/upload]
    B --> C{File Type Check - extended}
    C -->|Text/PDF/Office| D[pipeline_enqueue_file - existing]
    C -->|Image - .png .jpg .jpeg .gif .bmp .webp .svg| E[pipeline_multimodal_file - new]
    E --> F[RAGAnything.process_multimodal_content]
    D --> G[LightRAG.ainsert]
    C -->|Unsupported| H[HTTP 400]
    
    I[Client Query] --> J{Multimodal enabled?}
    J -->|Yes + multimodal_content| K[rag_anything.aquery_with_multimodal]
    J -->|No or text-only| L[rag.aquery_llm - existing]
```

---

## Key Design Decisions

1. **Wrapper Pattern, Not Replacement**: RAGAnything wraps the existing `LightRAG` instance via `RAGAnything(lightrag=rag, vision_model_func=...)`. The existing `rag` object remains the primary instance for all text-only operations. RAGAnything is only invoked for multimodal-specific paths.

2. **Feature Flag**: Controlled by `ENABLE_MULTIMODAL=true` in `.env`. When disabled, the server behaves identically to today.

3. **Vision Model Configuration**: New env vars for the VLM binding, separate from the text LLM:
   - `VLM_BINDING` - openai, ollama, gemini, etc.
   - `VLM_MODEL` - e.g., gpt-4o, llava:latest
   - `VLM_BINDING_HOST` - e.g., http://localhost:11434
   - `VLM_BINDING_API_KEY` - API key for VLM provider

4. **Backward Compatibility**: All existing endpoints, request/response schemas, and `.env` variables remain unchanged.

---

## Files to Modify

### 1. [`lightrag/api/config.py`](lightrag/api/config.py) — Add Multimodal Config Args

**Changes:**
- Add new argparse arguments for multimodal configuration:
  - `--enable-multimodal` / `ENABLE_MULTIMODAL` (bool, default: false)
  - `--vlm-binding` / `VLM_BINDING` (str, default: same as `LLM_BINDING`)
  - `--vlm-model` / `VLM_MODEL` (str, default: gpt-4o)
  - `--vlm-binding-host` / `VLM_BINDING_HOST` (str, default: same as `LLM_BINDING_HOST`)
  - `--vlm-binding-api-key` / `VLM_BINDING_API_KEY` (str, default: same as `LLM_BINDING_API_KEY`)
  - `--multimodal-output-dir` / `MULTIMODAL_OUTPUT_DIR` (str, default: ./multimodal_output)

### 2. [`lightrag/api/lightrag_server.py`](lightrag/api/lightrag_server.py) — Initialize RAGAnything Wrapper

**Changes at ~line 1050-1088** (after `rag = LightRAG(...)` initialization):
- Conditionally import and initialize `RAGAnything` when `args.enable_multimodal` is true
- Create a `vision_model_func` using the same binding pattern as `create_llm_model_func()` but targeting the VLM config
- Instantiate `rag_anything = RAGAnything(lightrag=rag, vision_model_func=vision_model_func)`
- Store `rag_anything` on `app.state` or pass it to route factories

**Changes at ~line 1090-1103** (route registration):
- Pass `rag_anything` (or `None`) to `create_document_routes()` and `create_query_routes()`

### 3. [`lightrag/api/routers/document_routes.py`](lightrag/api/routers/document_routes.py) — Extend Upload Pipeline

#### 3a. [`DocumentManager`](lightrag/api/routers/document_routes.py:763) — Add Image Extensions

**Changes at ~line 768:**
- Add image extensions to `supported_extensions` tuple:
  ```python
  ".png", ".jpg", ".jpeg", ".gif", ".bmp", ".webp", ".svg"
  ```
- These should only be added when multimodal is enabled. Two approaches:
  - **Option A**: Always include them in the tuple but gate the processing logic (simpler)
  - **Option B**: Dynamically extend the tuple based on a flag passed to `DocumentManager.__init__()` (cleaner)
  - **Recommended**: Option B — add an `enable_multimodal` parameter to `DocumentManager.__init__()`

#### 3b. [`pipeline_enqueue_file()`](lightrag/api/routers/document_routes.py:1194) — Route Image Files

**Changes at ~line 1267** (the `match ext:` block):
- Add a new case for image extensions:
  ```python
  case ".png" | ".jpg" | ".jpeg" | ".gif" | ".bmp" | ".webp" | ".svg":
      # Route to multimodal processing
      if rag_anything is None:
          # Multimodal not enabled, reject
          error_files = [...]
          await rag.apipeline_enqueue_error_documents(error_files, track_id)
          return False, track_id
      # Process via RAGAnything
      await rag_anything.process_multimodal_content(...)
  ```

#### 3c. New Endpoint: `POST /documents/upload_multimodal` (Optional)

- Alternative to modifying the existing `/documents/upload` endpoint
- Accepts file + optional metadata (content type hints, captions)
- Routes directly to `rag_anything.process_document_complete()` or `process_multimodal_content()`
- This keeps the existing upload endpoint clean and adds a dedicated multimodal path

#### 3d. Update [`create_document_routes()`](lightrag/api/routers/document_routes.py:2042) Signature

- Accept `rag_anything` parameter (can be `None`)
- Pass it through to the pipeline functions

### 4. [`lightrag/api/routers/query_routes.py`](lightrag/api/routers/query_routes.py) — Add Multimodal Query Support

#### 4a. Extend [`QueryRequest`](lightrag/api/routers/query_routes.py:16) Model

**Add new optional field:**
```python
multimodal_content: Optional[list[dict]] = Field(
    default=None,
    description="List of multimodal content items for combined text+image queries. "
                "Each item has type, and type-specific data."
)
```

#### 4b. Modify [`query_text()`](lightrag/api/routers/query_routes.py:325) and [`query_text_stream()`](lightrag/api/routers/query_routes.py:535)

**Changes:**
- When `request.multimodal_content` is provided and `rag_anything` is not None:
  ```python
  result = await rag_anything.aquery_with_multimodal(
      request.query,
      multimodal_content=request.multimodal_content,
      mode=request.mode,
  )
  ```
- When `multimodal_content` is None or `rag_anything` is None, use existing `rag.aquery_llm()` path

#### 4c. Update [`create_query_routes()`](lightrag/api/routers/query_routes.py:193) Signature

- Accept `rag_anything` parameter (can be `None`)

### 5. [`lightrag_webui/src/lib/constants.ts`](lightrag_webui/src/lib/constants.ts:42) — Add Image MIME Types

**Changes at ~line 86:**
```typescript
export const supportedFileTypes = {
  // ... existing types ...
  'image/png': ['.png'],
  'image/jpeg': ['.jpg', '.jpeg'],
  'image/gif': ['.gif'],
  'image/bmp': ['.bmp'],
  'image/webp': ['.webp'],
  'image/svg+xml': ['.svg'],
}
```

### 6. [`env.example`](env.example) — Document New Variables

**Add section:**
```env
# ===== Multimodal / RAGAnything Configuration =====
# Enable multimodal document processing (requires raganything package)
# ENABLE_MULTIMODAL=false

# Vision Language Model binding (openai, ollama, gemini)
# VLM_BINDING=openai

# Vision Language Model name
# VLM_MODEL=gpt-4o

# VLM API host (defaults to LLM_BINDING_HOST if not set)
# VLM_BINDING_HOST=

# VLM API key (defaults to LLM_BINDING_API_KEY if not set)
# VLM_BINDING_API_KEY=

# Output directory for multimodal processing artifacts
# MULTIMODAL_OUTPUT_DIR=./multimodal_output
```

### 7. [`pyproject.toml`](pyproject.toml) — Add Optional Dependency Group

**Add new optional dependency group:**
```toml
[project.optional-dependencies]
multimodal = [
    "raganything[all]",
]
```

Also add to the `api` extras or create a combined `api-multimodal` extra.

---

## Implementation Steps (Ordered)

1. **Add multimodal config args to [`config.py`](lightrag/api/config.py)**
   - New argparse arguments with env var fallbacks
   - No functional change to existing behavior

2. **Add `raganything` optional dependency to [`pyproject.toml`](pyproject.toml)**
   - New `multimodal` extras group

3. **Add multimodal env vars to [`env.example`](env.example)**
   - Document all new variables with comments

4. **Create vision model function factory in [`lightrag_server.py`](lightrag/api/lightrag_server.py)**
   - `create_vision_model_func()` — mirrors `create_llm_model_func()` but for VLM
   - Supports openai, ollama, gemini bindings with image_data parameter

5. **Initialize RAGAnything wrapper in [`lightrag_server.py`](lightrag/api/lightrag_server.py)**
   - After `rag = LightRAG(...)` block
   - Conditional on `args.enable_multimodal`
   - Graceful error if `raganything` package not installed

6. **Extend [`DocumentManager`](lightrag/api/routers/document_routes.py:763) with image extensions**
   - Add `enable_multimodal` parameter
   - Conditionally include image file extensions

7. **Add image processing branch in [`pipeline_enqueue_file()`](lightrag/api/routers/document_routes.py:1194)**
   - New `case` in the `match ext:` block for image types
   - Route to RAGAnything multimodal processing

8. **Pass `rag_anything` to route factories**
   - Update [`create_document_routes()`](lightrag/api/routers/document_routes.py:2042) signature
   - Update [`create_query_routes()`](lightrag/api/routers/query_routes.py:193) signature

9. **Add `multimodal_content` field to [`QueryRequest`](lightrag/api/routers/query_routes.py:16)**
   - Optional field, no impact when absent

10. **Add multimodal query path in [`query_text()`](lightrag/api/routers/query_routes.py:325)**
    - Branch on `multimodal_content` presence
    - Call `rag_anything.aquery_with_multimodal()` when appropriate

11. **Update frontend [`constants.ts`](lightrag_webui/src/lib/constants.ts:42)**
    - Add image MIME types to `supportedFileTypes`

12. **Test the integration**
    - Verify text-only upload still works with `ENABLE_MULTIMODAL=false`
    - Verify image upload works with `ENABLE_MULTIMODAL=true`
    - Verify multimodal query works
    - Verify frontend accepts image files

---

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| Breaking existing text-only RAG | Feature-flagged behind `ENABLE_MULTIMODAL`; all changes are additive |
| `raganything` package not installed | Graceful import check with clear error message at startup |
| VLM API failures | Separate timeout/error handling for VLM calls; text-only fallback |
| Large image uploads | Existing `MAX_UPLOAD_SIZE` limit applies; image-specific size guidance in docs |
| Frontend changes break build | Image types are additive to the `supportedFileTypes` object; no removal |

---

## Testing Strategy

1. **Unit Tests**: Test `DocumentManager.is_supported_file()` with image extensions
2. **Integration Tests**: Upload an image file via `/documents/upload` with multimodal enabled
3. **Query Tests**: Send a query with `multimodal_content` field populated
4. **Regression Tests**: Run existing test suite with `ENABLE_MULTIMODAL=false` to confirm no breakage
5. **Frontend Tests**: Verify file picker accepts image files in the WebUI
