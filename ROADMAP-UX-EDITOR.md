# Roadmap — ux-editor bridges

**Date:** 2026-05-19
**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](docs/superpowers/specs/2026-05-19-ux-editor-design.md)
**Decision log:** [`DECISIONS-UX-EDITOR.md`](DECISIONS-UX-EDITOR.md)

## Bridge catalogue

| # | Bridge | Format | License | Bundle (~) | Version | Notes |
|---|---|---|---|---|---|---|
| 1 | **EditorJS**  | Blocks | Apache-2.0       | 80 KB           | **v1.0** | Smallest, modular, easiest baseline |
| 2 | **CKEditor 5** | Html  | GPL/commercial   | 500 KB          | **v1.0** | Needs `licenseKey: 'GPL'` since v44 |
| 3 | **GrapesJS**  | Page   | BSD-3            | 800 KB          | **v1.0** | Visual web builder, finalizes Page format abstraction |
| 4 | **Quill**     | Html   | BSD-3            | 200 KB          | v1.x | Modular, Delta format internally |
| 5 | **TinyMCE**   | Html   | GPLv2/commercial | 400 KB          | v1.x | Cloud + self-hosted modes |
| 6 | **TipTap**    | Html   | MIT              | 80 KB core      | v1.x | Headless, ProseMirror-based |
| 7 | **BlockNote** | Blocks | MPL-2.0          | 250 KB          | v1.x | Notion-style, React-flavored core |
| 8 | **VvvebJS**   | Page   | Apache-2.0       | 500 KB +jQuery+BS | **v2.0** | Heaviest deps, deferred |

## v1.0 — abstraction validation release

**Goal:** prove the polymorphic content model end-to-end against one bridge per content format.

**Bridges:** EditorJS + CKEditor + GrapesJS.
**Implementation order:** EditorJS → CKEditor → GrapesJS.

**Rationale:**
- EditorJS first: small surface, JSON output, exercises `BlockContent` + `BlockRendererRegistry` + Block format abstracts. Cheapest to discover Tier 0/1 gaps.
- CKEditor second: largest WYSIWYG, stresses capability flags, license-key handling, common-options translation. Exercises `HtmlContent` + sanitizer + paste hook path.
- GrapesJS third: only Page-format bridge in v1. Forces `PageContent` shape (`html` + `css` + `components` + `assets`), asset extractor, sandbox renderer to finalize before more bridges layer on.

**Deliverables in v1:**
- `symfony/ux-editor` core package (Tier 0 + Tier 1).
- `symfony/ux-editor-editorjs`, `symfony/ux-editor-ckeditor`, `symfony/ux-editor-grapesjs` (Tier 2).
- npm sub-packages mirroring the above.
- `splitsh.json` updated for the three bridges.
- Demo routes `/editor/wysiwyg`, `/editor/block`, `/editor/page`, `/editor/live` on `ux.symfony.com`.
- Documentation page per bridge + one umbrella doc.
- Playwright projects: `ckeditor`, `editorjs`, `grapesjs`, `live`.

## v1.x — wide WYSIWYG + second block editor

Added incrementally, each as its own composer/npm sub-package:

1. **Quill** — second WYSIWYG, validates that `HtmlContent` round-trips losslessly across two bridges; first cross-bridge converter shipped (identity for Html pairs).
2. **TipTap** — third WYSIWYG, headless; stresses default config translation since TipTap has no native toolbar concept (capabilities adjusted via `WysiwygCapabilities::default()->with(...)`).
3. **TinyMCE** — fourth WYSIWYG; tests commercial-license path + cloud-vs-self-host config branch.
4. **BlockNote** — second Block bridge; first cross-bridge Block converter (EditorJS ↔ BlockNote block vocabulary map).

Each addition validates:
- Tier 1 abstract holds up without change.
- Tier 2 bridge ships in ~150 LOC PHP + ~50 LOC TS.
- `controllers.json` auto-registration works in both Encore and AssetMapper apps.

## v2.0 — page builder coverage

**VvvebJS.** Deferred because:
- Bundles jQuery + Bootstrap as upstream deps — heaviest peer-dep tree of any bridge.
- Component DSL differs significantly from GrapesJS; may surface `PageContent` extension needs (subclassing or component-tree variants).
- Lowest immediate user demand (GrapesJS already covers visual web building).

Lands after v1.x stabilizes the Block bridge converter pattern (whose lessons inform Page converter design between GrapesJS and VvvebJS).

## Out of v1/v2 scope

- Real-time collaborative editing (Y.js, ProseMirror sync).
- Server-side editor UI rendering.
- Format conversion across families (Html ↔ Blocks ↔ Page) — only same-family conversion shipped.
- Built-in retry for autosave/upload — app concern, surfaced via events.

## Future bridge candidates (post-v2)

Tracked separately, not committed:
- **Lexical** (Facebook) — Html, framework-agnostic React-leaning.
- **ProseMirror** — low-level WYSIWYG primitives.
- **Plate** — React WYSIWYG.
- **Toast UI Editor** — Markdown + WYSIWYG hybrid (would force new `MarkdownContent` value object — proves the polymorphic model's extension story).

Adding any of the above remains additive: new Tier 2 bridge directory only, no change to Tier 0/1.
