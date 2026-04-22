---
name: markitdown-convert
description: Convert source documents to clean Markdown by delegating to the markitdown-converter agent. Supported sources include PDF, Word (.docx), PowerPoint (.pptx), Excel (.xlsx), HTML, EPUB, images (.png/.jpg/.jpeg/.webp/.gif/.bmp), audio (.mp3/.wav/.m4a), and data files (.csv/.json/.xml) — single files or whole folders. Trigger this skill WHENEVER the user shares or references such a source and wants to read it, extract it, parse it, dump it, turn it into markdown/text, or make it LLM-friendly — even if they never say the word "markitdown" or "convert". Typical cues include "can you read this PDF", "extract the text from these slides", "parse this spreadsheet", "turn these docs into markdown", "I dragged a folder of reports, summarize them", or simply attaching/mentioning a file of one of the supported types together with any comprehension task. Do NOT attempt the conversion yourself with Read/pdftotext/unzip — always delegate to the `markitdown-converter` agent so caching, image extraction, and semantic image-to-markdown enrichment all work correctly.
---

# markitdown-convert

This is a **dispatcher skill**. Your only job is to recognize that the user's request is a markitdown conversion task, resolve the source path, and hand off to the `markitdown-converter` agent. The agent handles everything else: dependency bootstrap, caching via `convertinfo.json`, image extraction, image-to-markdown enrichment, and folder-tree mirroring.

## When this fires

The trigger is the combination of **(a) a supported source** and **(b) any request that implies the user wants the content in markdown/text form**. The request does not need to say "markdown" — "read", "extract", "parse", "summarise", "make it usable", "ingest into Claude", "show me what's inside" all count once a supported source is in play.

Supported extensions (from the agent's folder-mode glob): `.pdf`, `.docx`, `.pptx`, `.xlsx`, `.html`, `.htm`, `.epub`, `.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`, `.bmp`, `.mp3`, `.wav`, `.m4a`, `.csv`, `.json`, `.xml`. If the source is a folder, trigger if it plausibly contains any of the above — you do not need to enumerate first.

If the user only provides a URL (no local path), do not trigger — tell them to download it locally first, because `markitdown-converter` expects an absolute filesystem path.

## How to delegate

1. **Resolve to an absolute path** before calling the agent. Expand `~`, resolve relative paths against the current working directory, and on Windows pass a normal path (the agent normalises slashes internally). Do **not** quote-wrap the path inside the prompt — just include it as text.
2. **Decide the output folder.**
   - If the user named one ("put the markdown in `./out`"), pass it explicitly.
   - Otherwise, say nothing about `out` and let the agent default to `<cwd>/tempMarkdown`. Do not pre-create the folder yourself.
3. **Invoke the agent via the Task tool** with `subagent_type: "markitdown-converter"`. Use a single, focused prompt that names the source (and optional out) and otherwise trusts the agent. Example:

   ```
   Convert this source to markdown:
   source: C:/Users/c0917/Downloads/2025-annual-report.pdf
   ```

   Or with a custom output and a folder:

   ```
   Convert this folder to markdown:
   source: C:/Users/c0917/Documents/client-docs
   out:    C:/Users/c0917/Documents/client-docs-md
   ```

4. **Relay the agent's summary back to the user** — total files converted vs. cached, output folder, and `convertinfo.json` path. Don't reformat or re-narrate it; the agent's summary is the answer.

## What NOT to do

- Don't open the source with `Read` and paste contents. Binary formats (PDF, docx, pptx, xlsx, images, audio) cannot be meaningfully read that way, and for the text formats you'd be duplicating work the agent caches.
- Don't run `markitdown`, `pdftotext`, `unzip`, `pymupdf`, or any extraction yourself. The agent auto-installs its deps and caches by sha256 so repeated conversions of the same file are free — bypassing it loses that.
- Don't pre-create `tempMarkdown/` or `assets/` folders. The agent owns that tree.
- Don't pass multiple separate sources by calling the agent multiple times when they share a parent folder — pass the folder once and let the agent iterate. The shared `convertinfo.json` cache relies on one run seeing all files.
- Don't skip delegation just because the file "looks small". The cache + image enrichment are still worth it.

## Edge cases worth handling before delegating

- **User mentions a file that doesn't exist yet** (e.g., they're about to download one): confirm the path first, don't delegate on a phantom.
- **Extension not in the supported list** (e.g., `.rtf`, `.odt`): tell the user markitdown may or may not handle it; offer to try anyway by passing the path through, and warn that the agent will skip unsupported files in folder mode.
- **User pastes raw text instead of a file**: this skill doesn't apply. Handle inline.
- **Mixed ask** ("convert this PDF AND then summarise the result"): delegate the conversion first, then do the summary yourself on the produced `.md` file.
