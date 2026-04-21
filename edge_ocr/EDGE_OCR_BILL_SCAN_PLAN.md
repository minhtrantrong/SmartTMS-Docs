# SmartTMS Edge OCR Bill Scan Plan

Date: 2026-04-21

## 1. Purpose

This document defines a practical implementation plan for adding edge bill-scan OCR without removing the current backend-managed OCR flow.

In this plan, edge execution means:

- OCR runs on `SmartTMS-Mobile` for the primary driver capture flow.
- OCR can optionally run in `SmartTMS-FE` for browser-based upload flows.
- `SmartTMS-BE` remains the system of record for shipment bills and persisted OCR payloads.
- The current backend OCR API remains available as a separate server-side scanning path.
- `SmartTMS-AI` remains available as a fallback and reprocessing service during rollout.

## 2. Current State Summary

The codebase today has a mixed OCR architecture:

- `SmartTMS-Mobile` calls `SmartTMS-AI` directly from `lib/services/ocr_service.dart`.
- The mobile app currently embeds the AI API key in client code, which is not acceptable for a long-term production design.
- `SmartTMS-FE` already sends bill uploads to `SmartTMS-BE`, and the backend calls `SmartTMS-AI` synchronously.
- `SmartTMS-BE/internal/handler/bill_handler.go` stores OCR output in `shipment_bills.ocr_data` as JSON.
- The existing backend OCR route should be preserved because it is still useful for admin or dispatcher workflows and can later support higher-volume server-managed scanning.
- The persisted OCR contract currently looks like:

```json
{
  "filename": "bill.jpg",
  "text": ["line 1", "line 2"]
}
```

- Existing docs already record that centralized OCR introduces high latency and timeout pressure because the request path is `FE -> proxy -> BE -> AI` and the OCR model can take 30 to 60+ seconds.

## 3. Why Edge OCR

Moving OCR to the client side solves the current bottlenecks more directly than adding more timeout padding.

Expected benefits:

- Removes the AI secret from the mobile client.
- Reduces network hops and avoids most OCR timeout failures.
- Improves bill-scan responsiveness for drivers.
- Allows partial offline behavior on mobile.
- Keeps raw image upload and persistence in the backend, so auditability is preserved.

Constraints that still apply:

- The backend must remain the only trusted write path for shipment bills.
- Client-generated OCR must be treated as assistive data, not authoritative financial truth.
- PDF handling is more expensive on the edge than image handling and should not block the first rollout.

## 4. Recommended Direction

### 4.1 Primary recommendation

Implement edge OCR on `SmartTMS-Mobile` first, then add optional browser OCR in `SmartTMS-FE`.

This is the recommended order because:

- The main capture workflow is already mobile-first.
- Camera capture quality and preprocessing are easier to control on-device than in the browser.
- Removing the direct `SmartTMS-Mobile -> SmartTMS-AI` dependency closes the most obvious security gap first.
- Browser OCR has larger variability in CPU, memory, and bundle-size cost.

### 4.2 Scope recommendation

For v1 of edge OCR:

- Support image-based bill scanning on mobile first.
- Keep backend upload mandatory even when OCR happens locally.
- Keep the current backend OCR API as a first-class server OCR route for admin and dispatcher use cases.
- Add a new dedicated endpoint for client-generated OCR payloads instead of changing the behavior of the old backend OCR route.
- Keep `SmartTMS-AI` as the fallback path for unsupported devices, low-confidence OCR, and PDF inputs.
- Treat browser OCR as a feature-flagged second phase, not as the initial default path.

### 4.3 API strategy recommendation

Use two separate shipment bill APIs.

- Existing backend OCR endpoint: preserved for server-managed OCR.
- New client OCR endpoint: used only when OCR has already run on mobile or FE.

This split keeps the contract clear:

- admin and dispatcher flows can keep using the backend-managed OCR path
- mobile edge OCR does not depend on backend OCR behavior
- future async or batch backend scanning can evolve independently from the client OCR contract

## 5. Target Architecture

### 5.1 High-level flows

#### Server OCR flow

1. Admin, dispatcher, or another backend-driven workflow uploads a bill image to the existing backend OCR endpoint.
2. `SmartTMS-BE` runs the current server-side OCR path.
3. `SmartTMS-BE` persists the bill and normalized OCR payload.
4. The response returns the saved bill with OCR data.

#### Edge OCR flow

1. User captures or selects a bill image.
2. Client preprocesses the image locally.
3. Client runs OCR locally.
4. Client shows OCR text or parsed fields for review.
5. Client uploads the original file plus OCR payload to a new client OCR endpoint in `SmartTMS-BE`.
6. Backend validates and normalizes the payload.
7. Backend persists the bill and returns the normalized OCR structure.
8. If local OCR fails or is unsupported, the client can fall back to the server OCR route.

### 5.2 System responsibilities

`SmartTMS-Mobile`

- Primary edge OCR runtime.
- Image capture and preprocessing.
- Local review UI before upload.
- Offline-first capture behavior where feasible.

`SmartTMS-FE`

- Optional browser OCR runtime for desktop uploads.
- Feature-flagged worker-based inference.
- Review and correction UI before upload.

`SmartTMS-BE`

- Two public bill ingestion endpoints: one for server OCR and one for client OCR.
- Validation and normalization of OCR payloads.
- Persistence of original file metadata and OCR data.
- Server-side OCR orchestration for the preserved backend route.
- Clear separation between backend OCR and edge OCR ingestion.

`SmartTMS-AI`

- Fallback OCR engine.
- Reprocessing path for low-confidence or unsupported uploads.
- Non-blocking migration bridge while edge OCR rolls out.

## 6. Engine Strategy

### 6.1 Mobile options

| Option | Pros | Risks | Recommendation |
| --- | --- | --- | --- |
| Google ML Kit text recognition | Native on-device inference, offline capable, fast startup, good Flutter support | OCR output differs from PaddleOCR, limited layout metadata | Recommended for mobile MVP |
| Tesseract via Flutter wrapper | Easier to reason about, browser parity possible | Usually slower and weaker on noisy camera images | Not recommended as the primary mobile path |
| Shared custom model via ONNX or TFLite | Better parity across platforms, more control | Highest engineering, packaging, and optimization cost | Defer unless ML Kit accuracy is unacceptable |

### 6.2 Frontend options

| Option | Pros | Risks | Recommendation |
| --- | --- | --- | --- |
| Tesseract.js in a web worker | Lowest integration cost, browser-only | Slower, weaker on complex bill images, larger client CPU hit | Acceptable for a quick spike only |
| ONNX Runtime Web with a quantized OCR model | Better control and better long-term parity | Bundle size, worker complexity, browser memory pressure | Preferred for FE if browser OCR becomes a real product requirement |

### 6.3 Decision rule

Use different engines per platform if that gets the team to production faster.

The shared contract matters more than a shared model binary.

## 7. Data Contract Plan

### 7.1 Preserve current compatibility

The existing `ocrData.text` array should remain available because:

- `SmartTMS-FE` already renders `ocrData.text` in the bill scan UI.
- `SmartTMS-BE` already stores OCR JSON in a flexible JSONB field.
- Preserving the existing shape avoids a wide refactor.

### 7.2 Extend the payload safely

Add optional metadata to the OCR JSON payload while keeping `filename` and `text` intact.

Recommended normalized shape:

```json
{
  "filename": "bill.jpg",
  "text": ["line 1", "line 2"],
  "source": "edge-mobile",
  "engine": "mlkit-text-recognition",
  "engineVersion": "v1",
  "durationMs": 1340,
  "confidence": 0.87,
  "languageHints": ["en", "vi"],
  "needsReview": false
}
```

Optional fields can be added without an immediate database migration if they remain inside the existing JSONB payload.

Only add schema columns later if reporting or filtering on OCR source, confidence, or review status becomes necessary.

### 7.3 Backend API change

Do not overload the current backend OCR endpoint with mixed behavior.

Preserve the existing server OCR API and add a second dedicated client OCR API.

Recommended endpoint split:

- `POST /api/shipments/{id}/bills`
  - Existing server-side OCR endpoint.
  - Request contains only the uploaded file.
  - Backend performs OCR using the current server-managed flow.
  - This route remains the preferred path for admin and dispatcher workflows that should stay backend-managed.

- `POST /api/shipments/{id}/bills/client-ocr`
  - New edge OCR endpoint.
  - Request contains the uploaded file and `ocr_payload`.
  - Backend validates, normalizes, and persists client-generated OCR data.
  - This route does not depend on backend OCR for the normal success path.

- `POST /api/bills/upload`
  - Keep the current generic backend OCR endpoint for backward compatibility and non-persisted scan flows if still needed.

Recommended request shape for the new client OCR endpoint:

- multipart form field `file`
- multipart form field `ocr_payload` containing JSON text

Backend behavior:

- The existing server OCR route keeps its current behavior.
- The new client OCR route requires valid `ocr_payload`, normalizes it, and persists it.
- If client OCR confidence is below the threshold, persist with `needsReview=true` and optionally trigger a later server-side reprocess flow.
- Fallback between client OCR and server OCR should be explicit in the application flow, not hidden behind one ambiguous API contract.

### 7.4 Later backend scaling option

The preserved server OCR route is the right place for later admin or dispatcher scaling work.

If high-volume scanning becomes a real workflow, a later phase can add:

- async OCR jobs behind the preserved backend route, or
- a separate batch-oriented admin API

That later optimization should not change the contract of the new client OCR endpoint.

## 8. Rollout Phases

### Phase 0: Benchmark and decision lock

Estimated duration: 4 to 5 working days

Tasks:

- Build a representative sample set of real bill images.
- Include easy, medium, and poor-quality samples.
- Compare current `SmartTMS-AI` output against candidate mobile and FE engines.
- Define acceptance targets for latency and text quality.
- Decide the confidence threshold and fallback rule.

Exit criteria:

- OCR engine choices are locked.
- Sample set is versioned.
- Acceptance metrics are agreed.

### Phase 1: Backend contract preparation

Estimated duration: 3 to 4 working days

Tasks:

- Preserve the current shipment bill server OCR route without changing its behavior.
- Add a new shipment bill client OCR route that accepts `ocr_payload`.
- Normalize incoming client OCR payloads into the existing `ocrData` JSON shape.
- Keep the current `SmartTMS-AI` call path on the preserved server OCR route.
- Update Swagger and shared frontend types.
- Add audit metadata for OCR source.

Exit criteria:

- Backend exposes two clear shipment bill ingestion paths: server OCR and client OCR.
- Existing frontend bill rendering still works.

### Phase 2: Mobile edge OCR MVP

Estimated duration: 1.5 to 2 weeks

Tasks:

- Replace the direct AI call in `SmartTMS-Mobile/lib/services/ocr_service.dart` with an engine abstraction.
- Implement a local OCR engine.
- Add preprocessing for resize, rotation, and contrast normalization.
- Update the scan screen so users can review OCR output before upload.
- Upload the original file and `ocr_payload` to the new client OCR backend endpoint.
- Remove the embedded AI secret from mobile code.
- Keep a fallback path for unsupported devices or low-confidence results.

Exit criteria:

- Mobile image scans work without requiring `SmartTMS-AI` in the normal path.
- OCR text is persisted through backend upload.
- No AI secret remains in shipped mobile code.

### Phase 3: Frontend browser OCR spike and MVP

Estimated duration: 1 to 1.5 weeks

Tasks:

- Add a worker-based OCR adapter in `SmartTMS-FE`.
- Feature-flag browser OCR.
- Run local OCR only for supported browsers and file types.
- Send browser-generated OCR to the new client OCR endpoint when local OCR succeeds.
- Keep the existing backend OCR route available for admin or dispatcher flows and for explicit fallback.

Exit criteria:

- Browser OCR works behind a feature flag for image uploads.
- Unsupported cases degrade cleanly to the fallback path.

### Phase 4: Hardening and rollout

Estimated duration: 1 week

Tasks:

- Add telemetry for OCR duration, fallback rate, and user corrections.
- Add discrepancy review for low-confidence or heavily edited OCR results.
- Tune confidence thresholds.
- Update docs and local-development instructions.
- Roll out mobile edge OCR first, while keeping the preserved backend OCR route available for admin and dispatcher users.

Exit criteria:

- Mobile edge OCR is the default path.
- Browser OCR is either approved for broader rollout or left behind a flag.

## 9. Repo-by-Repo Work Plan

### 9.1 SmartTMS-Mobile

Target files and modules:

- `lib/services/ocr_service.dart`
- `lib/features/shipments/screens/scan_bill_screen.dart`
- `lib/core/network/api_client.dart`

Planned changes:

- Introduce an OCR engine interface, for example `LocalOcrEngine` and `RemoteOcrFallbackEngine`.
- Send mobile edge OCR results to the new client OCR shipment bill endpoint.
- Remove the direct dependency on `SmartTMS-AI` from the default user path.
- Add user review and correction before final upload.
- Add image-quality checks before OCR runs.

### 9.2 SmartTMS-FE

Target files and modules:

- `src/app/driver/scan-bill/page.tsx`
- `src/lib/apiProxy.ts`
- `src/types/index.ts`

Planned changes:

- Add a browser OCR adapter and load it lazily.
- Use the new client OCR shipment bill endpoint when browser OCR succeeds.
- Keep the existing backend OCR route available for admin and dispatcher workflows that should remain server-managed.
- Extend `ShipmentBill.ocrData` typing with optional metadata fields.
- Show OCR review UI before upload.
- Preserve current rendering of `ocrData.text`.

### 9.3 SmartTMS-BE

Target files and modules:

- `internal/handler/bill_handler.go`
- `internal/routes/routes.go`
- `docs/swagger.yaml`
- `docs/swagger.json`
- `docs/docs.go`

Planned changes:

- Preserve the existing `/api/shipments/{id}/bills` route as the server OCR path.
- Add a new `/api/shipments/{id}/bills/client-ocr` route for client-generated OCR payloads.
- Normalize client OCR payloads into a single backend response shape.
- Keep server-side OCR for PDFs, explicit fallback, and admin or dispatcher server-managed flows.
- Add source metadata such as `edge-mobile`, `edge-web`, or `server-ai`.
- Keep the bill upload API backward compatible.
- Leave room for a later async or batch backend OCR enhancement without changing the client OCR route.

### 9.4 SmartTMS-AI

Target role after migration:

- fallback OCR
- reprocessing tool
- non-edge document support such as PDFs

Planned changes:

- Keep `/apis/ocr/extract` stable during migration.
- Stop treating the AI service as a mandatory synchronous dependency for mobile image scanning.
- Consider adding a dedicated reprocess endpoint later if review workflows require it.

### 9.5 SmartTMS-Docs

Planned changes:

- Update `MULTI_REPO_DEVELOPMENT_GUIDE.md` after the implementation lands.
- Update mobile and frontend setup docs to remove direct AI requirements from the primary flow.
- Document fallback rules and supported file types.

## 10. Non-Goals for the First Delivery

Do not include these items in the first edge OCR release:

- full semantic receipt parsing beyond line extraction
- browser OCR for PDFs
- a fully shared model binary across mobile and FE
- replacing the backend as the source of truth
- treating OCR results as final accounting values without review
- redesigning the preserved backend OCR route into a full async batch job system

## 11. Risks and Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Client OCR results differ from current AI output | Inconsistent saved text and user confusion | Use a normalized contract, maintain sample benchmarks, and track correction rate |
| Browser OCR is too slow on weak machines | Poor FE UX | Keep FE OCR feature-flagged, lazy-load the model, and fall back to backend OCR |
| Low-quality bill images cause poor edge OCR | Bad extraction quality | Add image-quality hints, preprocessing, and low-confidence fallback |
| Client payload can be tampered with | Trust and data quality concerns | Treat OCR as user-provided assistive data, keep original file, and require review for business-critical fields |
| PDF support blocks rollout | Schedule slip | Keep PDF OCR on the existing server-side fallback path for v1 |
| Mixed route behavior causes client confusion | Wrong endpoint usage and harder maintenance | Keep separate server OCR and client OCR endpoints with explicit UI-to-endpoint mapping |

## 12. Success Metrics

Minimum rollout targets:

- Mobile image OCR no longer requires `SmartTMS-AI` on the normal path.
- No AI secret is shipped in `SmartTMS-Mobile`.
- Existing bill display still works with `ocrData.text`.
- The preserved backend OCR route remains available for admin and dispatcher workflows.
- Mobile OCR p95 completes in under 5 seconds on the target test device.
- Browser OCR, if enabled, completes in under 8 seconds p95 on supported desktops.
- Fallback rate is low enough that `SmartTMS-AI` is no longer a mandatory always-on dependency for image bills.

## 13. Acceptance Criteria

- A driver can scan an image bill from `SmartTMS-Mobile` and get OCR text without a direct call to `SmartTMS-AI`.
- The original file and OCR payload are still saved through `SmartTMS-BE`.
- `SmartTMS-BE` keeps the current server OCR endpoint for backend-managed scans.
- `SmartTMS-BE` adds a separate client OCR endpoint for uploads with `ocr_payload`.
- `SmartTMS-FE` can continue displaying `ocrData.text` without breakage.
- PDF uploads still have a supported fallback path.
- OCR source metadata is persisted so rollout behavior can be observed.

## 14. Recommended First Implementation Slice

If the team wants the fastest path to value, start with this slice:

1. Keep the current shipment bill backend OCR route unchanged as the server OCR path.
2. Add a new shipment bill client OCR route for `file + ocr_payload`.
3. Mobile switches to on-device OCR for image bills and uploads to the new client OCR route.
4. Browser OCR stays deferred behind a later feature flag.

This slice removes the largest current problem first, the direct mobile dependency on the centralized AI OCR service, while preserving the backend OCR path for later admin and dispatcher workflows.