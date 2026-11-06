# CLAUDE.md

This repository is a documentation-only reference collection for Wire-Cell Toolkit (WCT) algorithms and data models — no source code, no build/test process. See @./README.md for the directory layout and what each doc covers.

## Scope and usage

- These docs are consumed by other projects (e.g. `../wirecell-img-evaluation`) via symlinks into their own `docs/` directories, so a path like `wirecell-img-evaluation/docs/img_3d_imaging_workflow/` resolves here.
- Treat this repo as read reference material, not a place to run experiments or store project-specific results — those belong in the consuming project.
- Some docs quote project-specific example values (a particular detector's config, a source line number from a particular repo checkout) inline alongside the general algorithm explanation — see README's "주의" section. When editing, keep that distinction clear rather than presenting project-specific numbers as universal WCT constants.
- When editing a doc here, check whether other docs in this repo or in a consuming project link to it (`grep -rn` for the filename) before renaming or moving it, since links are plain relative markdown paths, not something a refactor tool tracks.
