# HLD Functional Flow Diagram

[User Interaction]
  1. User selects file in React UI.
  2. React checks file size (<= 5MB).
     - If > 5MB: Reject immediately.
     - If <= 5MB: Proceed to step 3.
  3. React passes file buffer to Web Worker.

[Web Worker Processing]
  4. Web Worker executes parsing and CID Regex.
     - If CID detected: Return { status: 'CID_FOUND', snippet }.
     - If clean: Generate SHA-256 hash and return { status: 'CLEAN', hash }.

[Client-Server Handshake]
  5. React receives Web Worker payload.
     - On 'CID_FOUND': Display redaction modal, terminate flow.
     - On 'CLEAN': Request `/api/upload/init` for 30s token.
  6. .NET API returns `upload_token`.
  7. React submits file, token, and hash to `/api/upload/commit`.
  8. .NET API validates token and recalculates hash.
     - If match: Save to SQL DB.
     - If mismatch: Reject with 400 Bad Request.

---

# LLD Functional Flow Diagram (Web Worker Internal)

[Input Router]
  1. Receive `postMessage({ fileBuffer, mimeType })` from React.
  2. Route buffer based on `mimeType`:
     - `application/pdf` -> Initialize `pdfjs-dist` text layer extractor.
     - `application/vnd.openxmlformats...` (Word/Excel) -> Initialize `mammoth.js` / `xlsx`.
     - `application/vnd.ms-outlook` -> Initialize `@dlemstra/msg-reader`.

[Extraction & Scan Loop]
  3. Extract text chunk by chunk (or page by page).
  4. For each chunk:
     - Execute CID `RegExp` array.
     - If match found: Halt extraction, redact sensitive string, return `postMessage({ status: 'CID_FOUND', snippet })`.
  5. If all chunks parsed and no matches found: Proceed to hash.

[Hashing]
  6. Call `crypto.subtle.digest('SHA-256', fileBuffer)`.
  7. Convert buffer to hex string.
  8. Return `postMessage({ status: 'CLEAN', hash: hexString })`.

---

# Verification Trace

| Constraint | Verification in Final Phase | Status |
| :--- | :--- | :--- |
| **No server scanning** | HLD and LLD strictly isolate text extraction and regex mapping inside the Web Worker. | Pass |
| **Specific file types** | LLD Input Router explicitly handles MIME type routing for PDF, Word, Excel, and Outlook. | Pass |
| **5 MB limit** | HLD User Interaction layer drops oversized files before Worker instantiation. | Pass |
| **Block & Show Redacted** | Extraction loop halts immediately on match, returning snippet to React for UI blocking. | Pass |