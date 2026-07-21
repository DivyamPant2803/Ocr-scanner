# API Security Specification

## 1. Token Handshake & Hashing Contract
- **Endpoint 1: `/api/upload/init` (GET)**
  - Authenticates the user session.
  - Issues a cryptographically secure, single-use `upload_token` valid for exactly 30 seconds.
- **Endpoint 2: `/api/upload/commit` (POST)**
  - **Inputs:** The file payload, the `upload_token`, and the client-generated `file_hash` (SHA-256).
  - **Validation Steps:**
    1. Verify `upload_token` is valid and unconsumed.
    2. Server independently recalculates the SHA-256 hash of the incoming file payload.
    3. Server strictly compares its hash against the client's `file_hash`. A mismatch triggers a 400 rejection.

## 2. Header & Transport Hardening
- **Strict Fetch Metadata Enforcement:** The server must reject any request to `/api/upload/*` missing `Sec-Fetch-Site: same-origin`. This neutralizes standard Postman/curl API bypass attempts and limits requests to the trusted Citrix browser origin.
- **MIME Type Validation:** The server must reject requests missing expected `Content-Type` headers.
- **Rate Limiting:** Enforce strict limits on `/init` to prevent token brute-forcing.

---

# Client-Side Low-Level Design (LLD)

## 1. Main Thread (React)
- **File Ingestion:** Intercepts the `onChange` event of `<input type="file" />`. Immediately rejects if `file.size > 5 * 1024 * 1024`.
- **Worker Instantiation:** Spawns `const worker = new Worker('/scanner.worker.js')`.
- **Message Passing:** Sends the raw `File` object or `ArrayBuffer` via `worker.postMessage()`.
- **UI State Handling:** Listens for `worker.onmessage`. On `{ status: 'CID_FOUND', snippet }`, mounts the Redaction Modal and blocks upload. On `{ status: 'CLEAN', hash }`, triggers the API Handshake.

## 2. Web Worker (`scanner.worker.js`)
- **PDF Parsing (`pdfjs-dist`):** Must load `pdf.worker.min.mjs` independently to run off the main thread. Extracts text layers sequentially; bypasses heavy canvas rendering entirely to save memory.
- **Other Parsers:** `mammoth.js` for DOCX, `xlsx` for Excel, `@dlemstra/msg-reader` for MSG. All execute synchronously inside the worker thread.
- **CID Regex Engine:** Executes predefined `RegExp` arrays against extracted strings. Terminates file parsing immediately upon the first match to save cycles.
- **Hashing (`Web Crypto API`):** If no CID is matched, executes `crypto.subtle.digest('SHA-256', fileBuffer)` and returns the hex string back to the Main Thread.

---

# Verification Trace

| Constraint | Verification in Phase 3 | Status |
| :--- | :--- | :--- |
| **No server scanning** | Server only calculates hash matching; absolutely no text extraction or regex processing occurs server-side. | Pass |
| **Web Worker isolation** | LLD explicitly offloads heavy parsing (e.g., `pdfjs-dist`) and hashing to `scanner.worker.js`. | Pass |
| **API Bypass Guardrail** | `Sec-Fetch-Site: same-origin` and 30s token + hash handshake explicitly defined to drop rogue API payloads. | Pass |