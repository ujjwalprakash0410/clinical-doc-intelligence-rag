# 3. Document Processing

## Components

- `app/document_ai/parser.py`: `ClinicalDocumentParser`
- `app/document_ai/chunker.py`: `ProvenanceChunker`
- `app/models/schemas.py`: `EvidenceBlock` and `Chunk`

## Document identity

At parse time, the parser reads the complete file bytes and computes SHA-1, retaining the first 16 hexadecimal characters as `document_id`. Identical file contents therefore produce the same document ID regardless of filename. The SQLite documents table uses this ID as its primary key.

## Supported parsing paths

### PDF

For a PDF:
1. Open with PyMuPDF (`fitz`).
2. Iterate pages starting at page number 1.
3. Extract born-digital text via `page.get_text('text')`.
4. If OCR is enabled and extracted text length is below `min_pdf_text_chars` (default 80), rasterize that page at 180 DPI and run Tesseract OCR.
5. Open the PDF separately with pdfplumber.
6. Extract tables page by page, convert each row to a pipe-delimited string, and store non-empty tables as evidence blocks.
7. Return blocks, number of pages and table count.

A page can produce both a short/empty digital-text block (only when text exists) and an OCR block if the threshold triggers. Table blocks are separate from page text and OCR blocks.

### Image

For non-PDF files, the parser opens the file with Pillow and runs Tesseract if OCR is enabled. It creates one page-1 evidence block with type `ocr_text`. If OCR is disabled, the block may contain empty text and is later skipped by the chunker.

## EvidenceBlock model

Fields:
- `document_id`: content-derived document identifier.
- `page_number`: one-based source page.
- `block_id`: deterministic identifier derived from document, page, block kind and block sequence.
- `block_type`: `pdf_text`, `ocr_text` or `table`.
- `text`: extracted evidence content.
- `bbox`: optional bounding box (not populated by current parser).
- `confidence`: optional extraction confidence (not populated by current parser).

Block ID formats currently include `{document_id}_p{page}_text_1`, `..._ocr_1` and `..._table_{index}`.

## ProvenanceChunker

The chunker:
- Collapses runs of whitespace to a single space.
- Processes each evidence block independently; it does not combine adjacent blocks.
- Uses configured chunk size (default 900 characters) and overlap (default 150 characters).
- Advances by `max(1, chunk_size - overlap)`.
- Discards chunks whose non-whitespace content is shorter than 20 characters.
- Generates a 12-character SHA-1 digest from block ID, start offset and chunk text.
- Retains document ID, page number, source block ID and source type.

Chunk IDs are formed as `{document_id}_{digest}`.

## Why preserve blocks before chunks?

Keeping extracted blocks separate preserves page and content-kind provenance. A retrieved chunk can be traced to a particular page and source block, and the citation can refer back to the original document page.

## Current limitations visible in the implementation

- The parser does not currently populate bounding boxes.
- OCR confidence is not returned.
- PDF table extraction uses a simple row-to-text representation; it does not preserve cell-level coordinates or rich table structure.
- Parsing is not layout-tree aware beyond separating text, OCR and detected tables.
- No file-type/content validation or upload-size limit is visible in the ingestion route.
- Image parsing is not wrapped in a PDF/image-specific exception boundary in the parser.
