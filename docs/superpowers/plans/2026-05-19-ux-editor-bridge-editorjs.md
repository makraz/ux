# UX Editor — EditorJS Bridge Implementation Plan (Plan 2 / 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `symfony/ux-editor-editorjs` Tier 2 bridge — the first specific bridge for the `symfony/ux-editor` ecosystem. Provides `EditorJSBridge`, `EditorJSConfig`, EditorJS-specific Stimulus controller, default block renderers, and a service-registered preset.

**Architecture:** Tier 2 bridge sub-package depending on `symfony/ux-editor` (Tier 0 + Tier 1 from Plan 1). Inherits `AbstractBlockBridge` / `AbstractBlockConfig` / `AbstractBlockTransformer` / `AbstractBlockController`. Adds EditorJS-specific config fields (tools, defaultBlock, minHeight, logLevel), composer + npm sub-package, peerDep on `@editorjs/editorjs`, default block renderers, and the `blog.standard` preset.

**Tech Stack:** PHP 8.4, Symfony 7.4/8.0, Stimulus 3, TypeScript 5, Vitest, PHPUnit 11/12. New peerDeps: `@editorjs/editorjs ^2.30`.

**Depends on:** Plan 1 (`docs/superpowers/plans/2026-05-19-ux-editor-core.md`) fully merged.

**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](../specs/2026-05-19-ux-editor-design.md)
**Roadmap:** [`ROADMAP-UX-EDITOR.md`](../../../ROADMAP-UX-EDITOR.md) — EditorJS is the **first** v1.0 bridge (smallest, validates `BlockContent` end-to-end).

---

## File structure

```
src/Editor/src/Bridge/EditorJS/
├── composer.json
├── package.json
├── tsconfig.json
├── vitest.config.mjs
├── phpunit.dist.xml
├── playwright.config.ts
├── CHANGELOG.md
├── LICENSE
├── README.md
├── assets/
│   ├── controllers.json
│   └── src/
│       ├── controller.ts
│       └── tool-registry.ts
├── doc/
│   └── index.rst
├── config/
│   └── services.php
├── src/
│   ├── EditorJSBridge.php
│   ├── Config/
│   │   ├── EditorJSConfig.php
│   │   └── ToolDefinition.php
│   ├── Transformer/
│   │   └── EditorJSTransformer.php
│   ├── BlockRenderer/
│   │   ├── ParagraphRenderer.php
│   │   ├── HeaderRenderer.php
│   │   ├── ListRenderer.php
│   │   ├── ImageRenderer.php
│   │   └── QuoteRenderer.php
│   └── Preset/
│       └── BlogStandardPreset.php
└── tests/                                               # mirror of src/
```

Order: scaffold → config DTOs → bridge → transformer → block renderers → preset → service wiring → JS controller → integration → coverage → audit → docs → tag.

---

## Task 1 — Scaffold sub-package

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/{composer.json, package.json, tsconfig.json, vitest.config.mjs, phpunit.dist.xml, CHANGELOG.md, LICENSE, README.md, tests/.gitkeep}`

- [ ] **Step 1: Copy LICENSE**

```bash
cp src/Editor/LICENSE src/Editor/src/Bridge/EditorJS/LICENSE
```

- [ ] **Step 2: Write composer.json**

```json
{
    "name": "symfony/ux-editor-editorjs",
    "type": "symfony-bundle",
    "description": "EditorJS bridge for symfony/ux-editor",
    "keywords": ["symfony-ux", "editor", "editorjs", "block-editor"],
    "homepage": "https://symfony.com",
    "license": "MIT",
    "authors": [{"name": "Symfony Community", "homepage": "https://symfony.com/contributors"}],
    "autoload": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\EditorJS\\": "src/"}},
    "autoload-dev": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\EditorJS\\Tests\\": "tests/"}},
    "require": {
        "php": ">=8.4",
        "symfony/ux-editor": "^0.1|^1.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.1|^12.0",
        "symfony/framework-bundle": "^7.4|^8.0"
    },
    "extra": {"thanks": {"name": "symfony/ux", "url": "https://github.com/symfony/ux"}},
    "minimum-stability": "dev"
}
```

- [ ] **Step 3: Write package.json**

```json
{
    "name": "@symfony/ux-editor-editorjs",
    "description": "EditorJS bridge controller for symfony/ux-editor",
    "license": "MIT",
    "version": "0.1.0",
    "symfony": {
        "controllers": {
            "editorjs": {
                "main": "dist/controller.js",
                "fetch": "lazy",
                "enabled": true
            }
        },
        "importmap": {
            "@symfony/ux-editor": "^0.1.0",
            "@editorjs/editorjs": "^2.30.0"
        }
    },
    "type": "module",
    "main": "dist/controller.js",
    "types": "dist/controller.d.ts",
    "files": ["dist/"],
    "peerDependencies": {
        "@hotwired/stimulus": "^3.0.0",
        "@symfony/ux-editor": "^0.1.0",
        "@editorjs/editorjs": "^2.30.0"
    },
    "devDependencies": {
        "@hotwired/stimulus": "^3.2.2",
        "@editorjs/editorjs": "^2.30.0",
        "typescript": "^5.4.0",
        "vitest": "^1.5.0",
        "jsdom": "^24.0.0"
    }
}
```

- [ ] **Step 4: Write tsconfig.json**

```json
{
    "extends": "../../../../../tsconfig.package.json",
    "compilerOptions": {"outDir": "./dist", "rootDir": "./assets/src"},
    "include": ["assets/src/**/*"]
}
```

- [ ] **Step 5: Write vitest.config.mjs**

```js
import { defineConfig } from 'vitest/config';
import base from '../../../../../vitest.config.base.mjs';
export default defineConfig({ ...base, test: { ...base.test, environment: 'jsdom', include: ['assets/test/**/*.test.ts'] }});
```

- [ ] **Step 6: Write phpunit.dist.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/11.0/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true" failOnDeprecation="true" failOnNotice="true" failOnWarning="true">
    <php>
        <ini name="error_reporting" value="-1"/>
        <server name="APP_ENV" value="test" force="true"/>
    </php>
    <testsuites>
        <testsuite name="symfony/ux-editor-editorjs"><directory>./tests/</directory></testsuite>
    </testsuites>
    <source><include><directory>./src</directory></include></source>
</phpunit>
```

- [ ] **Step 7: Write README + CHANGELOG stubs**

`README.md`:
```markdown
# symfony/ux-editor-editorjs

EditorJS bridge for [symfony/ux-editor](https://github.com/symfony/ux-editor).

## Install

    composer require symfony/ux-editor-editorjs
    npm install @editorjs/editorjs

## Use

    $builder->add('body', EditorType::class, ['bridge' => 'editorjs']);

See doc/index.rst.
```

`CHANGELOG.md`:
```markdown
# CHANGELOG

## 0.1.0

- Initial release: EditorJS bridge for symfony/ux-editor.
```

- [ ] **Step 8: Smoke-run**

```bash
cd src/Editor/src/Bridge/EditorJS && composer install
vendor/bin/phpunit
```

Expected: 0 tests, exit 0.

- [ ] **Step 9: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/
git commit -m "feat(editor-editorjs): scaffold symfony/ux-editor-editorjs package"
```

---

## Task 2 — `ToolDefinition` DTO

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/Config/ToolDefinition.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/Config/ToolDefinitionTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\Config\ToolDefinition;

final class ToolDefinitionTest extends TestCase
{
    public function testDefaults(): void
    {
        $t = new ToolDefinition('Header');
        self::assertSame('Header', $t->class);
        self::assertSame([], $t->config);
        self::assertTrue($t->inlineToolbar);
        self::assertNull($t->shortcut);
    }

    public function testToArray(): void
    {
        $t = new ToolDefinition('Image', ['endpoints' => ['byFile' => '/u']], inlineToolbar: false, shortcut: 'CMD+I');
        self::assertSame([
            'class'         => 'Image',
            'inlineToolbar' => false,
            'config'        => ['endpoints' => ['byFile' => '/u']],
            'shortcut'      => 'CMD+I',
        ], $t->toArray());
    }

    public function testToArrayOmitsEmptyConfig(): void
    {
        $arr = (new ToolDefinition('Header'))->toArray();
        self::assertSame('Header', $arr['class']);
        self::assertArrayNotHasKey('config', $arr);
        self::assertArrayNotHasKey('shortcut', $arr);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Config;

final class ToolDefinition
{
    public function __construct(
        public readonly string  $class,
        public readonly array   $config = [],
        public readonly bool    $inlineToolbar = true,
        public readonly ?string $shortcut = null,
    ) {}

    public function toArray(): array
    {
        $out = ['class' => $this->class, 'inlineToolbar' => $this->inlineToolbar];
        if ($this->config !== [])      { $out['config']   = $this->config; }
        if ($this->shortcut !== null)  { $out['shortcut'] = $this->shortcut; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/Config/ToolDefinition.php src/Editor/src/Bridge/EditorJS/tests/Config/ToolDefinitionTest.php
git commit -m "feat(editor-editorjs): add ToolDefinition DTO"
```

---

## Task 3 — `EditorJSConfig`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/Config/EditorJSConfig.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/Config/EditorJSConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\BlockCapabilities;
use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\Config\ToolDefinition;
use Symfony\UX\Editor\Config\CommonOptions;

final class EditorJSConfigTest extends TestCase
{
    public function testBridgeIdAndCapabilities(): void
    {
        $cfg = new EditorJSConfig();
        self::assertSame('editorjs', $cfg->getBridgeId());
        self::assertEquals(BlockCapabilities::default(), $cfg->getCapabilities());
    }

    public function testTranslateOwn(): void
    {
        $cfg = new EditorJSConfig(
            common: new CommonOptions(placeholder: 'Write…'),
            tools: ['header' => new ToolDefinition('Header', ['levels' => [2,3,4]])],
            defaultBlock: 'paragraph',
            minHeight: 200,
            logLevel: 'WARN',
        );
        $native = $cfg->toNative();
        self::assertSame('Write…', $native['placeholder']);
        self::assertSame('Header', $native['tools']['header']['class']);
        self::assertSame('paragraph', $native['defaultBlock']);
        self::assertSame(200, $native['minHeight']);
        self::assertSame('WARN', $native['logLevel']);
    }

    public function testNoToolsKeyWhenEmpty(): void
    {
        self::assertArrayNotHasKey('tools', (new EditorJSConfig())->toNative());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Config;

use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockConfig;
use Symfony\UX\Editor\Config\CommonOptions;

final class EditorJSConfig extends AbstractBlockConfig
{
    /** @param array<string, ToolDefinition> $tools */
    public function __construct(
        CommonOptions $common = new CommonOptions(),
        public readonly array   $tools        = [],
        public readonly ?string $defaultBlock = 'paragraph',
        public readonly ?int    $minHeight    = null,
        public readonly ?string $logLevel     = null,
        array $nativeOverrides = [],
    ) {
        parent::__construct($common, $nativeOverrides);
    }

    public function getBridgeId(): string { return 'editorjs'; }

    protected function translateOwn(): array
    {
        $out = [];
        if ($this->tools !== []) {
            $out['tools'] = array_map(fn(ToolDefinition $t) => $t->toArray(), $this->tools);
        }
        if ($this->defaultBlock !== null) { $out['defaultBlock'] = $this->defaultBlock; }
        if ($this->minHeight !== null)    { $out['minHeight']    = $this->minHeight; }
        if ($this->logLevel !== null)     { $out['logLevel']     = $this->logLevel; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/Config/EditorJSConfig.php src/Editor/src/Bridge/EditorJS/tests/Config/EditorJSConfigTest.php
git commit -m "feat(editor-editorjs): add EditorJSConfig"
```

---

## Task 4 — `EditorJSTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/Transformer/EditorJSTransformer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/Transformer/EditorJSTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\Transformer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\Transformer\EditorJSTransformer;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class EditorJSTransformerTest extends TestCase
{
    public function testMetadata(): void
    {
        $t = new EditorJSTransformer();
        self::assertSame('editorjs', $t->getBridgeId());
        self::assertSame(BlockContent::class, $t->getContentClass());
        self::assertSame(StorageShape::Json, $t->getStorageShape());
    }

    public function testReverseStampsBridgeId(): void
    {
        $bc = (new EditorJSTransformer())->reverseTransform(['version' => '2.30.0', 'blocks' => [['type' => 'paragraph', 'data' => ['text' => 'hi']]]]);
        self::assertInstanceOf(BlockContent::class, $bc);
        self::assertSame('editorjs', $bc->getMetadata()['bridgeId']);
        self::assertSame('2.30.0', $bc->schemaVersion);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Transformer;

use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockTransformer;

final class EditorJSTransformer extends AbstractBlockTransformer
{
    public function getBridgeId(): string { return 'editorjs'; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/Transformer/EditorJSTransformer.php src/Editor/src/Bridge/EditorJS/tests/Transformer/EditorJSTransformerTest.php
git commit -m "feat(editor-editorjs): add EditorJSTransformer"
```

---

## Task 5 — `EditorJSBridge`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/EditorJSBridge.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/EditorJSBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\EditorJSBridge;
use Symfony\UX\Editor\Bridge\EditorJS\Transformer\EditorJSTransformer;
use Symfony\UX\Editor\Bridge\Format\Block\BlockCapabilities;

final class EditorJSBridgeTest extends TestCase
{
    public function testMetadata(): void
    {
        $b = new EditorJSBridge();
        self::assertSame('editorjs', $b->getId());
        self::assertSame('symfony--ux-editor--editorjs', $b->getControllerName());
        self::assertEquals(BlockCapabilities::default(), $b->getCapabilities());
        self::assertInstanceOf(EditorJSConfig::class, $b->getDefaultConfig());
        self::assertInstanceOf(EditorJSTransformer::class, $b->createTransformer());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS;

use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\Transformer\EditorJSTransformer;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockBridge;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class EditorJSBridge extends AbstractBlockBridge
{
    public function getId(): string { return 'editorjs'; }
    public function getDefaultConfig(): EditorConfigInterface { return new EditorJSConfig(); }
    public function createTransformer(): EditorContentTransformerInterface { return new EditorJSTransformer(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/EditorJSBridge.php src/Editor/src/Bridge/EditorJS/tests/EditorJSBridgeTest.php
git commit -m "feat(editor-editorjs): add EditorJSBridge"
```

---

## Task 6 — `ParagraphRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ParagraphRenderer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ParagraphRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\BlockRenderer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ParagraphRenderer;

final class ParagraphRendererTest extends TestCase
{
    public function testTypeAndRender(): void
    {
        $r = new ParagraphRenderer();
        self::assertSame('paragraph', $r->getBlockType());
        self::assertSame('<p>hello</p>', $r->render(['text' => 'hello']));
    }

    public function testEscapesHtml(): void
    {
        self::assertSame('<p>&lt;script&gt;x&lt;/script&gt;</p>', (new ParagraphRenderer())->render(['text' => '<script>x</script>']));
    }

    public function testEmptyTextEmptyParagraph(): void
    {
        self::assertSame('<p></p>', (new ParagraphRenderer())->render([]));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer;

use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;

final class ParagraphRenderer implements BlockRendererInterface
{
    public function getBlockType(): string { return 'paragraph'; }

    public function render(array $blockData, array $blockMeta = []): string
    {
        return '<p>'.htmlspecialchars((string)($blockData['text'] ?? ''), \ENT_QUOTES | \ENT_HTML5, 'UTF-8').'</p>';
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ParagraphRenderer.php src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ParagraphRendererTest.php
git commit -m "feat(editor-editorjs): add ParagraphRenderer"
```

---

## Task 7 — `HeaderRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/BlockRenderer/HeaderRenderer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/HeaderRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\BlockRenderer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\HeaderRenderer;

final class HeaderRendererTest extends TestCase
{
    public function testType(): void { self::assertSame('header', (new HeaderRenderer())->getBlockType()); }

    public function testRendersLevels(): void
    {
        $r = new HeaderRenderer();
        self::assertSame('<h2>Title</h2>', $r->render(['text' => 'Title', 'level' => 2]));
        self::assertSame('<h4>Sub</h4>',   $r->render(['text' => 'Sub',   'level' => 4]));
    }

    public function testClampsLevel(): void
    {
        $r = new HeaderRenderer();
        self::assertSame('<h2>X</h2>', $r->render(['text' => 'X', 'level' => 1]));
        self::assertSame('<h6>X</h6>', $r->render(['text' => 'X', 'level' => 9]));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer;

use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;

final class HeaderRenderer implements BlockRendererInterface
{
    public function getBlockType(): string { return 'header'; }

    public function render(array $blockData, array $blockMeta = []): string
    {
        $level = max(2, min(6, (int)($blockData['level'] ?? 2)));
        $text  = htmlspecialchars((string)($blockData['text'] ?? ''), \ENT_QUOTES | \ENT_HTML5, 'UTF-8');
        return sprintf('<h%1$d>%2$s</h%1$d>', $level, $text);
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/BlockRenderer/HeaderRenderer.php src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/HeaderRendererTest.php
git commit -m "feat(editor-editorjs): add HeaderRenderer"
```

---

## Task 8 — `ListRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ListRenderer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ListRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\BlockRenderer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ListRenderer;

final class ListRendererTest extends TestCase
{
    public function testTypeAndUnorderedDefault(): void
    {
        $r = new ListRenderer();
        self::assertSame('list', $r->getBlockType());
        self::assertSame('<ul><li>a</li><li>b</li></ul>', $r->render(['items' => ['a', 'b']]));
    }

    public function testOrdered(): void
    {
        self::assertSame('<ol><li>a</li></ol>', (new ListRenderer())->render(['style' => 'ordered', 'items' => ['a']]));
    }

    public function testEscapesItems(): void
    {
        self::assertSame('<ul><li>&lt;b&gt;x&lt;/b&gt;</li></ul>', (new ListRenderer())->render(['items' => ['<b>x</b>']]));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer;

use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;

final class ListRenderer implements BlockRendererInterface
{
    public function getBlockType(): string { return 'list'; }

    public function render(array $blockData, array $blockMeta = []): string
    {
        $tag = (($blockData['style'] ?? 'unordered') === 'ordered') ? 'ol' : 'ul';
        $items = '';
        foreach ((array)($blockData['items'] ?? []) as $i) {
            $items .= '<li>'.htmlspecialchars((string)$i, \ENT_QUOTES | \ENT_HTML5, 'UTF-8').'</li>';
        }
        return sprintf('<%1$s>%2$s</%1$s>', $tag, $items);
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ListRenderer.php src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ListRendererTest.php
git commit -m "feat(editor-editorjs): add ListRenderer"
```

---

## Task 9 — `ImageRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ImageRenderer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ImageRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\BlockRenderer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ImageRenderer;

final class ImageRendererTest extends TestCase
{
    public function testTypeAndUrl(): void
    {
        $r = new ImageRenderer();
        self::assertSame('image', $r->getBlockType());
        self::assertSame('<figure><img src="/x.png" alt=""></figure>', $r->render(['file' => ['url' => '/x.png']]));
    }

    public function testWithCaption(): void
    {
        self::assertSame(
            '<figure><img src="/x.png" alt="alt"><figcaption>cap</figcaption></figure>',
            (new ImageRenderer())->render(['file' => ['url' => '/x.png'], 'caption' => 'cap', 'alt' => 'alt'])
        );
    }

    public function testMissingUrlEmpty(): void
    {
        self::assertSame('', (new ImageRenderer())->render([]));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer;

use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;

final class ImageRenderer implements BlockRendererInterface
{
    public function getBlockType(): string { return 'image'; }

    public function render(array $blockData, array $blockMeta = []): string
    {
        $url = (string)($blockData['file']['url'] ?? '');
        if ($url === '') { return ''; }
        $alt = htmlspecialchars((string)($blockData['alt'] ?? ''), \ENT_QUOTES | \ENT_HTML5, 'UTF-8');
        $caption = $blockData['caption'] ?? null;
        $img = sprintf('<img src="%s" alt="%s">', htmlspecialchars($url, \ENT_QUOTES | \ENT_HTML5, 'UTF-8'), $alt);
        if (\is_string($caption) && $caption !== '') {
            return sprintf('<figure>%s<figcaption>%s</figcaption></figure>', $img, htmlspecialchars($caption, \ENT_QUOTES | \ENT_HTML5, 'UTF-8'));
        }
        return sprintf('<figure>%s</figure>', $img);
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/BlockRenderer/ImageRenderer.php src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/ImageRendererTest.php
git commit -m "feat(editor-editorjs): add ImageRenderer"
```

---

## Task 10 — `QuoteRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/BlockRenderer/QuoteRenderer.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/QuoteRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\BlockRenderer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\QuoteRenderer;

final class QuoteRendererTest extends TestCase
{
    public function testTypeAndBasic(): void
    {
        $r = new QuoteRenderer();
        self::assertSame('quote', $r->getBlockType());
        self::assertSame('<blockquote><p>Be water</p></blockquote>', $r->render(['text' => 'Be water']));
    }

    public function testWithCaption(): void
    {
        self::assertSame('<blockquote><p>Be water</p><cite>Bruce Lee</cite></blockquote>', (new QuoteRenderer())->render(['text' => 'Be water', 'caption' => 'Bruce Lee']));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer;

use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;

final class QuoteRenderer implements BlockRendererInterface
{
    public function getBlockType(): string { return 'quote'; }

    public function render(array $blockData, array $blockMeta = []): string
    {
        $text = htmlspecialchars((string)($blockData['text'] ?? ''), \ENT_QUOTES | \ENT_HTML5, 'UTF-8');
        $cap  = $blockData['caption'] ?? null;
        $body = sprintf('<p>%s</p>', $text);
        if (\is_string($cap) && $cap !== '') {
            $body .= sprintf('<cite>%s</cite>', htmlspecialchars($cap, \ENT_QUOTES | \ENT_HTML5, 'UTF-8'));
        }
        return sprintf('<blockquote>%s</blockquote>', $body);
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/BlockRenderer/QuoteRenderer.php src/Editor/src/Bridge/EditorJS/tests/BlockRenderer/QuoteRendererTest.php
git commit -m "feat(editor-editorjs): add QuoteRenderer"
```

---

## Task 11 — `BlogStandardPreset`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/src/Preset/BlogStandardPreset.php`
- Test: `src/Editor/src/Bridge/EditorJS/tests/Preset/BlogStandardPresetTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests\Preset;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\Preset\BlogStandardPreset;

final class BlogStandardPresetTest extends TestCase
{
    public function testBuildsEditorJSConfig(): void
    {
        $cfg = (new BlogStandardPreset())->build();
        self::assertInstanceOf(EditorJSConfig::class, $cfg);
        self::assertSame('editorjs', $cfg->getBridgeId());

        foreach (['paragraph','header','list','image','quote'] as $tool) {
            self::assertArrayHasKey($tool, $cfg->tools);
        }
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Preset;

use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\Config\ToolDefinition;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;

final class BlogStandardPreset implements EditorPresetInterface
{
    public function build(): EditorConfigInterface
    {
        return new EditorJSConfig(
            common: new CommonOptions(placeholder: 'Tell your story…'),
            tools: [
                'paragraph' => new ToolDefinition('Paragraph', ['preserveBlank' => true]),
                'header'    => new ToolDefinition('Header',    ['levels' => [2, 3, 4], 'defaultLevel' => 2]),
                'list'      => new ToolDefinition('List'),
                'image'     => new ToolDefinition('Image'),
                'quote'     => new ToolDefinition('Quote'),
            ],
        );
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/src/Preset/BlogStandardPreset.php src/Editor/src/Bridge/EditorJS/tests/Preset/BlogStandardPresetTest.php
git commit -m "feat(editor-editorjs): add blog.standard preset"
```

---

## Task 12 — Service wiring

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/config/services.php`

- [ ] **Step 1: Write file**

```php
<?php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\HeaderRenderer;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ImageRenderer;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ListRenderer;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\ParagraphRenderer;
use Symfony\UX\Editor\Bridge\EditorJS\BlockRenderer\QuoteRenderer;
use Symfony\UX\Editor\Bridge\EditorJS\EditorJSBridge;
use Symfony\UX\Editor\Bridge\EditorJS\Preset\BlogStandardPreset;

return static function (ContainerConfigurator $c): void {
    $s = $c->services()->defaults()->autowire()->autoconfigure();

    $s->set(EditorJSBridge::class)->tag('ux.editor.bridge');
    $s->set(BlogStandardPreset::class)->tag('ux.editor.preset', ['name' => 'blog.standard']);

    foreach ([ParagraphRenderer::class, HeaderRenderer::class, ListRenderer::class, ImageRenderer::class, QuoteRenderer::class] as $r) {
        $s->set($r)->tag('ux.editor.block_renderer');
    }
};
```

> No PHP extension class needed — services consumed via tagged-iterator registries declared in Plan 1 core `services.php`.

- [ ] **Step 2-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/config/services.php
git commit -m "feat(editor-editorjs): wire bridge + preset + block renderers"
```

---

## Task 13 — EditorJS Stimulus controller

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/assets/src/tool-registry.ts`
- Create: `src/Editor/src/Bridge/EditorJS/assets/src/controller.ts`
- Create: `src/Editor/src/Bridge/EditorJS/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (programmatic DOM construction, no `innerHTML`)**

```ts
import { describe, it, expect, vi } from 'vitest';
import { Application } from '@hotwired/stimulus';

vi.mock('@editorjs/editorjs', () => {
  return {
    default: class FakeEditorJS {
      constructor(public opts: any) {}
      isReady: Promise<void> = Promise.resolve();
      async save(): Promise<any> { return { version: '2.30.0', blocks: [{ type: 'paragraph', data: { text: 'hi' } }] }; }
      async destroy(): Promise<void> {}
    },
  };
});

function buildHost(): { root: HTMLElement; input: HTMLTextAreaElement; mount: HTMLElement } {
  const root = document.createElement('div');
  root.setAttribute('data-controller', 'editorjs');
  root.setAttribute('data-editorjs-config-value', '{"defaultBlock":"paragraph"}');
  root.setAttribute('data-editorjs-format-value', 'blocks');
  root.setAttribute('data-editorjs-bridge-id-value', 'editorjs');
  const input = document.createElement('textarea');
  input.setAttribute('data-editorjs-target', 'input');
  const mount = document.createElement('div');
  mount.setAttribute('data-editorjs-target', 'mount');
  root.append(input, mount);
  return { root, input, mount };
}

describe('EditorJSController', () => {
  it('mounts, dispatches connect, serialize returns save() payload', async () => {
    const { default: Controller } = await import('../src/controller.js');
    const { root } = buildHost();
    document.body.append(root);

    const events: string[] = [];
    ['ux:editor:pre-connect', 'ux:editor:connect'].forEach(n => root.addEventListener(n, () => events.push(n)));

    const app = Application.start();
    app.register('editorjs', Controller as any);
    await new Promise(r => setTimeout(r, 10));

    expect(events).toEqual(['ux:editor:pre-connect', 'ux:editor:connect']);
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(root, 'editorjs');
    const out: any = await ctrl.serialize(ctrl.instance);
    expect(out.blocks[0].type).toBe('paragraph');
    app.stop();
    root.remove();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`src/Editor/src/Bridge/EditorJS/assets/src/tool-registry.ts`:
```ts
// Maps tool class names (declared in EditorJSConfig::tools) to runtime tool classes.
// Apps register tool classes by name on window.UXEditorJSTools, or override resolveTool().
declare global {
  interface Window {
    UXEditorJSTools?: Record<string, any>;
  }
}

export function resolveTool(name: string): any {
  return (typeof window !== 'undefined' && window.UXEditorJSTools && window.UXEditorJSTools[name]) ?? undefined;
}
```

`src/Editor/src/Bridge/EditorJS/assets/src/controller.ts`:
```ts
import { AbstractBlockController, type BlockInstance } from '@symfony/ux-editor/format/block';
import { resolveTool } from './tool-registry.js';

export default class EditorJSController extends AbstractBlockController {
  static values = { ...(AbstractBlockController as any).values };

  async createEditor(mount: HTMLElement, config: Record<string, unknown>): Promise<BlockInstance> {
    const { default: EditorJS } = await import('@editorjs/editorjs');

    // Resolve declared tool class names to runtime classes.
    const tools = config.tools as Record<string, { class: string; config?: any; inlineToolbar?: boolean; shortcut?: string }> | undefined;
    const resolvedTools: Record<string, any> = {};
    if (tools) {
      for (const [name, spec] of Object.entries(tools)) {
        const klass = resolveTool(spec.class);
        if (!klass) {
          console.warn(`[ux-editor-editorjs] Tool class "${spec.class}" not registered on window.UXEditorJSTools — skipping`);
          continue;
        }
        resolvedTools[name] = { class: klass, config: spec.config ?? {}, inlineToolbar: spec.inlineToolbar ?? true, shortcut: spec.shortcut };
      }
    }

    const editor = new EditorJS({
      holder: mount,
      ...config,
      tools: resolvedTools,
    });
    await editor.isReady;
    return editor as unknown as BlockInstance;
  }

  hotReloadable(): Set<string> { return new Set(['readOnly', 'placeholder']); }

  async applyConfig(diff: Record<string, unknown>, instance: any): Promise<void> {
    if ('readOnly' in diff && typeof instance.readOnly?.toggle === 'function') {
      await instance.readOnly.toggle(Boolean(diff.readOnly));
    }
    // placeholder change → noop; EditorJS reads placeholder at mount only. Subsequent change falls through to the remount path.
  }

  async destroyEditor(instance: any): Promise<void> {
    if (typeof instance?.destroy === 'function') {
      await instance.destroy();
    }
  }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/assets/src/ src/Editor/src/Bridge/EditorJS/assets/test/controller.test.ts
git commit -m "feat(editor-editorjs): add EditorJSController + tool registry"
```

---

## Task 14 — `assets/controllers.json`

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/assets/controllers.json`

- [ ] **Step 1: Write file**

```json
{
    "controllers": {
        "@symfony/ux-editor-editorjs": {
            "editorjs": {
                "main": "dist/controller.js",
                "fetch": "lazy",
                "enabled": true,
                "name": "symfony--ux-editor--editorjs"
            }
        }
    },
    "entrypoints": []
}
```

- [ ] **Step 2: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('src/Editor/src/Bridge/EditorJS/assets/controllers.json', 'utf8'))" && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/assets/controllers.json
git commit -m "feat(editor-editorjs): register Stimulus controller (auto-enabled)"
```

---

## Task 15 — End-to-end PHPUnit integration

**Files:**
- Test: `src/Editor/src/Bridge/EditorJS/tests/EditorJSIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\EditorJS\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
use Symfony\UX\Editor\Bridge\EditorJS\Config\ToolDefinition;
use Symfony\UX\Editor\Bridge\EditorJS\EditorJSBridge;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Form\EditorType;

final class EditorJSIntegrationTest extends TestCase
{
    public function testRoundTripThroughEditorType(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new EditorJSBridge()]), new PresetRegistry([])))
            ->getFormFactory();

        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config' => new EditorJSConfig(tools: ['paragraph' => new ToolDefinition('Paragraph')]),
        ])->getForm();

        $payload = json_encode(['version' => '2.30.0', 'blocks' => [['type' => 'paragraph', 'data' => ['text' => 'hi']]]], \JSON_THROW_ON_ERROR);
        $form->submit(['body' => $payload]);
        self::assertTrue($form->isSynchronized());

        $data = $form->get('body')->getData();
        self::assertInstanceOf(BlockContent::class, $data);
        self::assertSame('editorjs', $data->getMetadata()['bridgeId']);
        self::assertSame('paragraph', $data->blocks[0]['type']);

        $attr = $form->get('body')->createView()->vars['attr'];
        self::assertStringContainsString('symfony--ux-editor--editorjs', $attr['data-controller']);
        $native = json_decode($attr['data-symfony--ux-editor--editorjs-config-value'], true);
        self::assertSame('Paragraph', $native['tools']['paragraph']['class']);
    }
}
```

- [ ] **Step 2: Run — expect 1 pass** (Tier 0 + Tier 1 wired in Plan 1; this verifies the Tier 2 bridge integrates cleanly).

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/tests/EditorJSIntegrationTest.php
git commit -m "test(editor-editorjs): end-to-end through EditorType"
```

---

## Task 16 — Playwright E2E specs

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/playwright.config.ts`
- Create: `src/Editor/src/Bridge/EditorJS/tests/e2e/mounts.spec.ts`
- Create: `src/Editor/src/Bridge/EditorJS/tests/e2e/serialize-roundtrip.spec.ts`

> These specs depend on demo routes in Plan 5. Added now to lock the contract — Plan 5 will make them runnable.

- [ ] **Step 1: Playwright config**

```ts
// src/Editor/src/Bridge/EditorJS/playwright.config.ts
import { defineConfig } from '@playwright/test';
import base from '../../../../../playwright.config.base';

export default defineConfig({
  ...base,
  testDir: './tests/e2e',
  projects: [{ name: 'editorjs', testMatch: '**/*.spec.ts' }],
});
```

- [ ] **Step 2: `mounts.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('EditorJS controller mounts and emits connect', async ({ page }) => {
  await page.goto('/editor/block');
  const root = page.locator('[data-controller~="symfony--ux-editor--editorjs"]');
  await expect(root).toBeVisible();
  await expect(page.locator('[data-test-id="editor-state"]')).toHaveText('connected');
});
```

- [ ] **Step 3: `serialize-roundtrip.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('typing + submit round-trips through PHP form to BlockContent', async ({ page }) => {
  await page.goto('/editor/block');
  await page.locator('[data-test-id="editorjs-canvas"] .ce-paragraph').first().click();
  await page.keyboard.type('hello world');
  await page.locator('[data-test-id="submit-button"]').click();
  const dump = await page.locator('[data-test-id="entity-dump"]').textContent();
  expect(dump).toContain('"type":"paragraph"');
  expect(dump).toContain('"text":"hello world"');
});
```

- [ ] **Step 4: Verify spec listing (Playwright requires demo app to actually run; listing only)**

```bash
cd src/Editor/src/Bridge/EditorJS && npx playwright test --list 2>&1 | head -20
```

Expected: lists `mounts.spec.ts`, `serialize-roundtrip.spec.ts`.

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/playwright.config.ts src/Editor/src/Bridge/EditorJS/tests/e2e/
git commit -m "test(editor-editorjs): add Playwright specs (runs after Plan 5)"
```

---

## Task 17 — TypeScript build

- [ ] **Step 1: Build**

```bash
cd src/Editor/src/Bridge/EditorJS && npx tsc
```

Expected: zero errors; `dist/` populated.

- [ ] **Step 2: Smoke import**

```bash
node -e "import('./dist/controller.js').then(m => console.log(typeof m.default))"
```

Expected: `function`.

- [ ] **Step 3-5: N/A** (no source change unless `tsc` fails — fix tsconfig in single commit if so).

---

## Task 18 — Sphinx docs

**Files:**
- Create: `src/Editor/src/Bridge/EditorJS/doc/index.rst`

- [ ] **Step 1: Write file**

```rst
EditorJS Bridge
===============

`symfony/ux-editor-editorjs`_ integrates `EditorJS`_ with ``symfony/ux-editor``.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/ux-editor-editorjs

Then add the upstream EditorJS library. With AssetMapper:

.. code-block:: terminal

    $ php bin/console importmap:require @editorjs/editorjs

Or with Webpack Encore:

.. code-block:: terminal

    $ npm install @editorjs/editorjs

Optional EditorJS tool packages (apps install the tools they need and register classes on
``window.UXEditorJSTools`` so the controller can resolve them):

.. code-block:: terminal

    $ npm install @editorjs/header @editorjs/list @editorjs/image @editorjs/quote

Use
---

.. code-block:: php

    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\UX\Editor\Bridge\EditorJS\Config\EditorJSConfig;
    use Symfony\UX\Editor\Bridge\EditorJS\Config\ToolDefinition;
    use Symfony\UX\Editor\Config\CommonOptions;
    use Symfony\UX\Editor\Form\EditorType;

    final class ArticleType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $b, array $options): void
        {
            $b->add('body', EditorType::class, [
                'config' => new EditorJSConfig(
                    common: new CommonOptions(placeholder: 'Tell your story…'),
                    tools: [
                        'header'    => new ToolDefinition('Header',    ['levels' => [2, 3, 4]]),
                        'paragraph' => new ToolDefinition('Paragraph', ['preserveBlank' => true]),
                        'list'      => new ToolDefinition('List'),
                        'image'     => new ToolDefinition('Image',     ['endpoints' => ['byFile' => '/_ux_editor/upload/body']]),
                        'quote'     => new ToolDefinition('Quote'),
                    ],
                ),
            ]);
        }
    }

Or use the shipped preset:

.. code-block:: php

    $b->add('body', EditorType::class, ['preset' => 'blog.standard']);

Render saved content
--------------------

.. code-block:: twig

    {{ ux_editor_render(article.body) }}

Five block renderers ship with the bridge (paragraph, header, list, image, quote). Override
or add renderers by tagging your service with ``ux.editor.block_renderer``.

Tool registration on the JS side
--------------------------------

The bridge controller resolves tool class names listed in ``EditorJSConfig::tools`` against
``window.UXEditorJSTools``. Register tools in your app entrypoint:

.. code-block:: js

    import Header from '@editorjs/header';
    import List   from '@editorjs/list';
    import Image  from '@editorjs/image';
    import Quote  from '@editorjs/quote';

    window.UXEditorJSTools = { Header, List, Image, Quote };

Unregistered tool names are skipped with a console warning.

.. _`symfony/ux-editor-editorjs`: https://github.com/symfony/ux
.. _`EditorJS`: https://editorjs.io
```

- [ ] **Step 2: Verify non-empty**

```bash
test -s src/Editor/src/Bridge/EditorJS/doc/index.rst && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/EditorJS/doc/index.rst
git commit -m "docs(editor-editorjs): add Sphinx index.rst"
```

---

## Task 19 — `splitsh.json` update

**Files:**
- Modify: `splitsh.json` at repo root

- [ ] **Step 1: Inspect existing entries first** to match the exact shape:

```bash
head -40 splitsh.json
```

- [ ] **Step 2: Edit `splitsh.json` — add entry for the bridge** (shape matches existing entries; the example below assumes `{name, directory, target}` keys — adjust to match observed shape):

```json
{
    "name": "ux-editor-editorjs",
    "directory": "src/Editor/src/Bridge/EditorJS",
    "target": "https://github.com/symfony/ux-editor-editorjs.git"
}
```

- [ ] **Step 3: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('splitsh.json', 'utf8'))" && echo OK
```

- [ ] **Step 4: N/A**

- [ ] **Step 5: Commit**

```bash
git add splitsh.json
git commit -m "chore(editor-editorjs): split symfony/ux-editor-editorjs to its own repo"
```

---

## Task 20 — Coverage gates

- [ ] **Step 1: PHPUnit ≥ 75% Tier 2**

```bash
cd src/Editor/src/Bridge/EditorJS && XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text
```

Expected: ≥ 75% line coverage on Tier 2 sources.

- [ ] **Step 2: Vitest ≥ 75%**

```bash
npx vitest run --coverage
```

Expected: ≥ 75% line coverage on `assets/src/controller.ts`.

- [ ] **Step 3: If under threshold — add focused tests + re-run**

Same iteration pattern as Plan 1 coverage gates. Commit per added test.

- [ ] **Step 4-5: N/A**

---

## Task 21 — Audit

- [ ] **Step 1: PHP**

```bash
cd src/Editor/src/Bridge/EditorJS && composer audit
```

Expected: zero advisories.

- [ ] **Step 2: NPM**

```bash
npm audit --omit=dev
```

Expected: zero advisories.

- [ ] **Step 3-5: N/A** (bump pinned versions on advisory as in Plan 1).

---

## Task 22 — Finalize CHANGELOG + annotated tag

**Files:**
- Modify: `src/Editor/src/Bridge/EditorJS/CHANGELOG.md`

- [ ] **Step 1: Replace content**

```markdown
# CHANGELOG

## 0.1.0 — 2026-05-19

Initial release: EditorJS bridge for `symfony/ux-editor`.

### Added

- `EditorJSBridge` (extends `AbstractBlockBridge`).
- `EditorJSConfig` (extends `AbstractBlockConfig`) with `tools`, `defaultBlock`, `minHeight`, `logLevel`.
- `ToolDefinition` DTO for typed EditorJS tool config.
- `EditorJSTransformer` (extends `AbstractBlockTransformer`).
- Five default block renderers: `ParagraphRenderer`, `HeaderRenderer`, `ListRenderer`, `ImageRenderer`, `QuoteRenderer`.
- `BlogStandardPreset` (`name: blog.standard`).
- Stimulus controller `symfony--ux-editor--editorjs` with hot config reload for `readOnly` + `placeholder`,
  dynamic import of `@editorjs/editorjs`, tool class resolution via `window.UXEditorJSTools`.

### Requires

- `symfony/ux-editor` ^0.1.
- `@editorjs/editorjs` ^2.30 (peer dep).
```

- [ ] **Step 2: Commit CHANGELOG**

```bash
git add src/Editor/src/Bridge/EditorJS/CHANGELOG.md
git commit -m "docs(editor-editorjs): finalize CHANGELOG 0.1.0"
```

- [ ] **Step 3: Verify clean working tree**

```bash
git status
```

Expected: clean.

- [ ] **Step 4: Annotated tag (local only)**

```bash
git tag -a editor-editorjs/v0.1.0 -m "symfony/ux-editor-editorjs 0.1.0"
```

Do not push.

- [ ] **Step 5: Verify**

```bash
git tag -l 'editor-editorjs/v0.1.0'
```

Expected: `editor-editorjs/v0.1.0`.

---

## Final checkpoint

- [ ] **Run all tests**

```bash
cd src/Editor/src/Bridge/EditorJS && vendor/bin/phpunit && npx vitest run
```

Expected: both green. Playwright specs skipped (runnable after Plan 5).

- [ ] **Coverage**

```bash
XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text
npx vitest run --coverage
```

Expected: ≥ 75% Tier 2 lines (PHP + JS).

- [ ] **Audit**

```bash
composer audit && npm audit --omit=dev
```

Expected: zero advisories.

---

## Self-review for Plan 2

**1. Spec coverage:**
- §3 Tier 2 EditorJS bridge structure → Tasks 1, 5.
- §4.3 EditorJS `BlockContent` transform with metadata stamping → Tasks 4, 15.
- §5 Typed config + native overrides → Tasks 2, 3.
- §5.5 Service-registered preset → Task 11.
- §9 Twig block rendering with default renderers → Tasks 6-10.
- §6 npm sub-package + peerDeps + dynamic import + AssetMapper/Encore both supported → Tasks 1, 13, 14.
- §8 Hot-reload of `readOnly` and `placeholder` → Task 13.
- §10 Phasing: EditorJS is the first Tier 2 bridge in v1 — Task 1 (composer pkg name), Task 22 (tag).
- §12 Testing: PHPUnit Tier 2 ≥ 75%, Vitest ≥ 75%, Playwright deferred to Plan 5 — Tasks 13, 15, 16, 20.

**2. Placeholder scan:**
- Task 16 explicitly notes Playwright specs depend on Plan 5 demo routes — bounded dependency, not a TODO.
- Task 17 is a build verification (no source change unless fail) — bounded.
- Task 19 instructs to inspect `splitsh.json` shape first — bounded (adapt-to-existing), not handwave.
- Task 20 uses the same bounded "if under threshold — add tests" pattern as Plan 1.
- No `TBD`/`FIXME`/`handle errors`/etc.

**3. Type consistency:**
- `EditorJSBridge` extends `AbstractBlockBridge` (Plan 1 Task 45).
- `EditorJSConfig` extends `AbstractBlockConfig` (Plan 1 Task 42); reuses `CommonOptions` (Plan 1 Task 8).
- `EditorJSTransformer` extends `AbstractBlockTransformer` (Plan 1 Task 44).
- All block renderers implement `BlockRendererInterface` (Plan 1 Task 33).
- Service tags match Plan 1's `services.php`: `ux.editor.bridge`, `ux.editor.preset`, `ux.editor.block_renderer`.
- Stimulus controller id format `symfony--ux-editor--editorjs` matches `AbstractBridge::getControllerName()` derivation (Plan 1 Task 17).
- JS controller imports from `@symfony/ux-editor/format/block` — matches `exports` declared in Plan 1 `package.json` (Task 1).

No inline fixes needed.

---

## Execution handoff

Plan 2 saved to `docs/superpowers/plans/2026-05-19-ux-editor-bridge-editorjs.md`.

Two execution options:

1. **Subagent-Driven (recommended)** — `superpowers:subagent-driven-development` dispatches one subagent per task.
2. **Inline Execution** — `superpowers:executing-plans` walks all 22 tasks with checkpoints after Tasks 5 (bridge core), 10 (block renderers), 15 (end-to-end PHPUnit), 22 (release).

Plans 3-5 follow once Plan 2 lands:
- Plan 3: CKEditor bridge (Tier 2 Wysiwyg).
- Plan 4: GrapesJS bridge (Tier 2 Page).
- Plan 5: Demo app on `ux.symfony.com` + LiveComponent showcase (depends on Plans 1-4).

Which approach for Plan 2?
