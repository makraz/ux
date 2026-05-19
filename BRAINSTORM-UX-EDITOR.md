# Brainstorm history — ux-editor package

**Date:** 2026-05-19
**Branch:** feat/icons-documentation (brainstorm only, no code yet)
**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](docs/superpowers/specs/2026-05-19-ux-editor-design.md)

Full Q&A trace of the brainstorming session that produced the `ux-editor` design. Each question lists the options offered, the recommended option, and the user's final pick.

---

## Q1 — Scope of "editor" abstraction

Options:
- **A.** WYSIWYG-only (CKEditor, Quill, TinyMCE, TipTap)
- **B.** WYSIWYG + Block editors (adds EditorJS, BlockNote)
- **C.** All content authoring (adds VvvebJS, GrapesJS)
- **D.** Separate packages per family

**User pick:** custom — "one ux-editor ergonomic minimum viable bundle with many Bridges for All content authoring editors. One common API for all of them"

Equivalent to option **C** with constraints: single package, single API.

---

## Q2/Q3 — Content / output model

Options:
- **A.** `EditorContent` value object with `format` discriminator (uniform API).
- **B.** Per-bridge DataTransformer + `string` storage everywhere.
- **C.** Polymorphic per-bridge content classes implementing common interface.

**User pick:** **Hybrid A+B+C** — polymorphic value objects for IDE help, `EditorContentInterface` as uniform contract, per-bridge `DataTransformer` for storage flexibility.

**Decision:** three-layer design — `HtmlContent`/`BlockContent`/`PageContent` (polymorphic) all implement `EditorContentInterface` (contract); each Tier 2 bridge ships its own `DataTransformer` declaring `Scalar`/`Json`/`Split` storage shape. Two Doctrine integration modes: custom `Type` or `Embeddable`.

---

## Q4/Q5 — Config shape for `EditorType` / bridges

Options:
- **A.** Common options + escape hatch (typed common + `nativeOptions: array` passthrough).
- **B.** Presets only + raw native overrides.
- **C.** Per-bridge typed config objects.

**User pick:** **A + C combined** — common options + escape hatch AND per-bridge typed config objects.

**Decision:** layered config — `EditorConfigInterface` + `AbstractEditorConfig` (`translateCommon` + `translateOwn` + `nativeOverrides`), per-bridge typed config classes (`CKEditorConfig`, `EditorJSConfig`, etc.). `CommonOptions` DTO carries the 80% portable options; `BridgeCapabilities` flags what each bridge supports. Three input modes for `EditorType`: typed config, preset name, or quick array. Presets registered as services tagged `ux.editor.preset`.

---

## Q6 — JS distribution + Stimulus registration

Options:
- **A.** Per-bridge composer + npm sub-packages with peerDeps on upstream libs.
- **B.** Bundle upstream libs into each bridge npm package.
- **C.** CDN loader with subresource integrity.

**User pick:** **A** + auto-register controllers + both Encore and AssetMapper supported.

**Decision:** each Tier 2 bridge ships its own composer + npm sub-package. `peerDependencies` declare upstream version. Controllers auto-registered via `assets/controllers.json` (StimulusBundle reads this in both Encore and AssetMapper). Lazy load via dynamic `import()` of upstream lib inside `createEditor()`. Controller id format: `symfony--ux-editor--{bridge}`.

---

## Q7a — Sanitization default

Options:
- **A.** Sanitize on submit, opt-out per field.
- **B.** Never sanitize automatically; render-time only.
- **C.** Sanitize on both submit and render.

**User pick:** **A**.

**Decision:** `EditorType` runs `symfony/html-sanitizer` on `HtmlContent` after the transformer. `sanitize: false` disables; `sanitizer: 'profile_name'` picks profile. `BlockContent`: per-block sanitizers for HTML-bearing fields. `PageContent`: sanitize off by default (warns in dev). Bundle config `editor.html.sanitize_required: true` (default in prod) forces boot-time check.

---

## Q7b — Asset upload pipeline

Options:
- **A.** Common `EditorUploadInterface` + default controller route.
- **B.** App-owned endpoint; bridge accepts URL config only.
- **C.** Defer uploads to v2.

**User pick:** **A**.

**Decision:** package owns `POST /_ux_editor/upload/{field}` route, guarded by signed URL (HMAC + CSRF token). Apps implement `EditorUploadHandlerInterface`. Default `DefaultLocalUploadHandler` ships. Per-bridge upload adapters bridge upstream lib's upload API to `SignedUploadClient`.

---

## Q8 — LiveComponent integration

Options:
- **A.** Mount once, never re-morph; hot config reload via `applyConfig(diff)`.
- **B.** Mount once, full destroy+create on any Live re-render.
- **C.** No special integration.

**User pick:** **A** + autosave trait **yes**.

**Decision:** mount wrapped in `data-live-ignore`. `AbstractEditorController.configValueChanged()` computes diff; hot-reloadable keys applied in-place, others trigger destroy+remount. Bridges declare `hotReloadable` whitelist. `LiveEditor` trait provides debounced `#[LiveAction] saveDraft` + dirty flag UI.

---

## Q9 — Bridge phasing for v1

Options:
- **A.** v1 ships 3 bridges (one per format): CKEditor + EditorJS + GrapesJS.
- **B.** v1 ships all 8.
- **C.** v1 ships 2 WYSIWYG only.

**User pick:** **A** (confirmed after clarification on the 8 bridges).

**Decision:** v1 = EditorJS + CKEditor + GrapesJS, in that implementation order. v1.x adds Quill, TinyMCE, TipTap, BlockNote. v2 adds VvvebJS.

---

## Q10 — Three-tier abstraction with format-bridge layer

Triggered by user question: *"is there a way to have an abstraction in the core bundle to have representation of WYSIWYG | JSON | HTML+CSS+components or Html | Blocks | Page?"*

Options:
- **A.** Approve both (three-tier + v1 phasing).
- **B.** Approve tier, change phasing.
- **C.** Adjust tier.

**User pick:** **A**.

**Decision:** Tier 0 (core) → Tier 1 (format abstracts: `AbstractWysiwygBridge`, `AbstractBlockBridge`, `AbstractPageBuilderBridge` + format config/transformer/controller abstracts + capability factories) → Tier 2 (specific bridges). Tier 1 lives in core composer package, not separate packages. Each Tier 2 bridge inherits format-shared logic — ~150 LOC PHP + ~50 LOC TS each.

---

## Q11 — Testing surface

Options:
- **A.** Three-layer test pyramid (PHPUnit + Vitest + Playwright per bridge).
- **B.** PHPUnit-only + smoke screenshot.
- **C.** Defer E2E to per-bridge experimental phase.

**User pick:** **A**.

**Decision:** PHPUnit for Tier 0/1/2 PHP (≥ 90% Tier 0/1, ≥ 75% Tier 2). Vitest for JS abstracts + utilities (≥ 85%). Playwright per bridge + cross-cutting Live specs, sharded as Playwright projects (4 projects in v1: ckeditor, editorjs, grapesjs, live).

---

## Q12–Q16 — Section approvals

User approved each design section A–A–A–A–A:
- Q12 — Architecture (three-tier stack diagram): **A**
- Q13 — Components (PHP + JS + demo app layouts): **A**
- Q14 — Data flow (render/submit/hydrate/read-only/live/autosave/upload): **A**
- Q15 — Error handling (exception hierarchy + defensive guarantees + dev aids): **A**
- Q16 — Testing strategy (three-layer pyramid + CI matrix + coverage targets): **A**

---

## Final shape

- **1 core composer package** (`symfony/ux-editor`) containing Tier 0 + Tier 1 abstracts.
- **8 bridge composer + npm sub-packages** (`symfony/ux-editor-{bridge}`), one per editor.
- **3 ship in v1**, 5 follow in v1.x and v2.
- Single `EditorType` form field, polymorphic content value objects, uniform interface, per-bridge transformer.
- Both Encore and AssetMapper supported.
- LiveComponent + autosave integrated.
- Three-layer test pyramid with sharded Playwright projects per bridge.
