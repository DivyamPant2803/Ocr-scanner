# High-Level Architecture Diagram (Data Flow)

[User / Citrix VM] -> (File Selection < 5MB) -> [React UI]
       |
       v
[React Main Thread] --(Spawns)--> [Web Worker Pool]
                                       |
                                       |-- 1. [Parser Module]: Extracts text (pdfjs, mammoth, xlsx, msg-reader).
                                       |-- 2. [Regex Engine]: Scans text for CID patterns.
                                       |
                   (If CID Found) <----|
[React UI] <----------------------- [Redacted text snippet generated]
   |
   |--(Blocks Upload & Displays Redaction)
   
                   (If Clean) <--------|
[React UI] <----------------------- [Generates SHA-256 File Hash]
   |
   |--(1. Request Token + Hash)--> [.NET API (Green Zone)]
                                       |--(Generates 30s single-use token tied to hash)
   |
   |--(2. Submit File + Token)--> [.NET API (Green Zone)]
                                       |--(Validates Token + Recalculates Hash)
                                       |--(If Valid)--> [SQL Database]

---

# Client-Side High-Level Design (HLD)

## 1. UI Layer (React)
- **Upload Component:** Drag-and-drop zone with strict 5MB size limit and MIME-type validation.
- **Redaction Modal:** Displays the snippet returned by the Web Worker if CID is found. Prevents submission.

## 2. Processing Layer (Web Workers)
- **Worker Manager:** Handles messaging between the main thread and background workers.
- **Parsing Engines:** Browser-compatible libraries configured for local execution only.

## 3. Security & Network Layer
- **Hashing Module:** Web Crypto API (`crypto.subtle.digest`) to generate the SHA-256 hash post-scan.
- **API Client:** Configured to enforce `Sec-Fetch-Site: same-origin` headers and handle the two-step token/upload handshake.

---

# Verification Trace

| Constraint | Verification in Phase 2 | Status |
| :--- | :--- | :--- |
| **No server scanning** | Architecture routes scanning purely through Web Workers before API contact. | Pass |
| **Specific file types** | Parser Module allocated for PDF, Word, Excel, Msg. | Pass |
| **5 MB limit** | UI Layer strictly drops files > 5MB before passing to workers. | Pass |
| **Block & Show Redacted** | Redaction Modal intercepts UI flow if CID regex triggers. | Pass |
| **Citrix/Intranet** | Relies on standard browser APIs (Web Crypto, Web Workers) available in modern Citrix VDI browsers. | Pass |