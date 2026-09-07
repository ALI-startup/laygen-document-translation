# laygen-document-translation

An [Agent Skill](https://skills.sh) for translating DOCX, PPTX, XLSX, HWPX, CSV and TXT
files with their layout intact.

```bash
npx skills add ALI-startup/laygen-document-translation
```

Add `-g` to install for every project, or `-a claude-code` to skip the agent prompt.

Then just ask — *"translate report.docx into Korean"*. The agent installs the
[`laygen`](https://pypi.org/project/laygen/) CLI on first use, extracts the text segments,
translates them itself, and rebuilds the document. No translation API is involved.

PDF, legacy `.doc` / `.hwp` / `.xls` and images are not supported here — convert them
at [ondevice.neoali.com/laygen](https://ondevice.neoali.com/laygen) first.

## License

Apache-2.0 — Copyright 2026 NEOALI CO., LTD. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
