---
name: academic-pdf-zh
description: Translate an English-language academic PDF (or a link to one) into a Chinese version while preserving the original layout as closely as possible. Deliver an openable translated PDF; never paste the translation into the chat.
---
# English academic PDF → Chinese

Translate an English academic paper PDF into a Chinese PDF whose layout stays as close to the original as possible. Translate readable text only.

This skill must never contain secrets, tokens, accounts, key-file paths, or any other credentials. Read keys only from a secure location already present on the machine, or from the user at the moment of use; never put them on the command line, in the chat, in logs, in attachments, or in this skill. Pass them to the translation tool in a form that tool will actually read: if environment variables are ignored, write the key into a tightly permissioned private config file (e.g. `chmod 600`) and pass it with the tool's config flag (e.g. `-c`). Do not keep that file in a repository or any working directory that might be committed, and do not print or attach it.

## Defaults

- Once the user sends a PDF or a link, start immediately — do not open with a pile of questions.
- Produce a Chinese-only version by default; produce a bilingual version only when the user explicitly asks for one.
- Always deliver an openable PDF attachment; never paste the translation into the chat.
- Name files `original-title-zh.pdf`, not `-中文` or `-zh-CN`.
- Report the result concisely in Chinese; handle only the one paper the user handed over, never go looking for new papers.
- Keep figures, algorithm boxes and code listings in their original English by default (see "Figures and algorithm boxes"); translate only captions and body text.
- For long jobs, say up front roughly how long it will take, then post one line when each stage (translation, patching, checks) finishes, so the user does not have to keep asking for progress.
- Translate locally. Do not upload the paper to unknown third-party sites.

## Toolchain (how it is actually done)

Main tools: BabelDOC (layout-preserving translation), PyMuPDF (post-translation patching), poppler (`pdfinfo` / `pdftotext` / `pdftoppm`, for spot checks). Install whatever is missing before running. If BabelDOC is unavailable, use pdf2zh or a local tool with equivalent capability; the steps below are unchanged.

### 1. Prepare

- Download the PDF or link the user sent into a local working directory, and keep an untouched English original for comparison and patching.
- Example install: `uv tool install --python 3.12 BabelDOC`; run `babeldoc --warmup` first if assets such as fonts need fetching.
- PyMuPDF can use the Python inside BabelDOC's virtual environment, or install `pymupdf` separately.

### 2. Layout-preserving translation with BabelDOC

Go through an OpenAI-compatible endpoint. Model and base URL come from whatever the user specifies or the machine already has configured; when something is missing, tell the user which kind of configuration is absent — do not guess or use credentials of unknown origin.

Confirm how the installed BabelDOC accepts input and credentials; do not copy stale flags. 0.7.x requires `--files` for the PDF (a positional path is rejected) and does not read `OPENAI_API_KEY` from the environment. Write the key into the private config file described above and pass it with `-c`; do not use `--openai-api-key` (it appears in the process list). If a later version starts honoring the environment variable, prefer that, and still never put the key on the command line.

Typical parameters for a Chinese-only version:

```bash
babeldoc --openai \
  --files "$INPUT_PDF" \
  -c "$BABELDOC_CONFIG" \
  --lang-in en --lang-out zh \
  --openai-base-url "$OPENAI_BASE_URL" \
  --openai-model "$OPENAI_MODEL" \
  --openai-thinking disabled \
  --watermark-output-mode no_watermark \
  --skip-scanned-detection \
  --primary-font-family serif \
  --no-dual \
  --qps 3 \
  --custom-system-prompt "Translate into accurate, fluent Simplified Chinese academic prose; keep terminology consistent throughout; keep proper nouns, abbreviations, formulas, code, citation markers and bibliographic reference information in the original language." \
  --output "$OUTDIR"
```

**Self-test the endpoint before the real run:** send a small piece of text to the same endpoint and model with `curl` or a short script, and confirm a non-empty translation comes back. Common traps:

- Some services require a `User-Agent` or an extra session header; without it they return 403/400 while the tool silently retries, which looks like a hang.
- A reasoning model with thinking enabled may put the translation in `reasoning_content` and leave `content` empty; disable thinking (e.g. `--openai-thinking disabled` or the service's own switch).
- Set timeouts for the whole job and for each request (e.g. 120 s per request). If wall-clock time looks long but the process elapsed time and the log are still advancing, the machine may have been paused — do not kill a translation that is still running. Only if the output directory stays empty and the log shows no progress should you check the endpoint instead of waiting.

Drop `--no-dual` if the user wants a bilingual version. Keep the unedited draft the tool produced, so patching can restart from it if something goes wrong.

### 3. Post-translation patching (PyMuPDF)

Fix typos, placeholders, layout breakage and mistranslations against the original. Common moves:

- Redact the wrong glyphs first, then write the correct Chinese back with `insert_text` / `insert_textbox`.
- When a list got collapsed into one paragraph, or references were mistranslated or squeezed together, paste the layout back from the English original. Method: copy that English page into a temporary PDF, erase everything outside the rectangle to keep and call `apply_redactions()`, then place it in with `show_pdf_page`. Never embed a whole English page, or text extraction will leak English that is not displayed.
- When inserting Chinese, confirm the font contains the glyph; some Chinese fonts lack certain math symbols (e.g. U+2217 ∗) — switch to a font that has them rather than leaving tofu boxes.
- After translation, scan for CJK Compatibility Ideographs (U+F900–U+FAFF). These are often mistaken for traditional or variant forms. NFKC-normalize them to standard Simplified Chinese and rewrite with a Chinese font that includes those standard glyphs (e.g. Source Han Serif CN); do not change the text without also changing the font.
- Subset fonts only once, after all edits are done (`subset_fonts()`, deflate / garbage collect on save). Do not insert new glyphs afterwards into the same already-subset font name.

### 3.5 Figures and algorithm boxes: restore the English original

BabelDOC sometimes translates text inside figures, algorithm boxes (Algorithm N) and code listings, and the result is often garbled; sometimes the artwork is already left in English. Do not paste every figure back from the English original by default — doing so often covers the Chinese caption.

- Inspect each figure, algorithm box and code listing first. If the internal text is still English and intact, leave it; do not overwrite it.
- Restore only those whose internal text was translated, garbled or whose layout broke. Use `get_image_rects`, clustering of `get_drawings`, and the position of the `Fig. N` caption to find the bounding box on the English original page. The box must cover all text and lines inside the figure but not the caption or body lines.
- On the translated page, cover the same region with a white rectangle first, then paste the English region back with the method above (`sanitize_clip` + `show_pdf_page`). Visually confirm the paste did not cover the caption or body text.
- Captions, table titles and in-text references outside the box stay in Chinese; the title, Input/Output and every step inside an algorithm box stay in English.
- In two-column papers cover only the column the item sits in; treat a figure spanning both columns as one piece.

### 3.6 Must-fix list and terminology consistency

After translation, always check and fix these places where BabelDOC regularly fails, instead of waiting for the spot check to find them:

- References translated or squeezed together: paste back from the English original.
- Function names, field names and identifiers in tables translated or garbled: paste back or restore the original.
- Lost indentation or monospace font in code listings: paste back from the original.
- Overlapping appendix or section headings; a damaged first-page author block, email addresses or footer. Keep names and email addresses in English; affiliation may be translated into Chinese.
- Broken table, figure or section references (e.g. "如 IV 所示" should be "如表 IV 所示").
- Stacked or mixed caption prefixes (e.g. 「Fig图 12」): normalize to 「图 12：…」 and 「表 1：…」.
- Words or acronyms split mid-token in tables or body text (e.g. `Wind-ows`, `空-ce`, `FV / M`): restore the intact identifier.
- Inconsistent terms: before translating, fix a glossary of key terms (e.g. keep "gadget" in English or use one fixed rendering); after translating, search the whole text with `pdftotext` for each term's variants and leftover English, and unify them before delivery.

### 3.7 Links and bookmarks: restore and translate

BabelDOC usually keeps named destinations and the outline structure, but it does **not** carry clickable link annotations into the translation, and bookmark titles stay in English. Handle this for every paper:

- Use PyMuPDF to count links (`get_links()`), named destinations (`resolve_names()`) and outline entries (`get_toc()`) in the English original and the translation to see what is missing.
- In-text citations `[n]`: per page, pair each linked English `[n]` with the same `[n]` in the translation (use a min-cost assignment by position when a number appears more than once), then add a link over the Chinese glyph box pointing at the original destination (e.g. `cite.xxx`). Do not invent links the English lacks (e.g. the `[4]` inside `[3]–[5]`).
- Cross-references: figure, table, section, algorithm, listing, equation and appendix references change wording in Chinese (e.g. 「第 II 节」, 「图 1」, 「表 IV」, 「算法 1」); search for the Chinese wording. Pair all English phrase instances (captions included) with Chinese ones first, then transfer only those that were linked in English, so a body link cannot snap onto a caption.
- Footnote marks and URLs/DOIs in references and appendices: transfer them too; build boxes per actual Chinese line when a URL wrapped.
- Links inside figure or algorithm regions pasted back from the English original can keep their original coordinates.
- No duplicate overlapping links; keep link boxes hidden and do not change body text or colour.
- Bookmarks: rename each title to the matching Chinese heading in the body (keep proper nouns and abbreviations; use 「参考文献」 for references). **Do not rebuild with `set_toc()`**: it drops named destinations and pages become -1. Walk the outline items and edit `/Title` in place with `xref_set_key` (UTF-16BE hex string), then verify each entry's level, page and destination are unchanged.
- When the English original has no links, do not invent links.
- When the English original has no bookmarks, still generate a Chinese outline from the body headings: find the abstract, section, subsection and references headings in the English original by font size, weight and numbering (e.g. 1, 2.1, 2.1.1), locate the matching Chinese headings in the translation, and write levels, Chinese titles and `/XYZ` destinations just above each heading with `set_toc()` (fine when the original has no named destinations). Do not rely on the reader auto-detecting headings; mixed fonts in the translation usually defeat it.

### 4. Spot checks before delivery

Check the page count with `pdfinfo`, look for untranslated passages, leftover long English sentences, tool placeholders and duplicated phrases with `pdftotext -layout`, and render pages with `pdftoppm` for a visual comparison. At minimum check:

- Does the page count match
- Are figures, tables and formulas still there and intact
- Any large overlaps, overflows, garbled glyphs or missing translations
- Any leftover CJK Compatibility Ideographs
- Are indentation and monospace fonts preserved in code or lists
- Were references mistranslated or squeezed together
- Were garbled figures and algorithm boxes restored to the English original, without covering captions or body text; were already-English figures left untouched
- Any stacked prefixes such as 「Fig图」, or words and acronyms split mid-token in tables or body text
- Is terminology consistent throughout
- Does the link count match the English original, do all destinations resolve, and do sampled citations land on the right entry
- Are bookmarks translated to Chinese with unchanged destinations; when the original had none, does the generated Chinese outline cover every section and jump to the right heading
- Obvious errors in the author block, footers, table headers, appendices

Do not force-fix minor issues that are already present in the English original (a URL that already overflowed, a caption that already lacked a period, a Type 3 font warning the original also triggers). Mention them at delivery if useful. Fix problems before delivering; if something cannot be fixed, say so plainly.

## Translation standards

- Accurate, fluent Simplified Chinese academic prose; terminology consistent within one paper. Follow the user's glossary strictly if one is provided.
- Keep proper nouns, abbreviations, formulas, code and citation markers in the original language.
- Keep reference entries (authors, titles, venues, years, links and other bibliographic data) in the original language; the section heading itself may become 「参考文献」.
- Translate all readable text: abstract, body, figure and table captions, list headings, footnotes, appendices.
- Keep the original clickable links (citations, cross-references, URLs); translate bookmark titles to Chinese; when the original has no bookmarks, generate a Chinese outline from the section headings.
- Keep figures, drawings, diagrams and the text inside them as the original English figure; do not redraw or replace them. Algorithm boxes and code listings also stay in English; translate only captions, table titles and in-text references.

## Delivery

Attach the PDF and, in one or two sentences, state whether it is Chinese-only or bilingual and what deserves attention (e.g. occasional overflow in tight layouts, columns with tight spacing).
