# Product Requirements Document (PRD)

## 1. Objective
Implement a 100% client-side React scanner to block Customer Identifying Data (CID) uploads before reaching the Green Zone, without server-side processing.

## 2. Scope & Constraints
- **Supported Formats:** PDF (digital), Excel (.xlsx/.csv), Word (.docx), Outlook (.msg/.eml).
- **Limit:** Maximum 5 MB per file.
- **Environment:** Citrix VM browser. Server-side scanning is non-compliant and prohibited.

## 3. Functional Requirements
- **FR1 (Parsing):** Application must parse files locally using Web Workers (e.g., `pdfjs-dist`, `mammoth.js`) to prevent main thread UI freezing.
- **FR2 (Scanning):** Application must evaluate extracted text against predefined CID regex patterns.
- **FR3 (Blocking):** If CID is detected, the upload is blocked locally, and redacted evidence is displayed to the user.
- **FR4 (Proof of Scan):** If clean, the client generates a SHA-256 hash of the file and requests a 30-second single-use upload token from the .NET API.

---

# Threat Model Document

## 1. Context & Boundaries
Data flow crosses from an untrusted client environment (React UI in Citrix) to a high-trust backend (.NET API / SQL Database).

## 2. STRIDE Analysis
- **Spoofing (Token Forgery):** Mitigated by tying short-lived (30s) upload tokens to the active, authenticated user session.
- **Tampering (In-flight File Swap):** Mitigated by the .NET server rejecting any upload where the file's payload does not match the client-provided SHA-256 "Proof of Scan" hash.
- **Repudiation (Upload Denial):** Mitigated by server-side logging of all token requests and hash matches.
- **Information Disclosure (Token Leak):** Mitigated by strict CORS and `Sec-Fetch-Site: same-origin` header enforcement, preventing cross-origin token extraction.
- **Denial of Service (Browser Crash):** Mitigated by the hard 5 MB limit and background Web Worker processing.
- **Elevation of Privilege (Raw API Bypass):** *Accepted Residual Risk.* A technical insider can theoretically extract tokens and forge a clean hash via Postman. This is mitigated primarily by Citrix VDI lockdown policies restricting developer tools.