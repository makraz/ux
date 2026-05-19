# UX Editor — GrapesJS Bridge Implementation Plan (Plan 4 / 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `symfony/ux-editor-grapesjs` — third Tier 2 bridge, only Page-format bridge in v1.0. Provides `GrapesJSBridge`, `GrapesJSConfig` (typed fields for components, blocks, storageManager, deviceManager, canvasCss), GrapesJS Stimulus controller, service-registered preset.

**Architecture:** Tier 2 bridge depending on `symfony/ux-editor`. Inherits `AbstractPageBuilderBridge` / `AbstractPageConfig` / `AbstractPageTransformer` / `AbstractPageBuilderController`. Adds GrapesJS-specific config, composer + npm sub-package, peerDep on `grapesjs`.

**Tech Stack:** Same as Plans 1-3. New peerDep: `grapesjs ^0.21`.

**Depends on:** Plan 1. Independent of Plans 2-3 (parallelizable).

**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](../specs/2026-05-19-ux-editor-design.md)
**Roadmap:** [`ROADMAP-UX-EDITOR.md`](../../../ROADMAP-UX-EDITOR.md) — GrapesJS is the **third** v1.0 bridge; finalizes `PageContent` end-to-end.

---

## File structure

```
src/Editor/src/Bridge/GrapesJS/
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
│   └── src/controller.ts
├── doc/index.rst
├── config/services.php
├── src/
│   ├── GrapesJSBridge.php
│   ├── Config/GrapesJSConfig.php
│   ├── Transformer/GrapesJSTransformer.php
│   └── Preset/PageBuilderLandingPreset.php
└── tests/                                # mirror of src/
```

Order: scaffold → config → bridge → transformer → preset → service wiring → JS controller → integration → coverage → audit → docs → tag.

---

## Task 1 — Scaffold sub-package

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/{composer.json, package.json, tsconfig.json, vitest.config.mjs, phpunit.dist.xml, CHANGELOG.md, LICENSE, README.md, tests/.gitkeep}`

- [ ] **Step 1: Copy LICENSE**

```bash
cp src/Editor/LICENSE src/Editor/src/Bridge/GrapesJS/LICENSE
```

- [ ] **Step 2: Write composer.json**

```json
{
    "name": "symfony/ux-editor-grapesjs",
    "type": "symfony-bundle",
    "description": "GrapesJS bridge for symfony/ux-editor",
    "keywords": ["symfony-ux", "editor", "grapesjs", "page-builder"],
    "homepage": "https://symfony.com",
    "license": "MIT",
    "authors": [{"name": "Symfony Community", "homepage": "https://symfony.com/contributors"}],
    "autoload": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\GrapesJS\\": "src/"}},
    "autoload-dev": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\GrapesJS\\Tests\\": "tests/"}},
    "require": {"php": ">=8.4", "symfony/ux-editor": "^0.1|^1.0"},
    "require-dev": {"phpunit/phpunit": "^11.1|^12.0", "symfony/framework-bundle": "^7.4|^8.0"},
    "extra": {"thanks": {"name": "symfony/ux", "url": "https://github.com/symfony/ux"}},
    "minimum-stability": "dev"
}
```

- [ ] **Step 3: Write package.json**

```json
{
    "name": "@symfony/ux-editor-grapesjs",
    "description": "GrapesJS bridge controller for symfony/ux-editor",
    "license": "MIT",
    "version": "0.1.0",
    "symfony": {
        "controllers": {"grapesjs": {"main": "dist/controller.js", "fetch": "lazy", "enabled": true}},
        "importmap": {"@symfony/ux-editor": "^0.1.0", "grapesjs": "^0.21.0"}
    },
    "type": "module",
    "main": "dist/controller.js",
    "types": "dist/controller.d.ts",
    "files": ["dist/"],
    "peerDependencies": {
        "@hotwired/stimulus": "^3.0.0",
        "@symfony/ux-editor": "^0.1.0",
        "grapesjs": "^0.21.0"
    },
    "devDependencies": {
        "@hotwired/stimulus": "^3.2.2",
        "grapesjs": "^0.21.0",
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
        <testsuite name="symfony/ux-editor-grapesjs"><directory>./tests/</directory></testsuite>
    </testsuites>
    <source><include><directory>./src</directory></include></source>
</phpunit>
```

- [ ] **Step 7: Write README + CHANGELOG stubs**

`README.md`:
```markdown
# symfony/ux-editor-grapesjs

GrapesJS bridge for [symfony/ux-editor](https://github.com/symfony/ux-editor).

## Install

    composer require symfony/ux-editor-grapesjs
    npm install grapesjs

## Use

    $builder->add('body', EditorType::class, ['bridge' => 'grapesjs']);

See doc/index.rst.
```

`CHANGELOG.md`:
```markdown
# CHANGELOG

## 0.1.0

- Initial release: GrapesJS bridge for symfony/ux-editor.
```

- [ ] **Step 8: Smoke-run**

```bash
cd src/Editor/src/Bridge/GrapesJS && composer install
vendor/bin/phpunit
```

Expected: 0 tests, exit 0.

- [ ] **Step 9: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/
git commit -m "feat(editor-grapesjs): scaffold symfony/ux-editor-grapesjs package"
```

---

## Task 2 — `GrapesJSConfig`

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/src/Config/GrapesJSConfig.php`
- Test: `src/Editor/src/Bridge/GrapesJS/tests/Config/GrapesJSConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\PageCapabilities;
use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Config\CommonOptions;

final class GrapesJSConfigTest extends TestCase
{
    public function testBridgeIdAndCapabilities(): void
    {
        $cfg = new GrapesJSConfig();
        self::assertSame('grapesjs', $cfg->getBridgeId());
        self::assertEquals(PageCapabilities::default(), $cfg->getCapabilities());
    }

    public function testDefaultStorageManagerNone(): void
    {
        self::assertSame(['type' => 'none'], (new GrapesJSConfig())->toNative()['storageManager']);
    }

    public function testTranslateOwn(): void
    {
        $cfg = new GrapesJSConfig(
            common: new CommonOptions(theme: 'dark', language: 'fr'),
            components: [['type' => 'image', 'tagName' => 'img']],
            blocks:     [['id' => 'h1-block', 'label' => 'Heading', 'content' => '<h1>Heading</h1>']],
            storageManager: ['type' => 'local', 'autosave' => true],
            deviceManager: ['devices' => [['name' => 'Desktop', 'width' => '']]],
            canvasCss: 'body{margin:0}',
        );
        $n = $cfg->toNative();
        self::assertSame('dark', $n['theme']);
        self::assertSame('fr',   $n['language']);
        self::assertCount(1, $n['components']);
        self::assertCount(1, $n['blocks']);
        self::assertSame(['type' => 'local', 'autosave' => true], $n['storageManager']);
        self::assertArrayHasKey('deviceManager', $n);
        self::assertSame('body{margin:0}', $n['canvas']['styles'][0]);
    }

    public function testNativeOverridesWinLast(): void
    {
        $cfg = new GrapesJSConfig(
            storageManager: ['type' => 'local'],
            nativeOverrides: ['storageManager' => ['type' => 'remote']],
        );
        self::assertSame(['type' => 'remote'], $cfg->toNative()['storageManager']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Config;

use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageConfig;
use Symfony\UX\Editor\Config\CommonOptions;

final class GrapesJSConfig extends AbstractPageConfig
{
    public function __construct(
        CommonOptions $common = new CommonOptions(),
        public readonly array  $components     = [],
        public readonly array  $blocks         = [],
        public readonly array  $storageManager = ['type' => 'none'],
        public readonly array  $deviceManager  = [],
        public readonly ?string $canvasCss     = null,
        array $nativeOverrides = [],
    ) {
        parent::__construct($common, $nativeOverrides);
    }

    public function getBridgeId(): string { return 'grapesjs'; }

    protected function translateOwn(): array
    {
        $out = [];
        if ($this->components !== [])     { $out['components']    = $this->components; }
        if ($this->blocks !== [])         { $out['blocks']        = $this->blocks; }
        $out['storageManager']            = $this->storageManager;
        if ($this->deviceManager !== [])  { $out['deviceManager'] = $this->deviceManager; }
        if ($this->canvasCss !== null)    { $out['canvas']        = ['styles' => [$this->canvasCss]]; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/src/Config/GrapesJSConfig.php src/Editor/src/Bridge/GrapesJS/tests/Config/GrapesJSConfigTest.php
git commit -m "feat(editor-grapesjs): add GrapesJSConfig"
```

---

## Task 3 — `GrapesJSTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/src/Transformer/GrapesJSTransformer.php`
- Test: `src/Editor/src/Bridge/GrapesJS/tests/Transformer/GrapesJSTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Tests\Transformer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\GrapesJS\Transformer\GrapesJSTransformer;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class GrapesJSTransformerTest extends TestCase
{
    public function testMetadata(): void
    {
        $t = new GrapesJSTransformer();
        self::assertSame('grapesjs', $t->getBridgeId());
        self::assertSame(PageContent::class, $t->getContentClass());
        self::assertSame(StorageShape::Json, $t->getStorageShape());
    }

    public function testReverseStampsBridgeId(): void
    {
        $pc = (new GrapesJSTransformer())->reverseTransform([
            'html' => '<h1>x</h1>',
            'css'  => 'h1{}',
            'assets' => [],
            'components' => [['type' => 'h1']],
        ]);
        self::assertInstanceOf(PageContent::class, $pc);
        self::assertSame('<h1>x</h1>', $pc->html);
        self::assertSame('grapesjs', $pc->getMetadata()['bridgeId']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Transformer;

use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageTransformer;

final class GrapesJSTransformer extends AbstractPageTransformer
{
    public function getBridgeId(): string { return 'grapesjs'; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/src/Transformer/GrapesJSTransformer.php src/Editor/src/Bridge/GrapesJS/tests/Transformer/GrapesJSTransformerTest.php
git commit -m "feat(editor-grapesjs): add GrapesJSTransformer"
```

---

## Task 4 — `GrapesJSBridge`

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/src/GrapesJSBridge.php`
- Test: `src/Editor/src/Bridge/GrapesJS/tests/GrapesJSBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\PageCapabilities;
use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Bridge\GrapesJS\GrapesJSBridge;
use Symfony\UX\Editor\Bridge\GrapesJS\Transformer\GrapesJSTransformer;

final class GrapesJSBridgeTest extends TestCase
{
    public function testMetadata(): void
    {
        $b = new GrapesJSBridge();
        self::assertSame('grapesjs', $b->getId());
        self::assertSame('symfony--ux-editor--grapesjs', $b->getControllerName());
        self::assertEquals(PageCapabilities::default(), $b->getCapabilities());
        self::assertInstanceOf(GrapesJSConfig::class, $b->getDefaultConfig());
        self::assertInstanceOf(GrapesJSTransformer::class, $b->createTransformer());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS;

use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageBuilderBridge;
use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Bridge\GrapesJS\Transformer\GrapesJSTransformer;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class GrapesJSBridge extends AbstractPageBuilderBridge
{
    public function getId(): string { return 'grapesjs'; }
    public function getDefaultConfig(): EditorConfigInterface { return new GrapesJSConfig(); }
    public function createTransformer(): EditorContentTransformerInterface { return new GrapesJSTransformer(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/src/GrapesJSBridge.php src/Editor/src/Bridge/GrapesJS/tests/GrapesJSBridgeTest.php
git commit -m "feat(editor-grapesjs): add GrapesJSBridge"
```

---

## Task 5 — `PageBuilderLandingPreset`

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/src/Preset/PageBuilderLandingPreset.php`
- Test: `src/Editor/src/Bridge/GrapesJS/tests/Preset/PageBuilderLandingPresetTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Tests\Preset;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Bridge\GrapesJS\Preset\PageBuilderLandingPreset;

final class PageBuilderLandingPresetTest extends TestCase
{
    public function testBuilds(): void
    {
        $cfg = (new PageBuilderLandingPreset())->build();
        self::assertInstanceOf(GrapesJSConfig::class, $cfg);
        self::assertSame('grapesjs', $cfg->getBridgeId());
        self::assertNotEmpty($cfg->blocks);
        self::assertNotEmpty($cfg->deviceManager);
    }

    public function testIncludesCommonBlocks(): void
    {
        $cfg = (new PageBuilderLandingPreset())->build();
        $ids = array_map(fn($b) => $b['id'], $cfg->blocks);
        foreach (['hero', 'section', 'text', 'image'] as $expected) {
            self::assertContains($expected, $ids);
        }
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Preset;

use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;

final class PageBuilderLandingPreset implements EditorPresetInterface
{
    public function build(): EditorConfigInterface
    {
        return new GrapesJSConfig(
            common: new CommonOptions(language: 'en'),
            blocks: [
                ['id' => 'hero',    'label' => 'Hero',    'category' => 'Layout', 'content' => '<section class="hero"><h1>Hero title</h1><p>Hero subtitle</p></section>'],
                ['id' => 'section', 'label' => 'Section', 'category' => 'Layout', 'content' => '<section><div class="container"><h2>Section</h2></div></section>'],
                ['id' => 'text',    'label' => 'Text',    'category' => 'Basic',  'content' => '<p>Insert your text here.</p>'],
                ['id' => 'image',   'label' => 'Image',   'category' => 'Basic',  'content' => '<img src="" alt="">'],
            ],
            deviceManager: [
                'devices' => [
                    ['name' => 'Desktop', 'width' => ''],
                    ['name' => 'Tablet',  'width' => '768px', 'widthMedia' => '992px'],
                    ['name' => 'Mobile',  'width' => '320px', 'widthMedia' => '480px'],
                ],
            ],
        );
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/src/Preset/PageBuilderLandingPreset.php src/Editor/src/Bridge/GrapesJS/tests/Preset/PageBuilderLandingPresetTest.php
git commit -m "feat(editor-grapesjs): add page_builder.landing preset"
```

---

## Task 6 — Service wiring

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/config/services.php`

- [ ] **Step 1: Write file**

```php
<?php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\UX\Editor\Bridge\GrapesJS\GrapesJSBridge;
use Symfony\UX\Editor\Bridge\GrapesJS\Preset\PageBuilderLandingPreset;

return static function (ContainerConfigurator $c): void {
    $s = $c->services()->defaults()->autowire()->autoconfigure();

    $s->set(GrapesJSBridge::class)->tag('ux.editor.bridge');
    $s->set(PageBuilderLandingPreset::class)->tag('ux.editor.preset', ['name' => 'page_builder.landing']);
};
```

- [ ] **Step 2-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/config/services.php
git commit -m "feat(editor-grapesjs): wire bridge + preset"
```

---

## Task 7 — GrapesJS Stimulus controller

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/assets/src/controller.ts`
- Create: `src/Editor/src/Bridge/GrapesJS/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (programmatic DOM)**

```ts
import { describe, it, expect, vi } from 'vitest';
import { Application } from '@hotwired/stimulus';

vi.mock('grapesjs', () => {
  return {
    default: {
      init(opts: any): any {
        return {
          opts,
          getHtml: () => '<h1>X</h1>',
          getCss:  () => 'h1{color:red}',
          getComponents: () => ({ toJSON: () => [{ type: 'h1' }] }),
          getAssets: () => ({ toJSON: () => [{ type: 'image', src: '/a.png' }] }),
          destroy: () => {},
          on: (_e: string, _fn: () => void) => {},
        };
      },
    },
  };
});

function buildHost(): HTMLElement {
  const root = document.createElement('div');
  root.setAttribute('data-controller', 'grapesjs');
  root.setAttribute('data-grapesjs-config-value', '{"storageManager":{"type":"none"}}');
  root.setAttribute('data-grapesjs-format-value', 'page');
  root.setAttribute('data-grapesjs-bridge-id-value', 'grapesjs');
  const input = document.createElement('textarea');
  input.setAttribute('data-grapesjs-target', 'input');
  const mount = document.createElement('div');
  mount.setAttribute('data-grapesjs-target', 'mount');
  root.append(input, mount);
  document.body.append(root);
  return root;
}

describe('GrapesJSController', () => {
  it('mounts and serialize returns bundle shape', async () => {
    const { default: Controller } = await import('../src/controller.js');
    const root = buildHost();
    const events: string[] = [];
    ['ux:editor:pre-connect', 'ux:editor:connect'].forEach(n => root.addEventListener(n, () => events.push(n)));

    const app = Application.start();
    app.register('grapesjs', Controller as any);
    await new Promise(r => setTimeout(r, 10));

    expect(events).toEqual(['ux:editor:pre-connect', 'ux:editor:connect']);
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(root, 'grapesjs');
    const out: any = ctrl.serialize(ctrl.instance);
    expect(out.html).toBe('<h1>X</h1>');
    expect(out.css).toBe('h1{color:red}');
    expect(out.components[0].type).toBe('h1');
    expect(out.assets[0].src).toBe('/a.png');
    app.stop();
    root.remove();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/src/Bridge/GrapesJS/assets/src/controller.ts
import { AbstractPageBuilderController, type PageInstance } from '@symfony/ux-editor/format/page';

interface GrapesEditor {
  getHtml(): string;
  getCss(): string;
  getComponents(): { toJSON(): unknown[] };
  getAssets():    { toJSON(): unknown[] };
  destroy(): void;
  on(event: string, fn: () => void): void;
}

export default class GrapesJSController extends AbstractPageBuilderController {
  static values = { ...(AbstractPageBuilderController as any).values };

  async createEditor(mount: HTMLElement, config: Record<string, unknown>): Promise<PageInstance> {
    const { default: grapesjs } = await import('grapesjs');
    const editor = grapesjs.init({ container: mount, ...(config as Record<string, unknown>) }) as GrapesEditor;

    // Wire change events so syncInput fires after edits.
    ['component:add', 'component:remove', 'component:update', 'asset:add', 'asset:remove'].forEach(ev =>
      editor.on(ev, () => this.syncInput())
    );

    // Adapt GrapesJS Backbone-Collections-returning getters to plain arrays.
    const adapter: PageInstance = {
      getHtml:       () => editor.getHtml(),
      getCss:        () => editor.getCss(),
      getComponents: () => editor.getComponents().toJSON(),
      getAssets:     () => editor.getAssets().toJSON(),
      destroy:       () => editor.destroy(),
    };
    return adapter;
  }

  serialize(instance: PageInstance): object {
    return {
      html:       instance.getHtml(),
      css:        instance.getCss(),
      components: instance.getComponents(),
      assets:     instance.getAssets(),
    };
  }

  hotReloadable(): Set<string> { return new Set(); }

  applyConfig(_diff: Record<string, unknown>, _instance: PageInstance): void {
    // No GrapesJS option safely hot-reloads; non-hot diff always triggers remount via base class.
  }

  async destroyEditor(instance: PageInstance): Promise<void> {
    if (typeof instance?.destroy === 'function') {
      await instance.destroy();
    }
  }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/assets/src/controller.ts src/Editor/src/Bridge/GrapesJS/assets/test/controller.test.ts
git commit -m "feat(editor-grapesjs): add GrapesJSController"
```

---

## Task 8 — `assets/controllers.json`

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/assets/controllers.json`

- [ ] **Step 1: Write file**

```json
{
    "controllers": {
        "@symfony/ux-editor-grapesjs": {
            "grapesjs": {
                "main": "dist/controller.js",
                "fetch": "lazy",
                "enabled": true,
                "name": "symfony--ux-editor--grapesjs"
            }
        }
    },
    "entrypoints": []
}
```

- [ ] **Step 2: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('src/Editor/src/Bridge/GrapesJS/assets/controllers.json', 'utf8'))" && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/assets/controllers.json
git commit -m "feat(editor-grapesjs): register Stimulus controller"
```

---

## Task 9 — End-to-end PHPUnit integration

**Files:**
- Test: `src/Editor/src/Bridge/GrapesJS/tests/GrapesJSIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\GrapesJS\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
use Symfony\UX\Editor\Bridge\GrapesJS\GrapesJSBridge;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Form\EditorType;

final class GrapesJSIntegrationTest extends TestCase
{
    public function testRoundTripThroughEditorType(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new GrapesJSBridge()]), new PresetRegistry([])))
            ->getFormFactory();

        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config' => new GrapesJSConfig(canvasCss: 'body{margin:0}'),
        ])->getForm();

        $payload = json_encode([
            'html'       => '<h1>X</h1>',
            'css'        => 'h1{color:red}',
            'assets'     => [['type' => 'image', 'src' => '/a.png']],
            'components' => [['type' => 'h1']],
        ], \JSON_THROW_ON_ERROR);
        $form->submit(['body' => $payload]);
        self::assertTrue($form->isSynchronized());

        $data = $form->get('body')->getData();
        self::assertInstanceOf(PageContent::class, $data);
        self::assertSame('grapesjs', $data->getMetadata()['bridgeId']);
        self::assertSame('<h1>X</h1>',    $data->html);
        self::assertSame('h1{color:red}', $data->css);
        self::assertCount(1, $data->assets);
        self::assertCount(1, $data->components);

        $attr = $form->get('body')->createView()->vars['attr'];
        self::assertStringContainsString('symfony--ux-editor--grapesjs', $attr['data-controller']);
        $native = json_decode($attr['data-symfony--ux-editor--grapesjs-config-value'], true);
        self::assertSame('body{margin:0}', $native['canvas']['styles'][0]);
    }
}
```

- [ ] **Step 2: Run — expect 1 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/tests/GrapesJSIntegrationTest.php
git commit -m "test(editor-grapesjs): end-to-end through EditorType"
```

---

## Task 10 — Playwright E2E specs

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/playwright.config.ts`
- Create: `src/Editor/src/Bridge/GrapesJS/tests/e2e/{mounts,serialize-roundtrip,asset-extraction}.spec.ts`

> Depends on Plan 5 demo routes.

- [ ] **Step 1: Playwright config**

```ts
import { defineConfig } from '@playwright/test';
import base from '../../../../../playwright.config.base';

export default defineConfig({
  ...base,
  testDir: './tests/e2e',
  projects: [{ name: 'grapesjs', testMatch: '**/*.spec.ts' }],
});
```

- [ ] **Step 2: `mounts.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('GrapesJS mounts on the demo page', async ({ page }) => {
  await page.goto('/editor/page');
  const root = page.locator('[data-controller~="symfony--ux-editor--grapesjs"]');
  await expect(root).toBeVisible();
  await expect(page.locator('.gjs-editor')).toBeVisible();
});
```

- [ ] **Step 3: `serialize-roundtrip.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('drag-and-drop + submit produces PageContent bundle', async ({ page }) => {
  await page.goto('/editor/page');
  await page.evaluate(() => {
    const editor = (window as any).__demoGrapes;
    if (editor) {
      editor.setComponents('<section class="hero"><h1>Hero</h1></section>');
      editor.setStyle('h1{color:red}');
    }
  });
  await page.locator('[data-test-id="submit-button"]').click();
  const dump = await page.locator('[data-test-id="entity-dump"]').textContent();
  expect(dump).toContain('<h1>Hero</h1>');
  expect(dump).toContain('h1{color:red}');
});
```

- [ ] **Step 4: `asset-extraction.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('image asset captured in PageContent.assets', async ({ page }) => {
  await page.goto('/editor/page');
  await page.evaluate(() => {
    const editor = (window as any).__demoGrapes;
    if (editor) {
      editor.AssetManager.add([{ type: 'image', src: '/uploads/test.png' }]);
      editor.setComponents('<img src="/uploads/test.png">');
    }
  });
  await page.locator('[data-test-id="submit-button"]').click();
  const dump = await page.locator('[data-test-id="entity-dump"]').textContent();
  expect(dump).toContain('/uploads/test.png');
});
```

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/playwright.config.ts src/Editor/src/Bridge/GrapesJS/tests/e2e/
git commit -m "test(editor-grapesjs): add Playwright specs (runs after Plan 5)"
```

---

## Task 11 — TypeScript build

- [ ] **Step 1: Build**

```bash
cd src/Editor/src/Bridge/GrapesJS && npx tsc
```

Expected: zero errors; `dist/` populated.

- [ ] **Step 2: Smoke import**

```bash
node -e "import('./dist/controller.js').then(m => console.log(typeof m.default))"
```

Expected: `function`.

- [ ] **Step 3-5: N/A** (fix tsconfig on failure).

---

## Task 12 — Sphinx docs

**Files:**
- Create: `src/Editor/src/Bridge/GrapesJS/doc/index.rst`

- [ ] **Step 1: Write file**

```rst
GrapesJS Bridge
===============

`symfony/ux-editor-grapesjs`_ integrates `GrapesJS`_ with ``symfony/ux-editor``.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/ux-editor-grapesjs

Then add the upstream GrapesJS library. With AssetMapper:

.. code-block:: terminal

    $ php bin/console importmap:require grapesjs

Or with Webpack Encore:

.. code-block:: terminal

    $ npm install grapesjs

Use
---

.. code-block:: php

    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\UX\Editor\Bridge\GrapesJS\Config\GrapesJSConfig;
    use Symfony\UX\Editor\Config\CommonOptions;
    use Symfony\UX\Editor\Form\EditorType;

    final class LandingPageType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $b, array $options): void
        {
            $b->add('homepage', EditorType::class, [
                'config' => new GrapesJSConfig(
                    common: new CommonOptions(language: 'en'),
                    blocks: [
                        ['id' => 'hero', 'label' => 'Hero', 'content' => '<section><h1>Hero</h1></section>'],
                    ],
                    canvasCss: 'body{font-family:system-ui;margin:0}',
                ),
            ]);
        }
    }

Or use the shipped preset:

.. code-block:: php

    $b->add('homepage', EditorType::class, ['preset' => 'page_builder.landing']);

Rendering saved page content
----------------------------

``PageContent`` renders in a sandboxed iframe via ``ux_editor_render``:

.. code-block:: twig

    {{ ux_editor_render(page.homepage) }}

For embedding directly in your HTML (not iframed), use ``PageAssetExtractor`` shipped in
``symfony/ux-editor`` to collect asset URLs and reconcile them with your storage.

Native overrides
----------------

Any GrapesJS option not exposed by ``GrapesJSConfig`` passes through ``nativeOverrides``:

.. code-block:: php

    new GrapesJSConfig(
        nativeOverrides: ['styleManager' => ['sectors' => [...]]],
    )

.. _`symfony/ux-editor-grapesjs`: https://github.com/symfony/ux
.. _`GrapesJS`: https://grapesjs.com
```

- [ ] **Step 2: Verify non-empty**

```bash
test -s src/Editor/src/Bridge/GrapesJS/doc/index.rst && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/GrapesJS/doc/index.rst
git commit -m "docs(editor-grapesjs): add Sphinx index.rst"
```

---

## Task 13 — `splitsh.json` update

**Files:**
- Modify: `splitsh.json`

- [ ] **Step 1: Inspect existing entries**

```bash
head -40 splitsh.json
```

- [ ] **Step 2: Append entry**

```json
{
    "name": "ux-editor-grapesjs",
    "directory": "src/Editor/src/Bridge/GrapesJS",
    "target": "https://github.com/symfony/ux-editor-grapesjs.git"
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
git commit -m "chore(editor-grapesjs): split symfony/ux-editor-grapesjs to its own repo"
```

---

## Task 14 — Coverage gates

- [ ] **Step 1: PHPUnit ≥ 75% Tier 2**

```bash
cd src/Editor/src/Bridge/GrapesJS && XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text
```

- [ ] **Step 2: Vitest ≥ 75%**

```bash
npx vitest run --coverage
```

- [ ] **Step 3: If under threshold — add tests + re-run + commit per test**

- [ ] **Step 4-5: N/A**

---

## Task 15 — Audit

- [ ] **Step 1: PHP**

```bash
cd src/Editor/src/Bridge/GrapesJS && composer audit
```

- [ ] **Step 2: NPM**

```bash
npm audit --omit=dev
```

Expected (both): zero advisories.

- [ ] **Step 3-5: N/A**

---

## Task 16 — Finalize CHANGELOG + tag

**Files:**
- Modify: `src/Editor/src/Bridge/GrapesJS/CHANGELOG.md`

- [ ] **Step 1: Replace content**

```markdown
# CHANGELOG

## 0.1.0 — 2026-05-19

Initial release: GrapesJS bridge for `symfony/ux-editor`.

### Added

- `GrapesJSBridge` (extends `AbstractPageBuilderBridge`).
- `GrapesJSConfig` (extends `AbstractPageConfig`) with typed fields: `components`, `blocks`, `storageManager`, `deviceManager`, `canvasCss`.
- `GrapesJSTransformer` (extends `AbstractPageTransformer`).
- `PageBuilderLandingPreset` (`name: page_builder.landing`) with hero/section/text/image blocks + responsive devices.
- Stimulus controller `symfony--ux-editor--grapesjs` with dynamic import of `grapesjs`, change-event wiring (component/asset add/remove/update), Collection-to-array adapter for `getComponents`/`getAssets`.

### Requires

- `symfony/ux-editor` ^0.1.
- `grapesjs` ^0.21 (peer dep).
```

- [ ] **Step 2: Commit CHANGELOG**

```bash
git add src/Editor/src/Bridge/GrapesJS/CHANGELOG.md
git commit -m "docs(editor-grapesjs): finalize CHANGELOG 0.1.0"
```

- [ ] **Step 3: Verify clean tree**

```bash
git status
```

Expected: clean.

- [ ] **Step 4: Annotated tag (local only)**

```bash
git tag -a editor-grapesjs/v0.1.0 -m "symfony/ux-editor-grapesjs 0.1.0"
```

- [ ] **Step 5: Verify**

```bash
git tag -l 'editor-grapesjs/v0.1.0'
```

Expected: `editor-grapesjs/v0.1.0`.

---

## Final checkpoint

- [ ] **Run all tests**

```bash
cd src/Editor/src/Bridge/GrapesJS && vendor/bin/phpunit && npx vitest run
```

Expected: both green.

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

## Self-review for Plan 4

**1. Spec coverage:**
- §3 Tier 2 GrapesJS bridge structure → Tasks 1, 4.
- §4.3 `PageContent` transform + metadata → Tasks 3, 9.
- §5 Typed config + nativeOverrides → Task 2.
- §5.5 Service-registered preset → Task 5.
- §6 npm sub-package + peerDep + dynamic import + both asset stacks → Tasks 1, 7, 8.
- §8 Hot-reload: explicitly empty for GrapesJS → Task 7.
- §10 Phasing: GrapesJS is the third v1.0 bridge — Tasks 1, 16.
- §12 Testing → Tasks 7, 9, 10, 14.

**2. Placeholder scan:**
- Task 7 empty `hotReloadable()`: design intent, not a TODO.
- Task 7 Backbone-Collections adapter: intentional integration, not handwave.
- Task 10 deferral to Plan 5: explicit dependency.
- Task 11, 13, 14 conditional steps: bounded.
- No `TBD`/`FIXME`/handwave.

**3. Type consistency:**
- `GrapesJSBridge` extends `AbstractPageBuilderBridge` (Plan 1 Task 52).
- `GrapesJSConfig` extends `AbstractPageConfig` (Plan 1 Task 48); uses `CommonOptions` (Plan 1 Task 8).
- `GrapesJSTransformer` extends `AbstractPageTransformer` (Plan 1 Task 51).
- Service tags match Plan 1 core.
- Stimulus id matches `AbstractBridge::getControllerName()` derivation.
- JS controller imports from `@symfony/ux-editor/format/page`.
- `PageInstance` shape (`getHtml/getCss/getComponents/getAssets/destroy?`) matches Plan 1 Task 63.
- `PageContent` round-trip (`html/css/assets/components`) consistent with Plan 1 Task 6.

No inline fixes needed.

---

## Execution handoff

Plan 4 saved to `docs/superpowers/plans/2026-05-19-ux-editor-bridge-grapesjs.md`.

Execution options:

1. **Subagent-Driven (recommended)** — `superpowers:subagent-driven-development`.
2. **Inline** — `superpowers:executing-plans` with checkpoints after Tasks 4, 6, 9, 16.

Plan 5 follows: `ux.symfony.com` demo + LiveComponent showcase + makes Playwright specs from Plans 2/3/4 runnable.

Which approach for Plan 4?
