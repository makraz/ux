# Locked decisions — ux-editor

**Date:** 2026-05-19
**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](docs/superpowers/specs/2026-05-19-ux-editor-design.md)
**Brainstorm log:** [`BRAINSTORM-UX-EDITOR.md`](BRAINSTORM-UX-EDITOR.md)

One-line decision table. Source of truth for implementation phase.

| # | Decision area | Locked choice |
|---|---|---|
| 1 | Package shape | Single `symfony/ux-editor` core + 8 bridge sub-packages (composer + npm) |
| 2 | API surface | One `EditorType` form field covering all bridges |
| 3 | Content model | Polymorphic value objects (`HtmlContent`/`BlockContent`/`PageContent`) implementing `EditorContentInterface` |
| 4 | Persistence | Per-bridge `DataTransformer`; storage shapes `Scalar`/`Json`/`Split`; Doctrine custom Types + Embeddable both supported |
| 5 | Config shape | `CommonOptions` DTO + per-bridge typed `*Config` classes + `nativeOverrides` escape hatch; three input modes (config / preset / bridge+array) |
| 6 | Presets | Services tagged `ux.editor.preset`; app overrides win by priority |
| 7 | Capability guard | `BridgeCapabilities` flags; default warn, opt-in strict throw |
| 8 | Distribution | Per-bridge composer + npm sub-packages; `peerDependencies` for upstream libs; dynamic `import()` lazy load |
| 9 | Asset stacks | Both Encore and AssetMapper supported via shared `assets/controllers.json` + ESM dynamic imports |
| 10 | Sanitization | On submit by default for `HtmlContent`; per-block sanitizers for `BlockContent`; off for `PageContent` |
| 11 | Sanitizer service | Mandatory in prod (`editor.html.sanitize_required: true`); boot-time check |
| 12 | Upload pipeline | First-party `POST /_ux_editor/upload/{field}` + `EditorUploadHandlerInterface`; signed URLs + CSRF |
| 13 | LiveComponent | `data-live-ignore` on mount; hot config reload via `applyConfig(diff)`; remount on non-hot keys |
| 14 | Autosave | `LiveEditor` trait with debounced `#[LiveAction] saveDraft` + dirty flag UI |
| 15 | Abstraction depth | Three-tier stack: Tier 0 core / Tier 1 format abstracts in core / Tier 2 specific bridges |
| 16 | Tier 1 location | Lives inside `symfony/ux-editor` composer package, not separate |
| 17 | v1 bridges | EditorJS + CKEditor + GrapesJS — one per content format |
| 18 | v1 implementation order | EditorJS → CKEditor → GrapesJS |
| 19 | v1.x bridges | Quill, TinyMCE, TipTap, BlockNote |
| 20 | v2 bridges | VvvebJS |
| 21 | Testing — PHP | PHPUnit Tier 0/1 ≥ 90%, Tier 2 ≥ 75% |
| 22 | Testing — JS | Vitest ≥ 85% for controller abstracts + utilities |
| 23 | Testing — E2E | Playwright per bridge + cross-cutting Live specs; sharded as Playwright projects |
| 24 | Demo app | `ux.symfony.com` routes `/editor/{wysiwyg,block,page,live}` |
| 25 | Stimulus IDs | `symfony--ux-editor--{bridge}` |
| 26 | Exception hierarchy | Single `EditorExceptionInterface` marker; never silent-default on data |
| 27 | Form-layer errors | `TransformationFailedException` mapped to field errors |
| 28 | JS load failure | Degrade to plain textarea; never block form submit |
| 29 | Cross-bridge conversion | Opt-in via `ContentConverterRegistry`; default install ships identity converters only |
| 30 | Dev aids | WebProfiler collector + `bin/console debug:ux-editor` + visible warning boxes in debug mode |
