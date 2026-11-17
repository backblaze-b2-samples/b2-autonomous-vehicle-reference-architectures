# Contributing

The architecture catalog is the extension point. Edit `catalog/architectures.yaml`, then run `python3 tools/generate_catalog.py`; do not hand-edit generated README tables or indexes.

- New architecture: copy an existing `architectures/*/architecture.md` and name it by workflow, not tool. Keep its sections: workload, data flow (Mermaid), B2 zones, boundaries, B2 capability map, integration configuration, security, references. Add its catalog entry. Reuse `examples/` instead of duplicating configs.
- New integration: add it under the relevant catalog architecture with `connection` (exact settings), `connection_type`, authoritative documentation `url`, and, where one exists, a `setup_url` for the B2 setup guide. Connection types and their labels are shared across the B2 category repos (see `CONNECTION_LABELS` in `tools/generate_catalog.py`). Count a tool only when its docs or source show a configurable endpoint, a native B2 option, or a presigned-URL import. A tool shared by several architectures uses a YAML alias so every use is identical. Add an architecture only if the tool changes the data flow or a security boundary. If you add a new `connection_type`, add it to the schema enum and `CONNECTION_LABELS`; the enum must equal what the catalog uses.
- New sample application: build it in its own repository (for example under `backblaze-b2-samples`) and add a `samples:` entry with `group`, `summary`, `description`, and `architectures`. Confirm the repository is public.
- Regenerate `README.md`, `docs/related-b2-samples.md`, `docs/architecture-index.md`, `skill/references/architecture-catalog.md`, `llms.txt`, and `llms-full.txt` in the same change.

Content rules: attribute to a vendor only what its docs or code show; label B2-side patterns as such; never hardcode a region or endpoint; link B2 pricing instead of quoting numbers.
Security: no real keys anywhere; secrets in gitignored `0600` `.env`; scoped keys, not master, unless provisioning requires it (say so).
Tests: `python3 tools/generate_catalog.py --check`, `python3 tools/check_markdown_links.py`, and `python3 -m pytest tests`. Run `python3 tools/check_markdown_links.py --external` when adding outbound links.
