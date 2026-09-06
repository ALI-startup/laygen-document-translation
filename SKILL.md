---
name: laygen-document-translation
description: Translate DOCX, PPTX, XLSX, HWPX, CSV or TXT files while preserving their layout. Use when asked to translate, localise or rewrite the text of a document file and return it in the same format. Extracts text segments to JSON, you translate them, then rebuilds the original document.
metadata:
  openclaw:
    emoji: 📄
    homepage: https://ondevice.neoali.com/laygen
    requires:
      bins: [laygen]
    install:
      - kind: uv
        package: laygen
        bins: [laygen]
---

# Translating a document with laygen

`laygen` splits a document into text segments, hands them to you as JSON, and puts your
translations back where they came from. It never calls a model — **you** do the
translating, between the commands.

## Install

```bash
uv tool install laygen        # once; or run it as: uvx laygen <args>
```

## The workflow

```bash
laygen extract report.docx -w work/
```

Extraction always splits the document into batches — `work/batches/batch_000.jsonl`,
`batch_001.jsonl`, … — capped at 32 segments and 4000 characters each so one fits
comfortably in context. Do not read `work/segments.json`; work batch by batch, whatever
the size of the document.

```bash
laygen status -w work/
# 1240 segments, 39 batches, 3 done, 36 outstanding
# next: work/batches/batch_003.jsonl
```

Translate the batch it names and write the result beside it as `batch_003.done.jsonl` —
the same records, `text` replaced by your translation. Check it before moving on:

```bash
laygen check -w work/ --batch 3 -t work/batches/batch_003.done.jsonl
```

`--batch N` scores against that batch alone, so the coverage figure means something and
ids belonging to a different batch are reported as errors — which is what saving a
translation under the wrong batch number looks like. Repeat until `status` shows none
outstanding, then:

```bash
laygen merge -w work/ work/batches/*.done.jsonl -o translated.jsonl
laygen reconstruct -w work/ -t translated.jsonl -o report.ko.docx
```

Finished batches stay on disk, so running out of context costs nothing: `laygen status`
says where to pick up. Never start over.

## Anything left untranslated

`merge` reports coverage and does not fail on gaps — missing segments keep their source
text. To close them:

```bash
laygen missing -w work/ -t translated.jsonl -o retry.jsonl
# translate retry.jsonl -> retry.done.jsonl
laygen merge -w work/ translated.jsonl retry.done.jsonl -o final.jsonl
```

## Write the file programmatically, not by hand

**This is the single most common way to break a run.** Translated text frequently
contains straight double quotes — rendering 「지재입국」 as `"Industrial Revolution"`, for
instance — and one unescaped `"` inside a value corrupts the record.

Generate the file with a real JSON serialiser:

```python
import json
with open("batch_000.done.jsonl", "w", encoding="utf-8") as f:
    for cid, text in translations.items():
        f.write(json.dumps({"id": cid, "text": text}, ensure_ascii=False) + "\n")
```

Never type raw JSON syntax by hand. JSONL limits the damage when it happens anyway — a
mangled line costs that one segment, which falls back to its source text, instead of
destroying the file — but generating it correctly costs nothing.

A single `.json` object mapping id to text also works, and is fine for short documents.
Be aware that one syntax error in that format loses every translation in the file.

## Rules

- **Never invent, drop or renumber ids.** Ids in your output must exist in the input.
  An unknown id fails the run.
- **An id you omit keeps its source text.** That is the correct way to leave something
  untranslated. Never emit an empty string for that — it is ignored and warned about,
  but the intent is unclear.
- **Preserve leading and trailing whitespace** exactly as it appears in the source. It
  carries real spacing in the document.
- **Do not translate** URLs, email addresses, file paths, code, or placeholders like
  `{name}`, `%s` or `<<FIELD>>`. Copy them through unchanged.
- **Never emit control characters.** They produce a file Office refuses to open; the
  tool rejects them.
- **Segments are independent.** Do not merge two into one or split one across ids.
- **Keep terminology consistent** across the whole document, including across batches.
- **One batch, one output file.** `batch_007.jsonl` becomes `batch_007.done.jsonl`,
  never any other number.

## Verifying your work

`laygen check -w work/ -t <file>` exits non-zero on errors and reports:

| Reported | Meaning |
|---|---|
| `N/M segments translated` | coverage — confirm it is what you expect |
| `unknown id(s)` | **error** — ids not in the source. Fix them. |
| `non-string value(s)` | **error** — a value was a number or object |
| `control characters` | **error** — would produce an unopenable file |
| `unparseable line(s), skipped` | warning — those segments keep their source text |
| `translated differently in more than one file` | warning — two files claim the same id. Usually a `.done` file saved under the wrong batch number. |
| `untranslated, source kept` | warning — expected only if deliberate |
| `identical to source` | warning — fine for dates and codes, suspicious in bulk |

## Supported inputs

`.docx` `.pptx` `.xlsx` `.hwpx` `.csv` `.txt` — run `laygen formats` to confirm. No PDF.

## When something goes wrong

| Message | Meaning |
|---|---|
| `Unsupported file type` | Not one of the six formats. PDF and legacy `.doc`/`.hwp` are out of scope. |
| `legacy binary or password-protected file` | Encrypted, or the wrong extension. Ask the user to unlock or re-save it. |
| `is not valid JSON` | Syntax error, with the line and nearby text. Regenerate with a serialiser; prefer `.jsonl`. |
| `Not a laygen workspace` | Wrong `-w` path, or extract was never run. |
| `refusing to reconstruct` | Errors listed above it must be fixed first. |
| `No such file or directory` | A path is wrong, or a `*.done.jsonl` glob matched nothing because no batch is finished yet. |
| `would be orphaned by re-splitting` | `laygen batches` would discard translations already done. Merge them first. |

## Limits worth stating to the user

- Formatting that varies **inside** one paragraph (a single bold word mid-sentence) is
  not preserved; the paragraph takes its first run's formatting. Paragraph and character
  styles, tables, images and layout are preserved.
- Do not edit the source document between extract and reconstruct — ids are positional.
- XLSX text is deduplicated, so a label repeated across many cells is one segment and is
  translated once everywhere.
