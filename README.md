================================================================================
       CardioTrack CT-200 QA Test Case Generator — Approach Summary
================================================================================
Author  : Kovarthini P
Email   : minekovarthini@gmail.com
Date    : July 17, 2026
Assignment: AI Engineering Internship — Tri9T AI

--------------------------------------------------------------------------------
EXECUTIVE SUMMARY
--------------------------------------------------------------------------------
A backend API for the CardioTrack CT-200 manual that parses the PDF into a
versioned hierarchical tree, supports re-ingestion of new versions without
destroying old ones, lets users pin node selections and generate QA test
cases via Google Gemini, and detects when generated test cases go stale as
source text changes.

Stack: FastAPI + Pydantic, SQLAlchemy/SQLite, TinyDB, PyMuPDF (fitz),
Google Gemini Flash, pytest

--------------------------------------------------------------------------------
PARSING & HIERARCHY RECONSTRUCTION
--------------------------------------------------------------------------------
PyMuPDF was chosen over pdfplumber/OCR/LLM approaches for its span-level
font metadata (size, weight, position), speed, and deterministic behavior —
the source PDF is text-based, so OCR isn't needed.

  - Heading detection: font-size thresholds calibrated to the CT-200 doc
    map spans to Title/H1/H2/H3/Body.
  - Tree build: stack-based construction — pop to the right parent level,
    push new heading, attach body text to the most recent heading.
  - Post-processing: merges fragmented text, normalizes whitespace,
    preserves tables, computes content_hash = SHA-256(heading + body).

--------------------------------------------------------------------------------
STRUCTURAL EDGE CASES HANDLED
--------------------------------------------------------------------------------
  - Duplicate headings — disambiguated via UUID + materialized path
    (e.g. "Safety > Warnings").
  - Mis-parented body text — assigned by y-coordinate/document order;
    orphan text becomes a synthetic "Preamble" node.
  - Multi-line headings — consecutive same-size spans within 2pt merged
    into one heading.
  - Headers/footers — repeating text in top/bottom 50pt margins excluded.
  - Tables — PyMuPDF table detection with column-alignment fallback;
    stored as pipe-delimited text.
  - Figures — image blocks captured with adjacent caption as
    "[Figure: caption]".

--------------------------------------------------------------------------------
DATA MODEL
--------------------------------------------------------------------------------
Relational data (documents, versions, nodes, selections, generations)
lives in SQLite via SQLAlchemy for joins and integrity. LLM-generated test
cases live in TinyDB as schema-less JSON, since output shape varies per
generation. Production would swap TinyDB for MongoDB Atlas.

Key tables: documents, document_versions, nodes (with logical_node_id,
content_hash, path), selections/selection_items, generations.

--------------------------------------------------------------------------------
VERSIONING & MATCHING STRATEGY
--------------------------------------------------------------------------------
New ingestions are matched against the prior version using a hybrid
approach: exact heading_path match first, then fuzzy matching (difflib
SequenceMatcher, threshold >= 0.85) among unmatched same-level nodes.
Matched nodes keep the same logical_node_id; content_hash comparison
flags whether they changed. Unmatched v1 nodes are treated as deleted.

Known failure modes: heading renamed + moved simultaneously breaks both
matching strategies; large structural reorganizations (merges/splits)
break the 1:1 matching model; whitespace-only edits still trigger a
changed-content flag.

--------------------------------------------------------------------------------
LLM PROMPT DESIGN
--------------------------------------------------------------------------------
Gemini Flash (gemini-2.0-flash) is prompted with a strict system role (QA
engineer, medical devices) and a JSON-only schema requiring 3-5 test
cases with title, preconditions, steps, expected_result, and priority.
Responses are validated against a Pydantic schema with a 3-step retry
ladder: strip markdown fences -> re-prompt with a stricter instruction ->
mark as failed and store the raw response. Re-generating the same
selection creates a new generation record rather than overwriting, since
LLM output is non-deterministic.

--------------------------------------------------------------------------------
STALENESS DETECTION
--------------------------------------------------------------------------------
At generation time, each source node's content_hash is snapshotted. At
retrieval time, the current hash for that node's logical_node_id is
compared; a mismatch marks the test case "stale" (content_changed or
node_deleted).

  - Limitation: binary hash comparison can't tell a typo fix from a
    changed safety threshold — no severity grading.
  - Limitation: formatting-only edits can still falsely trigger
    staleness despite whitespace normalization.
  - By design: the system only detects staleness; it never
    auto-regenerates test cases without human review.

--------------------------------------------------------------------------------
API OVERVIEW
--------------------------------------------------------------------------------
  Endpoint            Method     Purpose
  ------------------- ---------- -------------------------------------------
  /ingest              POST       Ingest a PDF (new or re-ingestion)
  /nodes               GET        List sections / get node with children
  /nodes/{id}/diff     GET        Check change across versions
  /selections          POST/GET   Create / list version-pinned selections
  /generate            POST       Generate test cases from a selection
  /test-cases          GET        Retrieve test cases with staleness info

--------------------------------------------------------------------------------
DECISION LOG — KEY HONEST ANSWERS
--------------------------------------------------------------------------------
Most likely silent failure: version-matching. A wrong fuzzy match
silently assigns the wrong logical_node_id, so staleness checks compare
against the wrong baseline. Mitigation: log confidence scores per match,
add a match-history endpoint, and flag matches with dramatic content_hash
shifts for review.

Simplicity over correctness: binary content-hash staleness. A typo and a
critical threshold change (e.g. 300mmHg -> 250mmHg) trigger the same
flag — risking alert fatigue and eroded trust in production. Fix would be
LLM-based severity classification or diff visualization.

Unhandled input: scanned/image-only PDFs. PyMuPDF returns empty text, so
ingestion "succeeds" with node_count: 0 instead of erroring. A real fix
would detect near-zero extraction and fall back to Tesseract OCR or
return an explicit warning. Out of scope per the assignment's instruction
not to build a generic parser.

--------------------------------------------------------------------------------
WHAT WOULD CHANGE WITH MORE TIME
--------------------------------------------------------------------------------
  - LLM-based semantic staleness (cosmetic vs. material changes)
  - Dedicated table extraction (camelot-py / tabula-py)
  - Confidence scores on version-matching
  - Webhook notifications for staleness after re-ingestion
  - MongoDB Atlas in place of TinyDB; caching for hot nodes/searches

================================================================================
                              END OF SUMMARY
================================================================================
