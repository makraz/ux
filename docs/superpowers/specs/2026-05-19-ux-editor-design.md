# UX Editor — Design Specification

**Date:** 2026-05-19
**Status:** Approved (brainstorm complete, awaiting plan)
**Scope:** New Symfony UX package providing a single `EditorType` API over multiple content-authoring editors (WYSIWYG, block, page builder).
**Target version:** v1.0 ships three bridges; v1.x and v2 add remaining bridges.

---

## 1. Goal

Provide one Symfony UX package, `symfony/ux-editor`, that exposes a single PHP/Twig/Stimulus integration for content-authoring editors. Apps use the same `EditorType` form field regardless of which underlying editor library renders the UI.

Eight target editors split across three content-format families:

| Format | Bridges                                    |
|--------|--------------------------------------------|
| Html (WYSIWYG)         | CKEditor, Quill, TinyMCE, TipTap |
| Blocks (JSON)          | EditorJS, BlockNote              |
| Page (HTML+CSS+components) | GrapesJS, VvvebJS            |

Single common API across all bridges, hybrid content model (polymorphic value objects + uniform interface + per-bridge persistence), three-tier abstraction (Core / Format / Specific).

## 2. Non-goals

- Real-time collaborative editing (out of scope; bridges may add later behind feature flag).
- Headless server rendering of editor UI (each bridge owns its DOM lifecycle).
- Custom diff/merge across editor formats (only format-internal conversion provided; cross-format conversion opt-in via registered converters).
- Replacing existing UX packages (`ux-twig-component`, `ux-live-component`, `ux-stimulus-bundle`) — this package depends on them.

## 3. Architecture — three-tier stack

```
Tier 0 — Core (symfony/ux-editor)
    Content value objects, EditorContentInterface, EditorContentFormat enum,
    EditorType, AbstractEditorController (JS), BridgeInterface,
    ContentConverterRegistry, EditorUploadInterface + default route,
    LiveEditor trait, Twig ux_editor_render(), Doctrine types,
    SignedUploadUrlGenerator, BridgeRegistry, PresetRegistry.

Tier 1 — Format abstracts (live in symfony/ux-editor under Bridge/Format/)
    AbstractWysiwygBridge   AbstractBlockBridge   AbstractPageBuilderBridge
    AbstractWysiwygConfig   AbstractBlockConfig   AbstractPageConfig
    AbstractWysiwygTransformer  AbstractBlockTransformer  AbstractPageTransformer
    WysiwygCapabilities     BlockCapabilities     PageCapabilities
    AbstractWysiwygController  AbstractBlockController  AbstractPageBuilderController
    BlockRendererRegistry   PageAssetExtractor   PageSandboxRenderer

Tier 2 — Specific bridges (own composer + npm sub-packages)
    CKEditor   Quill   TinyMCE   TipTap   EditorJS   BlockNote   GrapesJS   VvvebJS
```

Each Tier 2 bridge: ~150 LOC PHP + ~50 LOC TS, inherits everything format-shared, declares only id + controller name + capability overrides + `createEditor()` + `translateCommon/Own()`.

## 4. Content model — hybrid (polymorphic + interface + per-bridge transformer)

### 4.1 Polymorphic value objects (Layer 1 — domain shape, IDE help)

```php
enum EditorContentFormat: string {
    case Html   = 'html';
    case Blocks = 'blocks';
    case Page   = 'page';
}

interface EditorContentInterface {
    public function getFormat(): EditorContentFormat;
    public function getRaw(): string|array;
    public function getMetadata(): array;   // version, bridgeId, schemaVersion
    public function isEmpty(): bool;
}

abstract class EditorContent implements EditorContentInterface {
    public function __construct(
        public readonly EditorContentFormat $format,
        public readonly array $metadata = [],
    ) {}
}

final class HtmlContent extends EditorContent {
    public function __construct(public readonly string $html, array $metadata = []) {
        parent::__construct(EditorContentFormat::Html, $metadata);
    }
    public function getRaw(): string { return $this->html; }
    public function getSanitized(?HtmlSanitizerInterface $s = null): string;
    public function isEmpty(): bool { return trim(strip_tags($this->html)) === ''; }
    public static function fromString(string $html, array $meta = []): self;
}

final class BlockContent extends EditorContent {
    /** @param list<array{type:string,data:array,id?:string}> $blocks */
    public function __construct(
        public readonly array $blocks,
        public readonly string $schemaVersion = '1.0',  // first-class: drives migration of stored block shape
        array $metadata = [],                            // reserved for arbitrary keys (bridgeId, editor version, ...)
    ) { parent::__construct(EditorContentFormat::Blocks, $metadata); }
    public function getRaw(): array { return $this->blocks; }
    public function filterByType(string $type): self;
    public function toHtml(BlockRendererRegistry $r): string;
    public function isEmpty(): bool { return $this->blocks === []; }
}

final class PageContent extends EditorContent {
    public function __construct(
        public readonly string $html,
        public readonly string $css = '',
        public readonly array  $assets = [],
        public readonly array  $components = [],
        array $metadata = [],
    ) { parent::__construct(EditorContentFormat::Page, $metadata); }
    public function getRaw(): array { return ['html' => $this->html, 'css' => $this->css, 'components' => $this->components]; }
    public function extractAssets(): array { return $this->assets; }
    public function isEmpty(): bool { return $this->html === '' && $this->components === []; }
}
```

Entity field typed `?HtmlContent $body` (or `?BlockContent`, `?PageContent`) → full IDE autocomplete. Switching CKEditor → Quill: same `HtmlContent` type, zero entity change. Switching CKEditor → EditorJS: type changes to `BlockContent` — intentional break that forces explicit migration.

### 4.2 Uniform contract (Layer 2 — `EditorContentInterface`)

Boundary code (form, Twig, Stimulus, normalizer) consumes only the interface. Adding a 9th format = new enum case + new subclass, existing consumers branch on format only if they care.

### 4.3 Per-bridge `DataTransformer` (Layer 3 — persistence flexibility)

Each Tier 2 bridge ships its own `EditorContentTransformerInterface` implementation. Three storage shapes:

| Shape  | Used by | Doctrine column |
|--------|---------|------------------|
| `Scalar` | CKEditor, Quill, TinyMCE, TipTap | `TEXT` |
| `Json`   | EditorJS, BlockNote               | `JSON` |
| `Split`  | VvvebJS, GrapesJS                 | `JSON` bundle OR three columns via Embeddable |

Two integration modes for entities, user picks per field:

```php
// Mode A — custom Doctrine Type
#[ORM\Column(type: 'editor_html', nullable: true)]
private ?HtmlContent $body = null;

// Mode B — Embedded (page builder split columns)
#[ORM\Embedded(class: PageContent::class)]
private PageContent $homepage;
```

Package ships `EditorContentHtmlType`, `EditorContentBlocksType`, `EditorContentPageType` (custom Doctrine `Type`s).

### 4.4 Cross-bridge interchange

`ContentConverterRegistry` is opt-in. Default install: identity converters only. Pairs added as bridges land. Same-format conversion = lossless for WYSIWYG pairs, requires schema map for Block pairs, bespoke for Page pairs.

## 5. Config layer — common options + escape hatch + per-bridge typed objects

### 5.1 Common contract

```php
interface EditorConfigInterface {
    public function getBridgeId(): string;
    public function getCommon(): CommonOptions;
    public function getNativeOverrides(): array;
    public function getCapabilities(): BridgeCapabilities;
    public function toNative(): array;
}

final class CommonOptions {
    public function __construct(
        public readonly ?array  $toolbar = null,
        public readonly ?string $placeholder = null,
        public readonly bool    $readOnly = false,
        public readonly ?string $height = null,
        public readonly ?string $theme = null,
        public readonly ?string $language = null,
        public readonly array   $plugins = [],
        public readonly bool    $autofocus = false,
        public readonly bool    $spellcheck = true,
    ) {}
}

final class BridgeCapabilities {
    public function __construct(
        public readonly bool  $supportsToolbar,
        public readonly bool  $supportsPlugins,
        public readonly bool  $supportsTheme,
        public readonly bool  $supportsLanguage,
        /** @var list<'html'|'blocks'|'page'> */
        public readonly array $supportedFormats,
    ) {}
}
```

### 5.2 Abstract base

```php
abstract class AbstractEditorConfig implements EditorConfigInterface {
    public function __construct(
        protected CommonOptions $common = new CommonOptions(),
        protected array $nativeOverrides = [],
    ) {}
    public function toNative(): array {
        $this->assertCapabilities();
        $base = $this->translateCommon($this->common);
        $own  = $this->translateOwn();
        return array_replace_recursive($base, $own, $this->nativeOverrides);
    }
    abstract protected function translateCommon(CommonOptions $c): array;
    protected function translateOwn(): array { return []; }
}
```

`nativeOverrides` is the escape hatch — merged last, wins any conflict.

### 5.3 Per-bridge typed config

`CKEditorConfig`, `QuillConfig`, `TinyMCEConfig`, `TipTapConfig`, `EditorJSConfig`, `BlockNoteConfig`, `GrapesJSConfig`, `VvvebJSConfig`. Each ~80–150 LOC, typed properties for bridge-specific options, implements `translateCommon/Own`.

### 5.4 Three input modes for `EditorType`

```php
// 1 — typed config (full IDE help, recommended)
$builder->add('body', EditorType::class, [
    'config' => new CKEditorConfig(
        common: new CommonOptions(toolbar: ['bold','italic','link'], placeholder: 'Write…', height: '400px'),
        extraPlugins: ['SourceEditing'],
        nativeOverrides: ['ui' => ['poweredBy' => ['forceVisible' => false]]],
    ),
]);

// 2 — preset by name (service-registered)
$builder->add('body', EditorType::class, ['preset' => 'blog.standard']);

// 3 — bridge + array (quick path)
$builder->add('body', EditorType::class, [
    'bridge' => 'quill',
    'common' => ['toolbar' => ['bold','italic']],
    'native' => ['modules' => ['clipboard' => ['matchVisual' => false]]],
]);
```

Mode 3 mapping: `'common'` array → `new CommonOptions(...$arr)`; `'native'` array → `$nativeOverrides`. `EditorType` resolves all three modes to a single `EditorConfigInterface` instance before view-building.

### 5.5 Presets

Services tagged `ux.editor.preset` with `name` attribute, implementing `EditorPresetInterface::build(): EditorConfigInterface`. Ships defaults: `wysiwyg.minimal`, `wysiwyg.full`, `blog.standard`, `page_builder.landing`. App overrides built-ins by registering same name (priority).

### 5.6 Capability guard

At form build time, `assertCapabilities()` checks `CommonOptions` against `BridgeCapabilities`. Default: log warning. With `strictCapabilities: true`: throws `IncompatibleConfigException`.

## 6. Distribution — per-bridge composer + npm sub-packages

- Core: `symfony/ux-editor` (composer) + `@symfony/ux-editor` (npm).
- Each bridge: `symfony/ux-editor-{bridge}` + `@symfony/ux-editor-{bridge}`.
- Each bridge `package.json` declares `peerDependencies` for upstream lib (e.g. `@ckeditor/ckeditor5-build-classic`). App pins its own version.
- Each bridge ships `assets/controllers.json` so StimulusBundle auto-registers controller. ID format: `symfony--ux-editor--{bridge}`.
- Lazy load: controller's `createEditor()` uses dynamic `import()` for upstream lib → AssetMapper resolves via importmap, Encore via webpack split-chunks. Same source code.
- `splitsh.json` (already in repo) updated to split each bridge to its own repo.

## 7. Security

### 7.1 Sanitization

- `HtmlContent`: sanitized on submit via `symfony/html-sanitizer`. Default in `EditorType`: `sanitize: true`. App passes `sanitize: false` or `sanitizer: 'profile_name'`.
- `BlockContent`: per-block sanitizers run on submit (HTML inside block data fields, e.g. paragraph's `text`). Registered per block type.
- `PageContent`: sanitize off by default (page builders expect non-sanitized HTML/CSS). Warns loudly in dev mode.
- Bundle config flag `editor.html.sanitize_required: true` (default `true` in prod env) → boot-time exception if no sanitizer service wired.

### 7.2 Upload pipeline

- Package owns route `POST /_ux_editor/upload/{field}`.
- Guarded by signed URL via `SignedUploadUrlGenerator` (HMAC over field + profile + expiry + CSRF token).
- Delegates to app-implemented `EditorUploadHandlerInterface` (validates mime/size, stores via Flysystem/local, returns URL).
- Default impl `DefaultLocalUploadHandler` writes to local public dir, configurable target + allowed mimes.
- Each bridge ships an upload adapter that bridges its upstream library's upload API (CKEditor `SimpleUploadAdapter` config, EditorJS Image tool `uploader.uploadByFile`, GrapesJS asset manager) to `SignedUploadClient`.

## 8. LiveComponent integration

- Mount wrapped in `data-live-ignore` so morphdom skips internal subtree.
- Controller root receives attr-level patches (config-value, format-value, upload-url updates).
- `AbstractEditorController.configValueChanged(newConfig, oldConfig)`:
  - Compute diff. If every changed key is in bridge's `hotReloadable` set → `bridge.applyConfig(diff)`.
  - Otherwise destroy + remount (and dispatch `ux:editor:remount` so app can warn user).
- Each bridge declares `hotReloadable` whitelist (e.g. CKEditor hot-applies `readOnly`, `placeholder`, `language`).

### 8.1 Autosave — `LiveEditor` trait

```php
trait LiveEditor {
    #[LiveAction]
    public function saveDraft(string $field, mixed $content): void {
        $this->draftRepo->upsert($this->getEntityId(), $field, $this->transformIncoming($content));
        $this->dirty[$field] = false;
        $this->lastSavedAt[$field] = new \DateTimeImmutable();
    }
}
```

Client side: `live-editor.ts` debounces `ux:editor:change` events 800ms then dispatches Live action with serialized content. CSRF refreshed transparently on stale-token detection.

## 9. Twig rendering

```twig
{# input — handled by EditorType view #}
{{ form_widget(form.body) }}

{# read-only render of saved content #}
{{ ux_editor_render(article.body) }}
{{ ux_editor_render(article.body, { sanitize: true, target: '_iframe' }) }}
```

`ux_editor_render` branches on format:
- `HtmlContent` → sanitized echo.
- `BlockContent` → walks blocks through `BlockRendererRegistry`; missing renderer → HTML comment in prod, visible warning box in `kernel.debug`.
- `PageContent` → `<iframe srcdoc>` with `sandbox="allow-same-origin"` OR shadow DOM with CSS scoping (controlled by `target` option).

Default block renderers shipped per bridge: `Paragraph`, `Header`, `Image`, `List`, `Quote` (EditorJS); BlockNote ships its own.

## 10. Phasing

| Version | Bridges shipped                          | Order  |
|---------|-------------------------------------------|--------|
| **v1.0** | EditorJS, CKEditor, GrapesJS             | EditorJS → CKEditor → GrapesJS |
| v1.x    | Quill, TinyMCE, TipTap, BlockNote        | Added incrementally |
| v2.0    | VvvebJS                                  | Last (heaviest deps: jQuery + Bootstrap) |

Rationale for v1 picks: one bridge per content format exercises the polymorphic content model, transformer storage shapes, capability flags, and converter matrix against real code on day 1. Order: smallest/simplest bridge first (EditorJS), then largest WYSIWYG (CKEditor) to stress capability flags + licensing path, then Page format to finalize sandbox renderer.

## 11. Error handling

Exception hierarchy under `Symfony\UX\Editor\Exception\`:

```
EditorExceptionInterface              # marker
├── UnknownBridgeException
├── BridgeConfigMismatchException
├── IncompatibleConfigException
├── UnsupportedConversionException
├── UploadException
│   ├── InvalidSignatureException
│   ├── UnsupportedFileException
│   └── UploadHandlerException
└── ContentSchemaException
```

Defensive guarantees:
- Malformed stored data → exception, never silent-default. Lost data must be loud.
- Form-layer errors map to field errors via `TransformationFailedException`.
- JS controller never blocks form submit; fallback to plain textarea on lib load failure.
- Sanitizer mandatory in prod for `HtmlContent` (boot-time check).
- Capability strictness opt-in per form.
- No built-in retry for autosave/upload — exposed via events, app concern.

Dev aids:
- `WebProfilerBundle` data collector: bridges in use, configs resolved, capability warnings, transformer roundtrips.
- `bin/console debug:ux-editor` — lists bridges, presets, converters, upload handlers.
- Visible warning boxes in `kernel.debug` for missing block renderers / sanitizer skips.

## 12. Testing — three-layer pyramid

### PHPUnit
- Tier 0: content classes, transformer interface, `EditorType` option resolution, registries, Doctrine types, `SignedUploadUrlGenerator`, `EditorUploadController` functional, `LiveEditor` trait, `EditorRenderExtension`.
- Tier 1: each format abstract via minimal concrete test subclass — transformer round-trip, capability factory, renderer registry.
- Tier 2: per-bridge config translate happy + edge, transformer smoke, bridge metadata.

### Vitest
- `AbstractEditorController` lifecycle, syncInput, configValueChanged hot vs remount.
- Each format-abstract controller — paste sanitize hook, serialize shape, sandbox iframe lifecycle.
- `SignedUploadClient`, `live-editor.ts` debouncer.

### Playwright E2E (per bridge + cross-cutting Live)
- Per bridge: `mounts`, `serialize-roundtrip`, `capability-warning`, `upload`, `sanitize-on-submit` (WYSIWYG only).
- Cross-cutting `live/`: `hot-reload`, `autosave`, `live-ignore`.
- Sharded as Playwright projects: one per bridge + one for live = 4 projects in v1.

### Coverage targets v1
- PHPUnit Tier 0 + Tier 1: ≥ 90% line coverage.
- Tier 2 bridges: ≥ 75% line coverage.
- Vitest: ≥ 85% for controller abstracts.
- E2E: every public-facing flow covered for each v1 bridge.

## 13. Demo app — `ux.symfony.com`

Three demo routes (`/editor/wysiwyg`, `/editor/block`, `/editor/page`), each renders the same `EditorType`-driven form with the corresponding v1 bridge. Plus one `/editor/live` route demonstrating cross-bridge LiveComponent integration (hot config reload + autosave indicator).

## 14. Extension surfaces — single seam per task

| Task                          | Files touched                                   |
|-------------------------------|-------------------------------------------------|
| Add new common option         | `CommonOptions` + each Tier 1 `translateCommon` |
| Add new bridge                | New Tier 2 directory: `XxxBridge`, `XxxConfig`, `XxxController`, `composer.json`, `package.json`, `controllers.json`, tests |
| Add new preset                | New service tagged `ux.editor.preset`          |
| Override native option        | `nativeOverrides` ctor arg, no class change     |
| Add new content format        | New enum case + new `XxxContent` subclass + Tier 1 format directory |
| Add cross-bridge converter    | New `ContentConverterInterface` impl + registry registration |
| Add new block renderer        | Service tagged `ux.editor.block_renderer`       |

## 15. Open questions parked for plan phase

- Exact API of `applyConfig(diff)` per bridge (concrete hot-reloadable keys per Tier 2).
- Default `BlockRendererRegistry` mapping per shipped block type — needs final EditorJS tool list.
- Doctrine migration helper for converting plain `text` columns to `editor_html` type.
- Whether `EditorContentInterface` should extend `\Stringable` (convenience vs format-blindness).
- Whether `splitsh.json` updates are part of v1 PR or follow-up — depends on Symfony release process.

These get resolved during writing-plans phase, not blocking design approval.

---

## Brainstorm decision log

| Q | Topic | Decision |
|---|-------|----------|
| Q1 | Scope | One package, 8 bridges, one common API |
| Q2/Q3 | Content model | Hybrid: polymorphic value objects + `EditorContentInterface` + per-bridge `DataTransformer` |
| Q4/Q5 | Config | Common options + escape hatch + per-bridge typed config objects + service-registered presets |
| Q6 | Distribution | Per-bridge composer + npm sub-packages with peerDeps; AssetMapper + Encore both supported |
| Q7a | Sanitization | Sanitize on submit, opt-out per field; `PageContent` exempt by default |
| Q7b | Upload | First-party `EditorUploadHandlerInterface` + signed-URL endpoint |
| Q8 | LiveComponent | Mount once + hot config reload via `applyConfig(diff)`; autosave trait yes |
| Q9 | Phasing | v1: EditorJS + CKEditor + GrapesJS; v1.x: Quill + TinyMCE + TipTap + BlockNote; v2: VvvebJS |
| Q10 | Abstraction depth | Three-tier stack with Tier 1 format-bridge layer in core package |
| Q11 | Testing | Three-layer pyramid (PHPUnit + Vitest + Playwright per bridge) |
