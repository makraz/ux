# UX Editor Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `symfony/ux-editor` core package (Tier 0 + Tier 1) — content value objects, EditorType form field, Doctrine types, upload pipeline, LiveComponent integration, and three format abstracts (Wysiwyg / Block / Page). No specific bridge in this plan.

**Architecture:** Three-tier stack. Tier 0 (core abstractions, value objects, registries) and Tier 1 (per-format abstract bridges + transformers + controllers + capability factories) live in a single composer package `symfony/ux-editor`. Tier 2 specific bridges (EditorJS, CKEditor, GrapesJS) are separate sub-packages, planned in follow-up plans.

**Tech Stack:** PHP 8.4, Symfony 7.4 / 8.0, `symfony/stimulus-bundle`, `symfony/html-sanitizer`, Doctrine ORM (custom Types + Embeddables), Stimulus 3, TypeScript 5, Vitest, PHPUnit 11/12.

**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](../specs/2026-05-19-ux-editor-design.md)
**Decisions:** [`DECISIONS-UX-EDITOR.md`](../../../DECISIONS-UX-EDITOR.md)

---

## File structure

```
src/Editor/
├── composer.json
├── phpunit.dist.xml
├── README.md
├── CHANGELOG.md
├── LICENSE
├── package.json
├── tsconfig.json
├── vitest.config.mjs
├── config/services.php
├── doc/index.rst
├── assets/
│   ├── controllers.json
│   ├── src/
│   │   ├── controller.ts
│   │   ├── content/{EditorContent,HtmlContent,BlockContent,PageContent}.ts
│   │   ├── format/{wysiwyg_controller,block_controller,page_builder_controller}.ts
│   │   ├── upload/{SignedUploadClient,BridgeUploaderInterface}.ts
│   │   └── live/live-editor.ts
│   └── test/
│       ├── controller.test.ts
│       ├── format/{wysiwyg,block,page}.test.ts
│       ├── upload.test.ts
│       └── live.test.ts
├── src/
│   ├── UXEditorBundle.php
│   ├── DependencyInjection/UXEditorExtension.php
│   ├── Content/{EditorContentInterface,EditorContentFormat,EditorContent,HtmlContent,BlockContent,PageContent}.php
│   ├── Content/Converter/{ContentConverterInterface,ContentConverterRegistry}.php
│   ├── Config/{EditorConfigInterface,AbstractEditorConfig,CommonOptions,BridgeCapabilities}.php
│   ├── Config/Preset/{EditorPresetInterface,PresetRegistry}.php
│   ├── Bridge/{BridgeInterface,AbstractBridge,BridgeRegistry}.php
│   ├── Bridge/Format/Wysiwyg/{AbstractWysiwygBridge,AbstractWysiwygConfig,AbstractWysiwygTransformer,WysiwygCapabilities}.php
│   ├── Bridge/Format/Block/{AbstractBlockBridge,AbstractBlockConfig,AbstractBlockTransformer,BlockRendererInterface,BlockRendererRegistry,BlockCapabilities}.php
│   ├── Bridge/Format/Page/{AbstractPageBuilderBridge,AbstractPageConfig,AbstractPageTransformer,PageAssetExtractor,PageSandboxRenderer,PageCapabilities}.php
│   ├── Form/EditorType.php
│   ├── Form/DataTransformer/EditorContentTransformerInterface.php
│   ├── Doctrine/{EditorContentHtmlType,EditorContentBlocksType,EditorContentPageType}.php
│   ├── Upload/{EditorUploadHandlerInterface,EditorUploadController,SignedUploadUrlGenerator,DefaultLocalUploadHandler}.php
│   ├── Live/LiveEditor.php
│   ├── Twig/EditorRenderExtension.php
│   └── Exception/{EditorExceptionInterface,UnknownBridgeException,BridgeConfigMismatchException,IncompatibleConfigException,UnsupportedConversionException,ContentSchemaException}.php
│   └── Exception/Upload/{UploadException,InvalidSignatureException,UnsupportedFileException,UploadHandlerException}.php
└── tests/   (mirror of src/)
```

Ordering: bottom-up. Enum → interfaces → value objects → registries → config → form/Doctrine → upload → Live → Twig → format abstracts → JS abstracts → bundle wiring.

Each task = TDD cycle (failing test → run → minimal impl → run → commit).

---

## Task 1 — Package scaffold

**Files:**
- Create: `src/Editor/{composer.json, phpunit.dist.xml, .gitignore, README.md, CHANGELOG.md, LICENSE, package.json, tsconfig.json, vitest.config.mjs, tests/.gitkeep}`

- [ ] **Step 1: Copy LICENSE**

```bash
cp src/Map/LICENSE src/Editor/LICENSE
```

- [ ] **Step 2: Write composer.json**

```json
{
    "name": "symfony/ux-editor",
    "type": "symfony-bundle",
    "description": "One Symfony UX form field over multiple content-authoring editors",
    "keywords": ["symfony-ux", "editor", "wysiwyg", "block-editor", "page-builder"],
    "homepage": "https://symfony.com",
    "license": "MIT",
    "authors": [
        {"name": "Symfony Community", "homepage": "https://symfony.com/contributors"}
    ],
    "autoload": {"psr-4": {"Symfony\\UX\\Editor\\": "src/"}, "exclude-from-classmap": []},
    "autoload-dev": {"psr-4": {"Symfony\\UX\\Editor\\Tests\\": "tests/"}},
    "require": {
        "php": ">=8.4",
        "symfony/stimulus-bundle": "^2.18.1|^3.0",
        "symfony/form": "^7.4|^8.0",
        "symfony/options-resolver": "^7.4|^8.0",
        "symfony/html-sanitizer": "^7.4|^8.0"
    },
    "require-dev": {
        "doctrine/orm": "^3.0",
        "doctrine/dbal": "^4.0",
        "phpunit/phpunit": "^11.1|^12.0",
        "symfony/asset-mapper": "^7.4|^8.0",
        "symfony/framework-bundle": "^7.4|^8.0",
        "symfony/twig-bundle": "^7.4|^8.0",
        "symfony/security-csrf": "^7.4|^8.0",
        "symfony/ux-live-component": "^2.21|^3.0",
        "symfony/ux-twig-component": "^2.21|^3.0"
    },
    "extra": {"thanks": {"name": "symfony/ux", "url": "https://github.com/symfony/ux"}},
    "minimum-stability": "dev"
}
```

- [ ] **Step 3: Write phpunit.dist.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/11.0/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         failOnDeprecation="true"
         failOnNotice="true"
         failOnWarning="true">
    <php>
        <ini name="error_reporting" value="-1"/>
        <server name="APP_ENV" value="test" force="true"/>
    </php>
    <testsuites>
        <testsuite name="symfony/ux-editor"><directory>./tests/</directory></testsuite>
    </testsuites>
    <source><include><directory>./src</directory></include></source>
</phpunit>
```

- [ ] **Step 4: Write .gitignore**

```
/vendor/
/composer.lock
/node_modules/
/dist/
.phpunit.result.cache
```

- [ ] **Step 5: Write package.json**

```json
{
    "name": "@symfony/ux-editor",
    "description": "Core abstractions for symfony/ux-editor bridges",
    "license": "MIT",
    "version": "0.1.0",
    "symfony": {"controllers": {}, "importmap": {}},
    "type": "module",
    "main": "dist/controller.js",
    "types": "dist/controller.d.ts",
    "exports": {
        ".":            {"types": "./dist/controller.d.ts", "default": "./dist/controller.js"},
        "./format/wysiwyg": {"types": "./dist/format/wysiwyg_controller.d.ts", "default": "./dist/format/wysiwyg_controller.js"},
        "./format/block":   {"types": "./dist/format/block_controller.d.ts",   "default": "./dist/format/block_controller.js"},
        "./format/page":    {"types": "./dist/format/page_builder_controller.d.ts", "default": "./dist/format/page_builder_controller.js"},
        "./upload":     {"types": "./dist/upload/SignedUploadClient.d.ts", "default": "./dist/upload/SignedUploadClient.js"},
        "./live":       {"types": "./dist/live/live-editor.d.ts", "default": "./dist/live/live-editor.js"},
        "./content":    {"types": "./dist/content/EditorContent.d.ts", "default": "./dist/content/EditorContent.js"}
    },
    "files": ["dist/"],
    "peerDependencies": {"@hotwired/stimulus": "^3.0.0"},
    "devDependencies": {
        "@hotwired/stimulus": "^3.2.2",
        "typescript": "^5.4.0",
        "vitest": "^1.5.0",
        "jsdom": "^24.0.0"
    }
}
```

- [ ] **Step 6: Write tsconfig.json**

```json
{
    "extends": "../../tsconfig.package.json",
    "compilerOptions": {"outDir": "./dist", "rootDir": "./assets/src"},
    "include": ["assets/src/**/*"]
}
```

- [ ] **Step 7: Write vitest.config.mjs**

```js
import { defineConfig } from 'vitest/config';
import base from '../../vitest.config.base.mjs';
export default defineConfig({ ...base, test: { ...base.test, environment: 'jsdom', include: ['assets/test/**/*.test.ts'] }});
```

- [ ] **Step 8: Write stub README.md + CHANGELOG.md + tests/.gitkeep**

`README.md`:
```markdown
# Symfony UX Editor

Core abstractions for content-authoring editor bridges (WYSIWYG, block, page builder).
See [doc/index.rst](doc/index.rst). Bridges ship as separate packages.
```

`CHANGELOG.md`:
```markdown
# CHANGELOG

## 0.1.0

- Initial release: Tier 0 core + Tier 1 format abstracts.
```

- [ ] **Step 9: Install deps + smoke-run**

```bash
cd src/Editor && composer install
vendor/bin/phpunit
```

Expected: PHPUnit runs 0 tests, exits 0.

- [ ] **Step 10: Commit**

```bash
git add src/Editor/
git commit -m "feat(editor): scaffold symfony/ux-editor package"
```

---

## Task 2 — `EditorContentFormat` enum

**Files:**
- Create: `src/Editor/src/Content/EditorContentFormat.php`
- Test: `src/Editor/tests/Content/EditorContentFormatTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\EditorContentFormat;

final class EditorContentFormatTest extends TestCase
{
    public function testCases(): void
    {
        self::assertSame('html',   EditorContentFormat::Html->value);
        self::assertSame('blocks', EditorContentFormat::Blocks->value);
        self::assertSame('page',   EditorContentFormat::Page->value);
        self::assertSame(EditorContentFormat::Html, EditorContentFormat::from('html'));
    }
}
```

- [ ] **Step 2: Run — expect "Class … not found"**

```bash
cd src/Editor && vendor/bin/phpunit tests/Content/EditorContentFormatTest.php
```

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Content;

enum EditorContentFormat: string
{
    case Html   = 'html';
    case Blocks = 'blocks';
    case Page   = 'page';
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/EditorContentFormat.php src/Editor/tests/Content/EditorContentFormatTest.php
git commit -m "feat(editor): add EditorContentFormat enum"
```

---

## Task 3 — `EditorContentInterface` + `EditorContent` base

**Files:**
- Create: `src/Editor/src/Content/EditorContentInterface.php`
- Create: `src/Editor/src/Content/EditorContent.php`
- Test: `src/Editor/tests/Content/EditorContentTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\EditorContent;
use Symfony\UX\Editor\Content\EditorContentFormat;
use Symfony\UX\Editor\Content\EditorContentInterface;

final class EditorContentTest extends TestCase
{
    public function testAbstractCarriesFormatAndMetadata(): void
    {
        $stub = new class('hi', ['bridgeId' => 'fake']) extends EditorContent {
            public function __construct(public readonly string $raw, array $meta) {
                parent::__construct(EditorContentFormat::Html, $meta);
            }
            public function getRaw(): string { return $this->raw; }
            public function isEmpty(): bool  { return $this->raw === ''; }
        };
        self::assertInstanceOf(EditorContentInterface::class, $stub);
        self::assertSame(EditorContentFormat::Html, $stub->getFormat());
        self::assertSame(['bridgeId' => 'fake'], $stub->getMetadata());
        self::assertSame('hi', $stub->getRaw());
        self::assertFalse($stub->isEmpty());
    }
}
```

- [ ] **Step 2: Run — expect "Interface not found"**

- [ ] **Step 3: Implementation**

`EditorContentInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Content;

interface EditorContentInterface
{
    public function getFormat(): EditorContentFormat;
    public function getRaw(): string|array;
    public function getMetadata(): array;
    public function isEmpty(): bool;
}
```

`EditorContent.php`:
```php
<?php
namespace Symfony\UX\Editor\Content;

abstract class EditorContent implements EditorContentInterface
{
    public function __construct(
        public readonly EditorContentFormat $format,
        public readonly array $metadata = [],
    ) {}

    public function getFormat(): EditorContentFormat { return $this->format; }
    public function getMetadata(): array             { return $this->metadata; }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/EditorContentInterface.php src/Editor/src/Content/EditorContent.php src/Editor/tests/Content/EditorContentTest.php
git commit -m "feat(editor): add EditorContentInterface and EditorContent base"
```

---

## Task 4 — `HtmlContent`

**Files:**
- Create: `src/Editor/src/Content/HtmlContent.php`
- Test: `src/Editor/tests/Content/HtmlContentTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;
use Symfony\UX\Editor\Content\EditorContentFormat;
use Symfony\UX\Editor\Content\HtmlContent;

final class HtmlContentTest extends TestCase
{
    public function testFormatAndRaw(): void
    {
        $c = new HtmlContent('<p>hi</p>');
        self::assertSame(EditorContentFormat::Html, $c->getFormat());
        self::assertSame('<p>hi</p>', $c->getRaw());
    }

    public function testIsEmpty(): void
    {
        self::assertTrue((new HtmlContent(''))->isEmpty());
        self::assertTrue((new HtmlContent('   '))->isEmpty());
        self::assertTrue((new HtmlContent('<p>  </p>'))->isEmpty());
        self::assertFalse((new HtmlContent('<p>hi</p>'))->isEmpty());
    }

    public function testFromString(): void
    {
        $c = HtmlContent::fromString('<b>x</b>', ['bridgeId' => 'ckeditor']);
        self::assertSame('<b>x</b>', $c->html);
        self::assertSame(['bridgeId' => 'ckeditor'], $c->getMetadata());
    }

    public function testGetSanitizedWithProvidedSanitizer(): void
    {
        $s = new HtmlSanitizer((new HtmlSanitizerConfig())->allowSafeElements());
        $out = (new HtmlContent('<script>x</script><p>ok</p>'))->getSanitized($s);
        self::assertStringNotContainsString('<script>', $out);
        self::assertStringContainsString('<p>ok</p>', $out);
    }

    public function testGetSanitizedWithoutSanitizerReturnsRaw(): void
    {
        self::assertSame('<p>x</p>', (new HtmlContent('<p>x</p>'))->getSanitized(null));
    }
}
```

- [ ] **Step 2: Run — expect "HtmlContent not found"**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Content;

use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;

final class HtmlContent extends EditorContent
{
    public function __construct(public readonly string $html, array $metadata = [])
    {
        parent::__construct(EditorContentFormat::Html, $metadata);
    }

    public function getRaw(): string { return $this->html; }

    public function isEmpty(): bool { return trim(strip_tags($this->html)) === ''; }

    public function getSanitized(?HtmlSanitizerInterface $sanitizer = null): string
    {
        return $sanitizer === null ? $this->html : $sanitizer->sanitize($this->html);
    }

    public static function fromString(string $html, array $metadata = []): self
    {
        return new self($html, $metadata);
    }
}
```

- [ ] **Step 4: Run — expect 5 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/HtmlContent.php src/Editor/tests/Content/HtmlContentTest.php
git commit -m "feat(editor): add HtmlContent value object"
```

---

## Task 5 — `BlockContent`

**Files:**
- Create: `src/Editor/src/Content/BlockContent.php`
- Test: `src/Editor/tests/Content/BlockContentTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Content\EditorContentFormat;

final class BlockContentTest extends TestCase
{
    public function testFormat(): void
    {
        self::assertSame(EditorContentFormat::Blocks, (new BlockContent([]))->getFormat());
    }

    public function testRawAndSchemaVersionDefault(): void
    {
        $bc = new BlockContent([['type' => 'paragraph', 'data' => ['text' => 'hi']]]);
        self::assertSame([['type' => 'paragraph', 'data' => ['text' => 'hi']]], $bc->getRaw());
        self::assertSame('1.0', $bc->schemaVersion);
    }

    public function testIsEmpty(): void
    {
        self::assertTrue((new BlockContent([]))->isEmpty());
        self::assertFalse((new BlockContent([['type' => 'p', 'data' => []]]))->isEmpty());
    }

    public function testFilterByType(): void
    {
        $bc = new BlockContent([
            ['type' => 'header', 'data' => ['text' => 'H']],
            ['type' => 'paragraph', 'data' => ['text' => 'P']],
            ['type' => 'header', 'data' => ['text' => 'H2']],
        ]);
        self::assertCount(2, $bc->filterByType('header')->blocks);
    }

    public function testFromArrayFactory(): void
    {
        $bc = BlockContent::fromArray(['version' => '2.0', 'blocks' => [['type' => 'p', 'data' => []]]]);
        self::assertSame('2.0', $bc->schemaVersion);
        self::assertCount(1, $bc->blocks);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Content;

final class BlockContent extends EditorContent
{
    /** @param list<array{type:string,data:array,id?:string}> $blocks */
    public function __construct(
        public readonly array  $blocks,
        public readonly string $schemaVersion = '1.0',
        array $metadata = [],
    ) {
        parent::__construct(EditorContentFormat::Blocks, $metadata);
    }

    public function getRaw(): array { return $this->blocks; }

    public function isEmpty(): bool { return $this->blocks === []; }

    public function filterByType(string $type): self
    {
        return new self(
            array_values(array_filter($this->blocks, fn(array $b) => ($b['type'] ?? null) === $type)),
            $this->schemaVersion,
            $this->metadata,
        );
    }

    public static function fromArray(array $payload, array $metadata = []): self
    {
        return new self(
            $payload['blocks'] ?? [],
            (string)($payload['version'] ?? '1.0'),
            $metadata,
        );
    }
}
```

- [ ] **Step 4: Run — expect 5 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/BlockContent.php src/Editor/tests/Content/BlockContentTest.php
git commit -m "feat(editor): add BlockContent value object"
```

---

## Task 6 — `PageContent`

**Files:**
- Create: `src/Editor/src/Content/PageContent.php`
- Test: `src/Editor/tests/Content/PageContentTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\EditorContentFormat;
use Symfony\UX\Editor\Content\PageContent;

final class PageContentTest extends TestCase
{
    public function testFormat(): void
    {
        self::assertSame(EditorContentFormat::Page, (new PageContent(''))->getFormat());
    }

    public function testRawBundle(): void
    {
        $p = new PageContent(html: '<h1>x</h1>', css: 'h1{color:red}', components: [['type' => 'h1']]);
        $raw = $p->getRaw();
        self::assertSame('<h1>x</h1>',  $raw['html']);
        self::assertSame('h1{color:red}', $raw['css']);
        self::assertSame([['type' => 'h1']], $raw['components']);
    }

    public function testIsEmpty(): void
    {
        self::assertTrue((new PageContent(''))->isEmpty());
        self::assertFalse((new PageContent('<p>x</p>'))->isEmpty());
        self::assertFalse((new PageContent('', '', [], [['type' => 'p']]))->isEmpty());
    }

    public function testExtractAssets(): void
    {
        $assets = [['type' => 'image', 'url' => '/x.png']];
        self::assertSame($assets, (new PageContent('', '', $assets))->extractAssets());
    }

    public function testFromBundle(): void
    {
        $p = PageContent::fromBundle([
            'html' => '<p>x</p>',
            'css'  => 'p{}',
            'assets' => [['type' => 'image', 'url' => '/x.png']],
            'components' => [['type' => 'p']],
        ]);
        self::assertSame('<p>x</p>', $p->html);
        self::assertSame('p{}', $p->css);
        self::assertCount(1, $p->assets);
        self::assertCount(1, $p->components);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Content;

final class PageContent extends EditorContent
{
    public function __construct(
        public readonly string $html,
        public readonly string $css = '',
        public readonly array  $assets = [],
        public readonly array  $components = [],
        array $metadata = [],
    ) {
        parent::__construct(EditorContentFormat::Page, $metadata);
    }

    public function getRaw(): array
    {
        return ['html' => $this->html, 'css' => $this->css, 'components' => $this->components];
    }

    public function isEmpty(): bool { return $this->html === '' && $this->components === []; }

    public function extractAssets(): array { return $this->assets; }

    public static function fromBundle(array $bundle, array $metadata = []): self
    {
        return new self(
            html: (string)($bundle['html'] ?? ''),
            css: (string)($bundle['css'] ?? ''),
            assets: $bundle['assets'] ?? [],
            components: $bundle['components'] ?? [],
            metadata: $metadata,
        );
    }
}
```

- [ ] **Step 4: Run — expect 5 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/PageContent.php src/Editor/tests/Content/PageContentTest.php
git commit -m "feat(editor): add PageContent value object"
```

---

## Continuation marker — tasks 7-50

This file is the start of a longer plan. Remaining tasks listed in summary form below; each follows the same TDD 5-step structure. **Subsequent tasks must be written out with full code blocks before implementation begins** — see the "Plan continuation" section at the bottom for the strategy.

### Tasks 7-15 — Exceptions + Config layer

- **7.** Exception hierarchy: `EditorExceptionInterface`, `UnknownBridgeException`, `BridgeConfigMismatchException`, `IncompatibleConfigException`, `UnsupportedConversionException`, `ContentSchemaException`, `Upload/UploadException`, `Upload/InvalidSignatureException`, `Upload/UnsupportedFileException`, `Upload/UploadHandlerException`. One commit per exception class file group.
- **8.** `CommonOptions` readonly DTO + test (all options nullable except booleans).
- **9.** `BridgeCapabilities` readonly DTO + test (`->with()` clone-with method).
- **10.** `EditorConfigInterface` (contract methods: `getBridgeId`, `getCommon`, `getNativeOverrides`, `getCapabilities`, `toNative`).
- **11.** `AbstractEditorConfig` + test via in-test concrete subclass (`translateCommon`/`translateOwn`/`nativeOverrides` merge order verified).
- **12.** `assertCapabilities()` in `AbstractEditorConfig` — warn vs throw paths; needs `LoggerInterface` injected via setter.
- **13.** `EditorPresetInterface` + `PresetRegistry` (tagged-service iterator).
- **14.** `ContentConverterInterface` + `ContentConverterRegistry` (identity registry; throws `UnsupportedConversionException` on missing pair).
- **15.** Run full PHPUnit suite — green checkpoint.

### Tasks 16-22 — Bridge + Form + Transformer

- **16.** `BridgeInterface` (getId, getControllerName, getDefaultConfig, getCapabilities, createTransformer).
- **17.** `AbstractBridge` (boilerplate; abstract method delegates).
- **18.** `BridgeRegistry` (id-keyed map, duplicate-id throws `UnknownBridgeException` on get-miss).
- **19.** `EditorContentTransformerInterface` (`getBridgeId`, `getContentClass`, `getStorageShape`, `transform`, `reverseTransform`). Storage shape enum `StorageShape: Scalar|Json|Split`.
- **20.** `EditorType` — option resolver (config/preset/bridge+array modes), `buildForm` wires `addModelTransformer`, sanitize hook via `addEventListener(FormEvents::SUBMIT)`.
- **21.** `EditorType::buildView` — sets `data-controller`, `data-...-config-value`, `data-...-format-value`, `data-...-upload-url`, `data-live-ignore`.
- **22.** `EditorType` strictCapabilities option — throws `IncompatibleConfigException` if `assertCapabilities` warns + `strictCapabilities: true`.

### Tasks 23-25 — Doctrine custom Types

- **23.** `EditorContentHtmlType` (Doctrine `Type` extending `TextType`); convert to/from `HtmlContent`. Test using Doctrine `TestUtil`.
- **24.** `EditorContentBlocksType` (extends `JsonType`); convert to/from `BlockContent`.
- **25.** `EditorContentPageType` (extends `JsonType`); convert to/from `PageContent`.

### Tasks 26-30 — Upload pipeline

- **26.** `SignedUploadUrlGenerator` — HMAC sign+verify field+profile+expiry, tamper-detection test.
- **27.** `EditorUploadHandlerInterface` + `DefaultLocalUploadHandler` — mime+size validation, filesystem write.
- **28.** Upload handler registry (tagged-service iterator).
- **29.** `EditorUploadController` — functional test via `WebTestCase`: 200 happy, 403 bad-sig, 422 bad-mime, 500 handler throw.
- **30.** Wire route in `config/routes.php`; bundle Extension registers it.

### Tasks 31-34 — Live + Twig

- **31.** `LiveEditor` trait (uses `DefaultActionTrait`) — debounced `saveDraft` Live action + `lastSavedAt`/`dirty` state.
- **32.** `LiveEditor` integration test via in-test concrete LiveComponent.
- **33.** `EditorRenderExtension` (Twig function `ux_editor_render`) — branches on format; missing-renderer fallback (comment in prod, warning box in debug).
- **34.** `EditorRenderExtension` page-render: iframe srcdoc + sandbox attr.

### Tasks 35-40 — Tier 1 Wysiwyg

- **35.** `WysiwygCapabilities::default()` factory + `->with()` test.
- **36.** `AbstractWysiwygConfig` (locks `format=Html`, provides default `translateCommon` for portable WYSIWYG keys: placeholder, language, readOnly, plugins).
- **37.** `AbstractWysiwygTransformer` (`HtmlContent` <-> string, sanitizer wiring opt-in).
- **38.** `AbstractWysiwygBridge` (default capabilities = `WysiwygCapabilities::default()`, default content class = `HtmlContent`).
- **39.** Integration test: in-test concrete `FakeWysiwygBridge` registered with `BridgeRegistry`, used in `EditorType`, round-trips a string.
- **40.** Capability warning test: `FakeWysiwygBridge` with `supportsToolbar=false` + `CommonOptions(toolbar: [...])` → warning logged.

### Tasks 41-46 — Tier 1 Block

- **41.** `BlockCapabilities::default()` factory + test.
- **42.** `AbstractBlockConfig` (locks `format=Blocks`, default `translateCommon` for readOnly/autofocus/placeholder).
- **43.** `BlockRendererInterface` + `BlockRendererRegistry` (id-keyed render dispatch, missing-type fallback).
- **44.** `AbstractBlockTransformer` (`BlockContent` <-> array, `schemaVersion` preserved, malformed JSON throws `ContentSchemaException`).
- **45.** `AbstractBlockBridge`.
- **46.** Integration test: `FakeBlockBridge` round-trips `[{type, data}]` array through form.

### Tasks 47-52 — Tier 1 Page

- **47.** `PageCapabilities::default()` factory + test.
- **48.** `AbstractPageConfig` (locks `format=Page`, `translateCommon` minimal — page builders ignore most common options).
- **49.** `PageAssetExtractor` — walks `$components` array, dedupes asset URLs, returns asset list.
- **50.** `PageSandboxRenderer` — produces `<iframe srcdoc="..." sandbox="allow-same-origin">` with HTML+CSS inlined.
- **51.** `AbstractPageTransformer` — supports `StorageShape::Json` (bundled) AND `StorageShape::Split` (assumes embedded entity-side; transformer asserts shape).
- **52.** `AbstractPageBuilderBridge`.

### Tasks 53-60 — JS Tier 0 + Tier 1

- **53.** Vitest harness + minimal `controller.ts` skeleton; test that controller registers values + targets.
- **54.** `AbstractEditorController.connect()` — dispatches `ux:editor:pre-connect` then `ux:editor:connect`; calls abstract `createEditor()`.
- **55.** `AbstractEditorController.syncInput()` + serialize abstract; emits `ux:editor:change`.
- **56.** `AbstractEditorController.configValueChanged()` — diff computation, hot vs remount branch (via abstract `hotReloadable` getter).
- **57.** `AbstractEditorController.disconnect()` — calls abstract `destroyEditor()`, emits `ux:editor:destroy`.
- **58.** `SignedUploadClient` (POST multipart with signed URL, parses bridge-shaped JSON response).
- **59.** `live-editor.ts` — debounce `ux:editor:change` 800ms, dispatch Live action with field+content.
- **60.** Content JS mirrors (`EditorContent`, `HtmlContent`, `BlockContent`, `PageContent`) — simple data classes for Live prop hydration.

### Tasks 61-65 — JS Tier 1 format abstracts

- **61.** `AbstractWysiwygController` — `serialize` returns `getHTML()`; paste-sanitize hook; a11y attrs.
- **62.** `AbstractBlockController` — `serialize` awaits `editor.save()`; block-event normalization.
- **63.** `AbstractPageBuilderController` — `serialize` returns bundle `{html, css, components, assets}`; sandbox iframe lifecycle helpers.
- **64.** Format controller tests via in-test concrete subclasses (jsdom + stimulus test harness).
- **65.** Build TS → JS: `tsc -p src/Editor/tsconfig.json`; verify `dist/` populated.

### Tasks 66-72 — Bundle wiring

- **66.** `UXEditorBundle.php` (extends `AbstractBundle`).
- **67.** `UXEditorExtension` — loads `config/services.php`, registers tagged-service iterators for: bridges, presets, upload handlers, block renderers, content converters.
- **68.** `config/services.php` — service definitions, tag bindings, default sanitizer alias.
- **69.** Wire bundle config schema: `editor.html.sanitize_required` (bool, default `%kernel.debug%` inverted), `editor.upload.handler` (string, default `default_local`).
- **70.** Boot-time sanitizer check: if `sanitize_required=true` and no `HtmlSanitizerInterface` service → `RuntimeException` during compile pass.
- **71.** `bin/console debug:ux-editor` command — list registered bridges/presets/handlers/converters/renderers with format columns.
- **72.** WebProfiler data collector (skeleton; details collected per request: configs resolved, capability warnings, transformer roundtrips).

### Tasks 73-78 — Documentation + final checks

- **73.** `doc/index.rst` — Sphinx skeleton documenting `EditorType` + the three Tier 1 abstracts.
- **74.** `assets/controllers.json` — registers Tier 1 abstract controllers as `enabled: false` (Tier 2 bridges enable per-controller).
- **75.** Full PHPUnit run — green; coverage report ≥ 90% for Tier 0/1.
- **76.** Full Vitest run — green; coverage ≥ 85% for controller abstracts.
- **77.** `composer audit` + `npm audit` clean.
- **78.** `CHANGELOG.md` update; tag `v0.1.0` (annotated, no push).

---

## Plan continuation

Tasks 7-78 above are listed in summary form because writing all 78 tasks with full TDD detail in one document exceeds the output budget. To execute this plan, follow either path:

- **A (recommended) — Just-in-time expansion.** Use `superpowers:subagent-driven-development`. When the subagent reaches a summary task, expand it into the full 5-step TDD cycle using the spec + the patterns established in Tasks 1-6 above as templates. Code samples for each tier are in `docs/superpowers/specs/2026-05-19-ux-editor-design.md`.

- **B — Full expansion now.** Re-invoke `superpowers:writing-plans` with the prompt: "expand tasks N-M of `docs/superpowers/plans/2026-05-19-ux-editor-core.md`". The skill will append full TDD blocks for the requested range as a new file `docs/superpowers/plans/2026-05-19-ux-editor-core-tasks-N-M.md`.

Tasks 1-6 ARE fully written with code; they establish the file structure, scaffold, tests, and the first content value objects so any contributor can start immediately.

---

## Self-review

**1. Spec coverage:** every section of `docs/superpowers/specs/2026-05-19-ux-editor-design.md` has at least one task:
- §3 Architecture (three-tier) → Task 1 scaffold + Tasks 35-52 Tier 1 + bridge structure across all tasks.
- §4 Content model → Tasks 2-6 + §14 transformer mapped to Task 19 + cross-bridge converter Task 14.
- §5 Config layer → Tasks 8-13.
- §6 Distribution → Task 1 + Task 65 (TS build) + Task 74 (controllers.json).
- §7 Security (sanitization, upload) → Tasks 20 (sanitize wiring) + 26-30 (upload pipeline) + 70 (boot-time sanitizer check).
- §8 LiveComponent → Tasks 31-32 (LiveEditor trait) + 56 (configValueChanged) + 59 (autosave debouncer).
- §9 Twig render → Tasks 33-34.
- §10 Phasing → not implemented in code; lives in `ROADMAP-UX-EDITOR.md`. Acceptable.
- §11 Error handling → Task 7 exception hierarchy + every transformer task throws on malformed data.
- §12 Testing → tests embedded in every task; coverage checkpoints at Tasks 15, 75, 76.
- §13 Demo app → out of scope for this plan (Plan 5 territory). Acceptable.
- §14 Extension surfaces → tagged-service iterators wired in Tasks 13, 14, 28, 67.

**2. Placeholder scan:** Tasks 1-6 contain full code. Tasks 7-78 use summary form but each summary names exact files + the concrete behavior to test, not "TBD". The plan continuation section is explicit about how to expand them. No "TODO/FIXME/handle edge cases" handwaves.

**3. Type consistency:** `EditorContentInterface::getRaw()` returns `string|array` across all tasks — `HtmlContent` returns `string`, `BlockContent` and `PageContent` return `array`. `EditorContentTransformerInterface::transform/reverseTransform` consistent across Task 19 + Tasks 23-25 + Tier 1 transformers. `StorageShape` enum referenced in Task 19 used in Tasks 23-25, 51. `BridgeCapabilities` ctor signature stable across Tasks 9, 35, 41, 47.

No fixes needed inline.

---

## Execution handoff

Plan saved to `docs/superpowers/plans/2026-05-19-ux-editor-core.md` (this file).

Two execution options:

1. **Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration. Use `superpowers:subagent-driven-development`.
2. **Inline Execution** — execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Tasks 1-6 ready to execute now; Tasks 7-78 need just-in-time expansion (recommended path A in "Plan continuation" above), OR a follow-up writing-plans invocation to expand a chosen range fully.

Which approach?
