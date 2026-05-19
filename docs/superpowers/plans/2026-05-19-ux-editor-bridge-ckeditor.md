# UX Editor — CKEditor Bridge Implementation Plan (Plan 3 / 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `symfony/ux-editor-ckeditor` — the second Tier 2 bridge. Provides `CKEditorBridge`, `CKEditorConfig` (typed fields for extra/remove plugins, heading, image, link, licenseKey), CKEditor Stimulus controller with hot `readOnly`, and two service-registered presets.

**Architecture:** Tier 2 bridge depending on `symfony/ux-editor` (Tier 0 + Tier 1 from Plan 1). Inherits `AbstractWysiwygBridge` / `AbstractWysiwygConfig` / `AbstractWysiwygTransformer` / `AbstractWysiwygController`. Adds CKEditor-specific config, composer + npm sub-package, peerDep on `@ckeditor/ckeditor5-build-classic`.

**Tech Stack:** Same as Plan 1/2. New peerDep: `@ckeditor/ckeditor5-build-classic ^41`.

**Depends on:** Plan 1. Independent of Plan 2 (parallelizable).

**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](../specs/2026-05-19-ux-editor-design.md)
**Roadmap:** [`ROADMAP-UX-EDITOR.md`](../../../ROADMAP-UX-EDITOR.md) — CKEditor is the **second** v1.0 bridge (largest WYSIWYG; stresses license-key path + capability flags).

---

## File structure

```
src/Editor/src/Bridge/CKEditor/
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
│   ├── CKEditorBridge.php
│   ├── Config/CKEditorConfig.php
│   ├── Transformer/CKEditorTransformer.php
│   └── Preset/{WysiwygFullPreset,WysiwygMinimalPreset}.php
└── tests/                                 # mirror of src/
```

Order: scaffold → config → bridge → transformer → presets → service wiring → JS controller → integration → coverage → audit → docs → tag.

---

## Task 1 — Scaffold sub-package

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/{composer.json, package.json, tsconfig.json, vitest.config.mjs, phpunit.dist.xml, CHANGELOG.md, LICENSE, README.md, tests/.gitkeep}`

- [ ] **Step 1: Copy LICENSE**

```bash
cp src/Editor/LICENSE src/Editor/src/Bridge/CKEditor/LICENSE
```

- [ ] **Step 2: Write composer.json**

```json
{
    "name": "symfony/ux-editor-ckeditor",
    "type": "symfony-bundle",
    "description": "CKEditor 5 bridge for symfony/ux-editor",
    "keywords": ["symfony-ux", "editor", "ckeditor", "wysiwyg"],
    "homepage": "https://symfony.com",
    "license": "MIT",
    "authors": [{"name": "Symfony Community", "homepage": "https://symfony.com/contributors"}],
    "autoload": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\CKEditor\\": "src/"}},
    "autoload-dev": {"psr-4": {"Symfony\\UX\\Editor\\Bridge\\CKEditor\\Tests\\": "tests/"}},
    "require": {"php": ">=8.4", "symfony/ux-editor": "^0.1|^1.0"},
    "require-dev": {"phpunit/phpunit": "^11.1|^12.0", "symfony/framework-bundle": "^7.4|^8.0"},
    "extra": {"thanks": {"name": "symfony/ux", "url": "https://github.com/symfony/ux"}},
    "minimum-stability": "dev"
}
```

- [ ] **Step 3: Write package.json**

```json
{
    "name": "@symfony/ux-editor-ckeditor",
    "description": "CKEditor 5 bridge controller for symfony/ux-editor",
    "license": "MIT",
    "version": "0.1.0",
    "symfony": {
        "controllers": {"ckeditor": {"main": "dist/controller.js", "fetch": "lazy", "enabled": true}},
        "importmap": {"@symfony/ux-editor": "^0.1.0", "@ckeditor/ckeditor5-build-classic": "^41.0.0"}
    },
    "type": "module",
    "main": "dist/controller.js",
    "types": "dist/controller.d.ts",
    "files": ["dist/"],
    "peerDependencies": {
        "@hotwired/stimulus": "^3.0.0",
        "@symfony/ux-editor": "^0.1.0",
        "@ckeditor/ckeditor5-build-classic": "^41.0.0"
    },
    "devDependencies": {
        "@hotwired/stimulus": "^3.2.2",
        "@ckeditor/ckeditor5-build-classic": "^41.0.0",
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
        <testsuite name="symfony/ux-editor-ckeditor"><directory>./tests/</directory></testsuite>
    </testsuites>
    <source><include><directory>./src</directory></include></source>
</phpunit>
```

- [ ] **Step 7: Write README + CHANGELOG stubs**

`README.md`:
```markdown
# symfony/ux-editor-ckeditor

CKEditor 5 bridge for [symfony/ux-editor](https://github.com/symfony/ux-editor).

## Install

    composer require symfony/ux-editor-ckeditor
    npm install @ckeditor/ckeditor5-build-classic

## Use

    $builder->add('body', EditorType::class, ['bridge' => 'ckeditor']);

See doc/index.rst.
```

`CHANGELOG.md`:
```markdown
# CHANGELOG

## 0.1.0

- Initial release: CKEditor 5 bridge for symfony/ux-editor.
```

- [ ] **Step 8: Smoke-run**

```bash
cd src/Editor/src/Bridge/CKEditor && composer install
vendor/bin/phpunit
```

Expected: 0 tests, exit 0.

- [ ] **Step 9: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/
git commit -m "feat(editor-ckeditor): scaffold symfony/ux-editor-ckeditor package"
```

---

## Task 2 — `CKEditorConfig`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/src/Config/CKEditorConfig.php`
- Test: `src/Editor/src/Bridge/CKEditor/tests/Config/CKEditorConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class CKEditorConfigTest extends TestCase
{
    public function testBridgeIdAndCapabilities(): void
    {
        $cfg = new CKEditorConfig();
        self::assertSame('ckeditor', $cfg->getBridgeId());
        self::assertEquals(WysiwygCapabilities::default(), $cfg->getCapabilities());
    }

    public function testTranslateCommonAndOwn(): void
    {
        $cfg = new CKEditorConfig(
            common: new CommonOptions(
                toolbar: ['bold', 'italic', 'link'],
                placeholder: 'Write…',
                language: 'fr',
            ),
            extraPlugins:  ['SourceEditing'],
            removePlugins: ['Markdown'],
            heading: ['options' => [
                ['model' => 'paragraph', 'title' => 'P'],
                ['model' => 'heading2',  'view' => 'h2', 'title' => 'H2'],
            ]],
            image: ['toolbar' => ['imageTextAlternative']],
            link:  ['decorators' => ['openInNewTab' => ['mode' => 'manual']]],
            licenseKey: 'GPL',
        );
        $n = $cfg->toNative();
        self::assertSame(['items' => ['bold', 'italic', 'link']], $n['toolbar']);
        self::assertSame('Write…', $n['placeholder']);
        self::assertSame('fr', $n['language']);
        self::assertSame(['SourceEditing'], $n['extraPlugins']);
        self::assertSame(['Markdown'], $n['removePlugins']);
        self::assertArrayHasKey('heading', $n);
        self::assertArrayHasKey('image', $n);
        self::assertArrayHasKey('link', $n);
        self::assertSame('GPL', $n['licenseKey']);
    }

    public function testToolbarOmittedWhenNotSet(): void
    {
        self::assertArrayNotHasKey('toolbar', (new CKEditorConfig())->toNative());
    }

    public function testNativeOverridesWinLast(): void
    {
        $cfg = new CKEditorConfig(
            common: new CommonOptions(language: 'fr'),
            nativeOverrides: ['language' => 'en', 'ui' => ['poweredBy' => ['forceVisible' => false]]],
        );
        $n = $cfg->toNative();
        self::assertSame('en', $n['language']);
        self::assertFalse($n['ui']['poweredBy']['forceVisible']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Config;

use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygConfig;
use Symfony\UX\Editor\Config\CommonOptions;

final class CKEditorConfig extends AbstractWysiwygConfig
{
    public function __construct(
        CommonOptions $common = new CommonOptions(),
        public readonly array   $extraPlugins  = [],
        public readonly array   $removePlugins = [],
        public readonly ?array  $heading       = null,
        public readonly ?array  $image         = null,
        public readonly ?array  $link          = null,
        public readonly ?string $licenseKey    = null,
        array $nativeOverrides = [],
    ) {
        parent::__construct($common, $nativeOverrides);
    }

    public function getBridgeId(): string { return 'ckeditor'; }

    protected function translateCommon(CommonOptions $c): array
    {
        $out = [];
        if ($c->toolbar !== null)     { $out['toolbar']      = ['items' => $c->toolbar]; }
        if ($c->placeholder !== null) { $out['placeholder']  = $c->placeholder; }
        if ($c->readOnly)             { $out['readOnly']     = true; }
        if ($c->language !== null)    { $out['language']     = $c->language; }
        if ($c->plugins !== [])       { $out['extraPlugins'] = $c->plugins; }
        return $out;
    }

    protected function translateOwn(): array
    {
        $out = [];
        if ($this->extraPlugins !== [])  { $out['extraPlugins']  = $this->extraPlugins; }
        if ($this->removePlugins !== []) { $out['removePlugins'] = $this->removePlugins; }
        if ($this->heading !== null)     { $out['heading']       = $this->heading; }
        if ($this->image !== null)       { $out['image']         = $this->image; }
        if ($this->link !== null)        { $out['link']          = $this->link; }
        if ($this->licenseKey !== null)  { $out['licenseKey']    = $this->licenseKey; }
        return $out;
    }
}
```

> Note: when both `CommonOptions::plugins` and `extraPlugins` are set, `translateOwn` overrides `translateCommon` per merge order. Apps wanting combined lists should pre-merge before constructing.

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/src/Config/CKEditorConfig.php src/Editor/src/Bridge/CKEditor/tests/Config/CKEditorConfigTest.php
git commit -m "feat(editor-ckeditor): add CKEditorConfig"
```

---

## Task 3 — `CKEditorTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/src/Transformer/CKEditorTransformer.php`
- Test: `src/Editor/src/Bridge/CKEditor/tests/Transformer/CKEditorTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests\Transformer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\CKEditor\Transformer\CKEditorTransformer;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class CKEditorTransformerTest extends TestCase
{
    public function testMetadata(): void
    {
        $t = new CKEditorTransformer();
        self::assertSame('ckeditor', $t->getBridgeId());
        self::assertSame(HtmlContent::class, $t->getContentClass());
        self::assertSame(StorageShape::Scalar, $t->getStorageShape());
    }

    public function testReverseStampsBridgeId(): void
    {
        $hc = (new CKEditorTransformer())->reverseTransform('<p>hi</p>');
        self::assertInstanceOf(HtmlContent::class, $hc);
        self::assertSame('<p>hi</p>', $hc->html);
        self::assertSame('ckeditor', $hc->getMetadata()['bridgeId']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Transformer;

use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygTransformer;

final class CKEditorTransformer extends AbstractWysiwygTransformer
{
    public function getBridgeId(): string { return 'ckeditor'; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/src/Transformer/CKEditorTransformer.php src/Editor/src/Bridge/CKEditor/tests/Transformer/CKEditorTransformerTest.php
git commit -m "feat(editor-ckeditor): add CKEditorTransformer"
```

---

## Task 4 — `CKEditorBridge`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/src/CKEditorBridge.php`
- Test: `src/Editor/src/Bridge/CKEditor/tests/CKEditorBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\CKEditor\CKEditorBridge;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Bridge\CKEditor\Transformer\CKEditorTransformer;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;

final class CKEditorBridgeTest extends TestCase
{
    public function testMetadata(): void
    {
        $b = new CKEditorBridge();
        self::assertSame('ckeditor', $b->getId());
        self::assertSame('symfony--ux-editor--ckeditor', $b->getControllerName());
        self::assertEquals(WysiwygCapabilities::default(), $b->getCapabilities());
        self::assertInstanceOf(CKEditorConfig::class, $b->getDefaultConfig());
        self::assertInstanceOf(CKEditorTransformer::class, $b->createTransformer());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor;

use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Bridge\CKEditor\Transformer\CKEditorTransformer;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygBridge;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class CKEditorBridge extends AbstractWysiwygBridge
{
    public function getId(): string { return 'ckeditor'; }
    public function getDefaultConfig(): EditorConfigInterface { return new CKEditorConfig(); }
    public function createTransformer(): EditorContentTransformerInterface { return new CKEditorTransformer(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/src/CKEditorBridge.php src/Editor/src/Bridge/CKEditor/tests/CKEditorBridgeTest.php
git commit -m "feat(editor-ckeditor): add CKEditorBridge"
```

---

## Task 5 — `WysiwygMinimalPreset`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/src/Preset/WysiwygMinimalPreset.php`
- Test: `src/Editor/src/Bridge/CKEditor/tests/Preset/WysiwygMinimalPresetTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests\Preset;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Bridge\CKEditor\Preset\WysiwygMinimalPreset;

final class WysiwygMinimalPresetTest extends TestCase
{
    public function testBuilds(): void
    {
        $cfg = (new WysiwygMinimalPreset())->build();
        self::assertInstanceOf(CKEditorConfig::class, $cfg);
        self::assertSame('ckeditor', $cfg->getBridgeId());
        self::assertSame(['bold', 'italic', 'link'], $cfg->getCommon()->toolbar);
        self::assertSame('GPL', $cfg->licenseKey);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Preset;

use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;

final class WysiwygMinimalPreset implements EditorPresetInterface
{
    public function build(): EditorConfigInterface
    {
        return new CKEditorConfig(
            common: new CommonOptions(
                toolbar: ['bold', 'italic', 'link'],
                placeholder: 'Write…',
            ),
            licenseKey: 'GPL',
        );
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/src/Preset/WysiwygMinimalPreset.php src/Editor/src/Bridge/CKEditor/tests/Preset/WysiwygMinimalPresetTest.php
git commit -m "feat(editor-ckeditor): add wysiwyg.minimal preset"
```

---

## Task 6 — `WysiwygFullPreset`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/src/Preset/WysiwygFullPreset.php`
- Test: `src/Editor/src/Bridge/CKEditor/tests/Preset/WysiwygFullPresetTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests\Preset;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Bridge\CKEditor\Preset\WysiwygFullPreset;

final class WysiwygFullPresetTest extends TestCase
{
    public function testBuildsRichConfig(): void
    {
        $cfg = (new WysiwygFullPreset())->build();
        self::assertInstanceOf(CKEditorConfig::class, $cfg);
        $toolbar = $cfg->getCommon()->toolbar;
        foreach (['heading', 'bold', 'italic', 'link', 'bulletedList', 'numberedList', 'blockQuote', 'undo', 'redo'] as $tool) {
            self::assertContains($tool, $toolbar);
        }
        self::assertNotNull($cfg->heading);
        self::assertSame('GPL', $cfg->licenseKey);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Preset;

use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;

final class WysiwygFullPreset implements EditorPresetInterface
{
    public function build(): EditorConfigInterface
    {
        return new CKEditorConfig(
            common: new CommonOptions(
                toolbar: [
                    'heading', '|',
                    'bold', 'italic', 'link', '|',
                    'bulletedList', 'numberedList', 'blockQuote', '|',
                    'undo', 'redo',
                ],
                placeholder: 'Write your content…',
            ),
            heading: ['options' => [
                ['model' => 'paragraph', 'title' => 'Paragraph', 'class' => 'ck-heading_paragraph'],
                ['model' => 'heading2',  'view'  => 'h2', 'title' => 'Heading 2', 'class' => 'ck-heading_heading2'],
                ['model' => 'heading3',  'view'  => 'h3', 'title' => 'Heading 3', 'class' => 'ck-heading_heading3'],
            ]],
            licenseKey: 'GPL',
        );
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/src/Preset/WysiwygFullPreset.php src/Editor/src/Bridge/CKEditor/tests/Preset/WysiwygFullPresetTest.php
git commit -m "feat(editor-ckeditor): add wysiwyg.full preset"
```

---

## Task 7 — Service wiring

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/config/services.php`

- [ ] **Step 1: Write file**

```php
<?php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\UX\Editor\Bridge\CKEditor\CKEditorBridge;
use Symfony\UX\Editor\Bridge\CKEditor\Preset\WysiwygFullPreset;
use Symfony\UX\Editor\Bridge\CKEditor\Preset\WysiwygMinimalPreset;

return static function (ContainerConfigurator $c): void {
    $s = $c->services()->defaults()->autowire()->autoconfigure();

    $s->set(CKEditorBridge::class)->tag('ux.editor.bridge');
    $s->set(WysiwygMinimalPreset::class)->tag('ux.editor.preset', ['name' => 'wysiwyg.minimal']);
    $s->set(WysiwygFullPreset::class)->tag('ux.editor.preset', ['name' => 'wysiwyg.full']);
};
```

- [ ] **Step 2-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/config/services.php
git commit -m "feat(editor-ckeditor): wire bridge + presets"
```

---

## Task 8 — CKEditor Stimulus controller

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/assets/src/controller.ts`
- Create: `src/Editor/src/Bridge/CKEditor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (programmatic DOM)**

```ts
import { describe, it, expect, vi } from 'vitest';
import { Application } from '@hotwired/stimulus';

vi.mock('@ckeditor/ckeditor5-build-classic', () => {
  return {
    default: class FakeClassicEditor {
      static async create(_element: HTMLElement, opts: any): Promise<any> {
        return new FakeClassicEditor(opts);
      }
      constructor(public opts: any) { this.isReadOnly = false; }
      isReadOnly: boolean;
      getData(): string { return '<p>x</p>'; }
      setData(_data: string): void {}
      enableReadOnlyMode(_id: string): void { this.isReadOnly = true; }
      disableReadOnlyMode(_id: string): void { this.isReadOnly = false; }
      destroy(): Promise<void> { return Promise.resolve(); }
    },
  };
});

function buildHost(): HTMLElement {
  const root = document.createElement('div');
  root.setAttribute('data-controller', 'ckeditor');
  root.setAttribute('data-ckeditor-config-value', '{"placeholder":"Write…"}');
  root.setAttribute('data-ckeditor-format-value', 'html');
  root.setAttribute('data-ckeditor-bridge-id-value', 'ckeditor');
  const input = document.createElement('textarea');
  input.setAttribute('data-ckeditor-target', 'input');
  const mount = document.createElement('div');
  mount.setAttribute('data-ckeditor-target', 'mount');
  root.append(input, mount);
  document.body.append(root);
  return root;
}

describe('CKEditorController', () => {
  it('mounts and serialize returns getData()', async () => {
    const { default: Controller } = await import('../src/controller.js');
    const root = buildHost();
    const events: string[] = [];
    ['ux:editor:pre-connect', 'ux:editor:connect'].forEach(n => root.addEventListener(n, () => events.push(n)));

    const app = Application.start();
    app.register('ckeditor', Controller as any);
    await new Promise(r => setTimeout(r, 10));

    expect(events).toEqual(['ux:editor:pre-connect', 'ux:editor:connect']);
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(root, 'ckeditor');
    expect(ctrl.serialize(ctrl.instance)).toBe('<p>x</p>');
    app.stop();
    root.remove();
  });

  it('hot-applies readOnly', async () => {
    const { default: Controller } = await import('../src/controller.js');
    const root = buildHost();
    const app = Application.start();
    app.register('ckeditor', Controller as any);
    await new Promise(r => setTimeout(r, 10));
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(root, 'ckeditor');
    expect(ctrl.hotReloadable().has('readOnly')).toBe(true);
    expect(ctrl.instance.isReadOnly).toBe(false);
    await ctrl.applyConfig({ readOnly: true }, ctrl.instance);
    expect(ctrl.instance.isReadOnly).toBe(true);
    await ctrl.applyConfig({ readOnly: false }, ctrl.instance);
    expect(ctrl.instance.isReadOnly).toBe(false);
    app.stop();
    root.remove();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/src/Bridge/CKEditor/assets/src/controller.ts
import { AbstractWysiwygController, type WysiwygInstance } from '@symfony/ux-editor/format/wysiwyg';

interface CKEditorInstance extends WysiwygInstance {
  getData(): string;
  setData(data: string): void;
  enableReadOnlyMode(lockId: string): void;
  disableReadOnlyMode(lockId: string): void;
  isReadOnly: boolean;
}

const READ_ONLY_LOCK_ID = 'symfony--ux-editor--ckeditor';

export default class CKEditorController extends AbstractWysiwygController {
  static values = { ...(AbstractWysiwygController as any).values };

  async createEditor(mount: HTMLElement, config: Record<string, unknown>): Promise<WysiwygInstance> {
    const { default: ClassicEditor } = await import('@ckeditor/ckeditor5-build-classic');
    const editor = await ClassicEditor.create(mount, config);
    // Wire change event so syncInput fires on user input.
    if (editor.model && typeof editor.model.document?.on === 'function') {
      editor.model.document.on('change:data', () => this.syncInput());
    }
    return editor as unknown as WysiwygInstance;
  }

  serialize(instance: WysiwygInstance): string {
    return (instance as CKEditorInstance).getData();
  }

  hotReloadable(): Set<string> { return new Set(['readOnly', 'placeholder']); }

  async applyConfig(diff: Record<string, unknown>, instance: any): Promise<void> {
    const ck = instance as CKEditorInstance;
    if ('readOnly' in diff) {
      if (diff.readOnly) {
        ck.enableReadOnlyMode(READ_ONLY_LOCK_ID);
      } else {
        ck.disableReadOnlyMode(READ_ONLY_LOCK_ID);
      }
    }
    // placeholder: CKEditor reads it at mount; cannot be applied without remount → falls through to remount path.
  }

  async destroyEditor(instance: any): Promise<void> {
    if (typeof instance?.destroy === 'function') {
      await instance.destroy();
    }
  }
}
```

> Compatibility: the test mock uses CKEditor 5 v41+ `enableReadOnlyMode`/`disableReadOnlyMode` API. If the app pins an older version (v40 or earlier where `editor.isReadOnly = true` was set directly), apps subclass and override `applyConfig`.

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/assets/src/controller.ts src/Editor/src/Bridge/CKEditor/assets/test/controller.test.ts
git commit -m "feat(editor-ckeditor): add CKEditorController with hot readOnly"
```

---

## Task 9 — `assets/controllers.json`

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/assets/controllers.json`

- [ ] **Step 1: Write file**

```json
{
    "controllers": {
        "@symfony/ux-editor-ckeditor": {
            "ckeditor": {
                "main": "dist/controller.js",
                "fetch": "lazy",
                "enabled": true,
                "name": "symfony--ux-editor--ckeditor"
            }
        }
    },
    "entrypoints": []
}
```

- [ ] **Step 2: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('src/Editor/src/Bridge/CKEditor/assets/controllers.json', 'utf8'))" && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/assets/controllers.json
git commit -m "feat(editor-ckeditor): register Stimulus controller"
```

---

## Task 10 — End-to-end PHPUnit integration + sanitize-on-submit

**Files:**
- Test: `src/Editor/src/Bridge/CKEditor/tests/CKEditorIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Bridge\CKEditor\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Bridge\CKEditor\CKEditorBridge;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\EditorType;

final class CKEditorIntegrationTest extends TestCase
{
    public function testRoundTripThroughEditorType(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new CKEditorBridge()]), new PresetRegistry([])))
            ->getFormFactory();

        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config' => new CKEditorConfig(common: new CommonOptions(toolbar: ['bold', 'italic'])),
        ])->getForm();

        $form->submit(['body' => '<p>hello</p>']);
        self::assertTrue($form->isSynchronized());

        $data = $form->get('body')->getData();
        self::assertInstanceOf(HtmlContent::class, $data);
        self::assertSame('<p>hello</p>', $data->html);
        self::assertSame('ckeditor', $data->getMetadata()['bridgeId']);

        $attr = $form->get('body')->createView()->vars['attr'];
        self::assertStringContainsString('symfony--ux-editor--ckeditor', $attr['data-controller']);
        $native = json_decode($attr['data-symfony--ux-editor--ckeditor-config-value'], true);
        self::assertSame(['items' => ['bold', 'italic']], $native['toolbar']);
    }

    public function testSanitizeStripsScriptOnSubmit(): void
    {
        $sanitizer = new HtmlSanitizer((new HtmlSanitizerConfig())->allowSafeElements());
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new CKEditorBridge()]), new PresetRegistry([]), $sanitizer))
            ->getFormFactory();

        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config'  => new CKEditorConfig(),
            'sanitize'=> true,
        ])->getForm();
        $form->submit(['body' => '<script>alert(1)</script><p>ok</p>']);
        self::assertStringNotContainsString('<script>', $form->get('body')->getData()->html);
        self::assertStringContainsString('<p>ok</p>', $form->get('body')->getData()->html);
    }
}
```

- [ ] **Step 2: Run — expect 2 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/tests/CKEditorIntegrationTest.php
git commit -m "test(editor-ckeditor): end-to-end + sanitize on submit"
```

---

## Task 11 — Playwright E2E specs

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/playwright.config.ts`
- Create: `src/Editor/src/Bridge/CKEditor/tests/e2e/{mounts,serialize-roundtrip,sanitize-on-submit}.spec.ts`

> Depends on Plan 5 demo routes.

- [ ] **Step 1: Playwright config**

```ts
// src/Editor/src/Bridge/CKEditor/playwright.config.ts
import { defineConfig } from '@playwright/test';
import base from '../../../../../playwright.config.base';

export default defineConfig({
  ...base,
  testDir: './tests/e2e',
  projects: [{ name: 'ckeditor', testMatch: '**/*.spec.ts' }],
});
```

- [ ] **Step 2: `mounts.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('CKEditor mounts on the demo page', async ({ page }) => {
  await page.goto('/editor/wysiwyg');
  const root = page.locator('[data-controller~="symfony--ux-editor--ckeditor"]');
  await expect(root).toBeVisible();
  await expect(page.locator('.ck-editor__editable')).toBeVisible();
});
```

- [ ] **Step 3: `serialize-roundtrip.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('typing + submit round-trips through PHP form to HtmlContent', async ({ page }) => {
  await page.goto('/editor/wysiwyg');
  await page.locator('.ck-editor__editable').click();
  await page.keyboard.type('hello world');
  await page.locator('[data-test-id="submit-button"]').click();
  const dump = await page.locator('[data-test-id="entity-dump"]').textContent();
  expect(dump).toContain('hello world');
  expect(dump).toContain('<p>');
});
```

- [ ] **Step 4: `sanitize-on-submit.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('script tags stripped after submit', async ({ page }) => {
  await page.goto('/editor/wysiwyg');
  await page.evaluate(() => {
    const editor = (window as any).__demoCkEditor;
    if (editor) editor.setData('<p>ok</p><script>alert(1)</script>');
  });
  await page.locator('[data-test-id="submit-button"]').click();
  const dump = await page.locator('[data-test-id="entity-dump"]').textContent();
  expect(dump).not.toContain('<script>');
  expect(dump).toContain('<p>ok</p>');
});
```

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/playwright.config.ts src/Editor/src/Bridge/CKEditor/tests/e2e/
git commit -m "test(editor-ckeditor): add Playwright specs (runs after Plan 5)"
```

---

## Task 12 — TypeScript build

- [ ] **Step 1: Build**

```bash
cd src/Editor/src/Bridge/CKEditor && npx tsc
```

Expected: zero errors; `dist/` populated.

- [ ] **Step 2: Smoke import**

```bash
node -e "import('./dist/controller.js').then(m => console.log(typeof m.default))"
```

Expected: `function`.

- [ ] **Step 3-5: N/A** (fix tsconfig in single commit on `tsc` failure).

---

## Task 13 — Sphinx docs

**Files:**
- Create: `src/Editor/src/Bridge/CKEditor/doc/index.rst`

- [ ] **Step 1: Write file**

```rst
CKEditor 5 Bridge
=================

`symfony/ux-editor-ckeditor`_ integrates `CKEditor 5`_ with ``symfony/ux-editor``.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/ux-editor-ckeditor

Then add the upstream CKEditor build. With AssetMapper:

.. code-block:: terminal

    $ php bin/console importmap:require @ckeditor/ckeditor5-build-classic

Or with Webpack Encore:

.. code-block:: terminal

    $ npm install @ckeditor/ckeditor5-build-classic

CKEditor 5 v44+ requires a license key. Use ``'GPL'`` for the open-source license:

.. code-block:: php

    new CKEditorConfig(licenseKey: 'GPL')

Use
---

.. code-block:: php

    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
    use Symfony\UX\Editor\Config\CommonOptions;
    use Symfony\UX\Editor\Form\EditorType;

    final class ArticleType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $b, array $options): void
        {
            $b->add('body', EditorType::class, [
                'config' => new CKEditorConfig(
                    common: new CommonOptions(
                        toolbar: ['heading', 'bold', 'italic', 'link', 'bulletedList'],
                        placeholder: 'Write…',
                        language: 'fr',
                    ),
                    extraPlugins: ['SourceEditing'],
                    licenseKey: 'GPL',
                ),
            ]);
        }
    }

Or use a shipped preset:

.. code-block:: php

    $b->add('body', EditorType::class, ['preset' => 'wysiwyg.full']);

Two presets ship with the bridge: ``wysiwyg.minimal`` (bold/italic/link) and ``wysiwyg.full``
(heading, formatting, lists, blockquote, undo/redo, heading config).

Sanitization
------------

By default ``EditorType`` runs the configured ``HtmlSanitizer`` on submit. Disable per field
if your content must round-trip raw HTML:

.. code-block:: php

    $b->add('body', EditorType::class, ['config' => new CKEditorConfig(), 'sanitize' => false]);

Native overrides
----------------

For any CKEditor option not exposed by ``CKEditorConfig``, pass through via ``nativeOverrides``:

.. code-block:: php

    new CKEditorConfig(
        nativeOverrides: ['ui' => ['poweredBy' => ['forceVisible' => false]]],
    )

Native overrides are merged **last** — they always win.

.. _`symfony/ux-editor-ckeditor`: https://github.com/symfony/ux
.. _`CKEditor 5`: https://ckeditor.com/ckeditor-5/
```

- [ ] **Step 2: Verify non-empty**

```bash
test -s src/Editor/src/Bridge/CKEditor/doc/index.rst && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/CKEditor/doc/index.rst
git commit -m "docs(editor-ckeditor): add Sphinx index.rst"
```

---

## Task 14 — `splitsh.json` update

**Files:**
- Modify: `splitsh.json`

- [ ] **Step 1: Inspect existing entries** to match exact shape:

```bash
head -40 splitsh.json
```

- [ ] **Step 2: Append entry**

```json
{
    "name": "ux-editor-ckeditor",
    "directory": "src/Editor/src/Bridge/CKEditor",
    "target": "https://github.com/symfony/ux-editor-ckeditor.git"
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
git commit -m "chore(editor-ckeditor): split symfony/ux-editor-ckeditor to its own repo"
```

---

## Task 15 — Coverage gates

- [ ] **Step 1: PHPUnit ≥ 75% Tier 2**

```bash
cd src/Editor/src/Bridge/CKEditor && XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text
```

- [ ] **Step 2: Vitest ≥ 75%**

```bash
npx vitest run --coverage
```

- [ ] **Step 3: If under threshold — add tests + re-run + commit per test**

- [ ] **Step 4-5: N/A**

---

## Task 16 — Audit

- [ ] **Step 1: PHP**

```bash
cd src/Editor/src/Bridge/CKEditor && composer audit
```

- [ ] **Step 2: NPM**

```bash
npm audit --omit=dev
```

Expected (both): zero advisories. Bump on advisory as Plan 1.

- [ ] **Step 3-5: N/A**

---

## Task 17 — Finalize CHANGELOG + tag

**Files:**
- Modify: `src/Editor/src/Bridge/CKEditor/CHANGELOG.md`

- [ ] **Step 1: Replace content**

```markdown
# CHANGELOG

## 0.1.0 — 2026-05-19

Initial release: CKEditor 5 bridge for `symfony/ux-editor`.

### Added

- `CKEditorBridge` (extends `AbstractWysiwygBridge`).
- `CKEditorConfig` (extends `AbstractWysiwygConfig`) with typed fields: `extraPlugins`, `removePlugins`, `heading`, `image`, `link`, `licenseKey`.
- `CKEditorTransformer` (extends `AbstractWysiwygTransformer`).
- Two presets: `wysiwyg.minimal` (bold/italic/link) and `wysiwyg.full` (full toolbar + heading config).
- Stimulus controller `symfony--ux-editor--ckeditor` with hot `readOnly` reload, dynamic import of `@ckeditor/ckeditor5-build-classic`, change-event wiring to `syncInput`.

### Requires

- `symfony/ux-editor` ^0.1.
- `@ckeditor/ckeditor5-build-classic` ^41 (peer dep). CKEditor 5 v44+ requires `licenseKey` — use `'GPL'` for OSS license.
```

- [ ] **Step 2: Commit CHANGELOG**

```bash
git add src/Editor/src/Bridge/CKEditor/CHANGELOG.md
git commit -m "docs(editor-ckeditor): finalize CHANGELOG 0.1.0"
```

- [ ] **Step 3: Verify clean tree**

```bash
git status
```

Expected: clean.

- [ ] **Step 4: Annotated tag (local only)**

```bash
git tag -a editor-ckeditor/v0.1.0 -m "symfony/ux-editor-ckeditor 0.1.0"
```

- [ ] **Step 5: Verify**

```bash
git tag -l 'editor-ckeditor/v0.1.0'
```

Expected: `editor-ckeditor/v0.1.0`.

---

## Final checkpoint

- [ ] **Run all tests**

```bash
cd src/Editor/src/Bridge/CKEditor && vendor/bin/phpunit && npx vitest run
```

Expected: both green. Playwright deferred to Plan 5.

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

## Self-review for Plan 3

**1. Spec coverage:**
- §3 Tier 2 CKEditor bridge structure → Tasks 1, 4.
- §4.3 `HtmlContent` transform + metadata → Tasks 3, 10.
- §5 Typed config + native overrides → Task 2.
- §5.5 Two service-registered presets → Tasks 5, 6.
- §7.1 Sanitize-on-submit integration → Task 10.
- §6 npm sub-package + peerDep + dynamic import → Tasks 1, 8, 9.
- §8 Hot-reload of `readOnly` → Task 8.
- §10 Phasing → Tasks 1, 17.
- §12 Testing → Tasks 8, 10, 11, 15.

**2. Placeholder scan:**
- Task 2 plugin-merge note: explicit guidance, not a TODO.
- Task 8 CKEditor v41+ API note: explicit compatibility note.
- Task 11 deferral to Plan 5: explicit bounded dependency.
- Task 12, 14, 15 conditional steps: all bounded.
- No `TBD`/`FIXME`/handwave.

**3. Type consistency:**
- `CKEditorBridge` extends `AbstractWysiwygBridge` (Plan 1 Task 38).
- `CKEditorConfig` extends `AbstractWysiwygConfig` (Plan 1 Task 36); uses `CommonOptions` (Plan 1 Task 8).
- `CKEditorTransformer` extends `AbstractWysiwygTransformer` (Plan 1 Task 37).
- Service tags match Plan 1 core.
- Stimulus id matches `AbstractBridge::getControllerName()` derivation.
- JS controller imports from `@symfony/ux-editor/format/wysiwyg`.

No inline fixes needed.

---

## Execution handoff

Plan 3 saved to `docs/superpowers/plans/2026-05-19-ux-editor-bridge-ckeditor.md`.

Execution options:

1. **Subagent-Driven (recommended)** — `superpowers:subagent-driven-development`.
2. **Inline** — `superpowers:executing-plans` with checkpoints after Tasks 4, 7, 10, 17.

Plans 4-5 follow:
- Plan 4: GrapesJS (Tier 2 Page).
- Plan 5: Demo app + LiveComponent showcase.

Which approach for Plan 3?
