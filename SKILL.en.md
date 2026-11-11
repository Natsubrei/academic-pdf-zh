---
name: academic-pdf-zh
description: Translate an English-language academic PDF (or a link to one) into a Chinese version while preserving the original layout as closely as possible. Deliver an openable translated PDF; never paste the translation into the chat.
---
# English academic PDF → Chinese

Translate an English academic paper PDF into a Chinese PDF whose layout stays as close to the original as possible. Translate readable text only.

This skill must never contain secrets, tokens, accounts, key-file paths, or any other credentials. Read keys only from a secure location already present on the machine, or from the user at the moment of use; pass them to tools via environment variables, never on the command line, in the chat, in logs, or in this skill.

## Defaults

- Once the user sends a PDF or a link, start immediately — do not open with a pile of questions.
- Produce a Chinese-only version by default; produce a bilingual version only when the user explicitly asks for one.
- Always deliver an openable PDF attachment; never paste the translation into the chat.
- Name files `original-title-zh.pdf`, not `-中文` or `-zh-CN`.
- Report the result concisely in Chinese; handle only the one paper the user handed over, never go looking for new papers.
- Translate locally. Do not upload the paper to unknown third-party sites.

## Toolchain (how it is actually done)

Main tools: BabelDOC (layout-preserving translation), PyMuPDF (post-translation patching), poppler (`pdfinfo` / `pdftotext` / `pdftoppm`, for spot checks). Install whatever is missing before running. If BabelDOC is unavailable, use pdf2zh or a local tool with equivalent capability; the steps below are unchanged.

### 1. Prepare

- Download the PDF or link the user sent into a local working directory, and keep an untouched English original for comparison and patching.
- Example install: `uv tool install --python 3.12 BabelDOC`; run `babeldoc --warmup` first if assets such as fonts need fetching.
- PyMuPDF can use the Python inside BabelDOC's virtual environment, or install `pymupdf` separately.

### 2. Layout-preserving translation with BabelDOC

Go through an OpenAI-compatible endpoint. Model and base URL come from whatever the user specifies or the machine already has configured; when something is missing, tell the user which kind of configuration is absent — do not guess or use credentials of unknown origin.

Typical parameters for a Chinese-only version:

```bash
babeldoc --openai \
  --lang-in en --lang-out zh \
  --openai-base-url "$OPENAI_BASE_URL" \
  --openai-model "$OPENAI_MODEL" \
  --watermark-output-mode no_watermark \
  --skip-scanned-detection \
  --primary-font-family serif \
  --no-dual \
  --qps 3 \
  --custom-system-prompt "Translate into accurate, fluent Simplified Chinese academic prose; keep terminology consistent throughout; keep proper nouns, abbreviations, formulas, code, citation markers and bibliographic reference information in the original language." \
  --output "$OUTDIR" \
  "$INPUT_PDF"
```

Pass the key via an environment variable (e.g. `OPENAI_API_KEY`), not `--openai-api-key`. Drop `--no-dual` if the user wants a bilingual version. Keep the unedited draft the tool produced, so patching can restart from it if something goes wrong.

### 3. Post-translation patching (PyMuPDF)

Fix typos, placeholders, layout breakage and mistranslations against the original. Common moves:

- Redact the wrong glyphs first, then write the correct Chinese back with `insert_text` / `insert_textbox`.
- When a list got collapsed into one paragraph, or references were mistranslated or squeezed together, paste the layout back from the English original. Method: copy that English page into a temporary PDF, erase everything outside the rectangle to keep and call `apply_redactions()`, then place it in with `show_pdf_page`. Never embed a whole English page, or text extraction will leak English that is not displayed.
- When inserting Chinese, confirm the font contains the glyph; some Chinese fonts lack certain math symbols (e.g. U+2217 ∗) — switch to a font that has them rather than leaving tofu boxes.
- Subset fonts only once, after all edits are done (`subset_fonts()`, deflate / garbage collect on save). Do not insert new glyphs afterwards into the same already-subset font name.

### 4. Spot checks before delivery

Check the page count with `pdfinfo`, look for untranslated passages, leftover long English sentences, tool placeholders and duplicated phrases with `pdftotext -layout`, and render pages with `pdftoppm` for a visual comparison. At minimum check:

- Does the page count match
- Are figures, tables and formulas still there and intact
- Any large overlaps, overflows, garbled glyphs or missing translations
- Are indentation and monospace fonts preserved in code or lists
- Were references mistranslated or squeezed together
- Obvious errors in the author block, footers, table headers, appendices

Fix problems before delivering; if something cannot be fixed, say so plainly.

## Translation standards

- Accurate, fluent Simplified Chinese academic prose; terminology consistent within one paper. Follow the user's glossary strictly if one is provided.
- Keep proper nouns, abbreviations, formulas, code and citation markers in the original language.
- Keep reference entries (authors, titles, venues, years, links and other bibliographic data) in the original language; the section heading itself may become 「参考文献」.
- Translate all readable text: abstract, body, figure and table captions, list headings, footnotes, appendices.
- Do not redraw or replace images, drawings or diagrams. If text baked into a figure cannot be extracted reliably, keep the original figure and say so on delivery.

## Delivery

Attach the PDF and, in one or two sentences, state whether it is Chinese-only or bilingual and what deserves attention (e.g. untranslated text inside figures, occasional overflow in tight layouts).
