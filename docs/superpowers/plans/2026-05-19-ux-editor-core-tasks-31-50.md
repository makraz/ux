# UX Editor Core — Tasks 31-50 (expanded)

> **Parent plan:** [`2026-05-19-ux-editor-core.md`](2026-05-19-ux-editor-core.md). Continues from [`-tasks-7-30.md`](2026-05-19-ux-editor-core-tasks-7-30.md). Requires Tasks 1-30 complete.

Coverage: LiveComponent trait, Twig render extension, Tier 1 Wysiwyg / Block / Page abstracts (PHP only).

---

## Task 31 — `LiveEditor` trait

**Files:**
- Create: `src/Editor/src/Live/LiveEditor.php`
- Test: `src/Editor/tests/Live/LiveEditorTraitTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Live;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Live\LiveEditor;

final class LiveEditorTraitTest extends TestCase
{
    public function testSaveDraftMarksFieldCleanAndStampsTime(): void
    {
        $host = $this->host();
        self::assertTrue($host->isDirty('body'));
        $host->saveDraft('body', 'hello');
        self::assertFalse($host->isDirty('body'));
        self::assertSame('hello', $host->draftRepo->store['42']['body']);
        self::assertNotNull($host->getLastSavedAt('body'));
    }

    public function testMarkDirtyExplicit(): void
    {
        $host = $this->host();
        $host->saveDraft('body', 'x');
        self::assertFalse($host->isDirty('body'));
        $host->markDirty('body');
        self::assertTrue($host->isDirty('body'));
    }

    private function host(): object
    {
        return new class { use LiveEditor; public function __construct() { $this->draftRepo = new InMemoryDrafts(); } public function getEntityId(): string { return '42'; } };
    }
}

final class InMemoryDrafts
{
    public array $store = [];
    public function upsert(string $entityId, string $field, mixed $content): void { $this->store[$entityId][$field] = $content; }
}
```

- [ ] **Step 2: Run — expect fail**

```bash
vendor/bin/phpunit tests/Live/LiveEditorTraitTest.php
```

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Live;

trait LiveEditor
{
    /** @var array<string, bool> */                public array $dirty = [];
    /** @var array<string, \DateTimeImmutable> */ public array $lastSavedAt = [];

    /** Host class assigns this in its constructor — any object with upsert(entityId, field, content) signature */
    public mixed $draftRepo;

    public function saveDraft(string $field, mixed $content): void
    {
        $this->draftRepo->upsert($this->getEntityId(), $field, $content);
        $this->dirty[$field] = false;
        $this->lastSavedAt[$field] = new \DateTimeImmutable();
    }

    public function isDirty(string $field): bool { return $this->dirty[$field] ?? true; }
    public function markDirty(string $field): void { $this->dirty[$field] = true; }
    public function getLastSavedAt(string $field): ?\DateTimeImmutable { return $this->lastSavedAt[$field] ?? null; }

    abstract public function getEntityId(): string;
}
```

> The `#[LiveAction]` attribute is applied by host components (`#[LiveAction] public function saveDraft(...)`) since attribute placement on trait methods is not portable across all LiveComponent versions. Integration test in Task 32 verifies host wiring.

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Live/LiveEditor.php src/Editor/tests/Live/LiveEditorTraitTest.php
git commit -m "feat(editor): add LiveEditor trait with saveDraft + dirty tracking"
```

---

## Task 32 — `LiveEditor` integration via composition

**Files:**
- Test: `src/Editor/tests/Live/LiveEditorIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Live;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Live\LiveEditor;

final class LiveEditorIntegrationTest extends TestCase
{
    public function testHostUsingTraitInheritsAllMethods(): void
    {
        $repo = new class { public array $store = []; public function upsert(string $id, string $field, mixed $content): void { $this->store[$id][$field] = $content; } };
        $host = new FakeArticleEditor($repo, 'a-1');

        self::assertTrue($host->isDirty('body'));
        $host->saveDraft('body', '<p>hello</p>');
        self::assertFalse($host->isDirty('body'));
        self::assertSame('<p>hello</p>', $repo->store['a-1']['body']);

        $host->saveDraft('title', 'My title');
        self::assertFalse($host->isDirty('title'));
        self::assertFalse($host->isDirty('body'));

        $host->markDirty('body');
        self::assertTrue($host->isDirty('body'));
    }
}

final class FakeArticleEditor
{
    use LiveEditor;
    public function __construct(public mixed $repo, private string $entityId) { $this->draftRepo = $repo; }
    public function getEntityId(): string { return $this->entityId; }
}
```

- [ ] **Step 2: Run — expect 1 pass** (trait already provides behavior).

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Live/LiveEditorIntegrationTest.php
git commit -m "test(editor): LiveEditor composes into host components"
```

---

## Task 33 — `EditorRenderExtension` Twig (Html + Blocks)

**Files:**
- Create: `src/Editor/src/Twig/EditorRenderExtension.php`
- Create: `src/Editor/src/Bridge/Format/Block/BlockRendererInterface.php` (final shape used in Task 43)
- Create: `src/Editor/src/Bridge/Format/Block/BlockRendererRegistry.php` (final shape used in Task 43)
- Test: `src/Editor/tests/Twig/EditorRenderExtensionTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Twig;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererRegistry;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Twig\EditorRenderExtension;

final class EditorRenderExtensionTest extends TestCase
{
    public function testNullReturnsEmpty(): void
    {
        self::assertSame('', (new EditorRenderExtension(new BlockRendererRegistry([]), null))->render(null));
    }

    public function testHtmlContentSanitized(): void
    {
        $s = new HtmlSanitizer((new HtmlSanitizerConfig())->allowSafeElements());
        $out = (new EditorRenderExtension(new BlockRendererRegistry([]), $s))->render(new HtmlContent('<script>x</script><p>ok</p>'));
        self::assertStringNotContainsString('<script>', $out);
        self::assertStringContainsString('<p>ok</p>', $out);
    }

    public function testHtmlContentNoSanitizerEchosRaw(): void
    {
        self::assertSame('<p>x</p>', (new EditorRenderExtension(new BlockRendererRegistry([]), null))->render(new HtmlContent('<p>x</p>')));
    }

    public function testBlockContentWalksRegistry(): void
    {
        $registry = new BlockRendererRegistry([
            new class implements BlockRendererInterface {
                public function getBlockType(): string { return 'paragraph'; }
                public function render(array $blockData, array $blockMeta = []): string { return '<p>'.htmlspecialchars($blockData['text'] ?? '').'</p>'; }
            },
        ]);
        $bc = new BlockContent([['type' => 'paragraph', 'data' => ['text' => 'hello']]]);
        self::assertSame('<p>hello</p>', (new EditorRenderExtension($registry, null))->render($bc));
    }

    public function testBlockContentMissingRendererCommentInProd(): void
    {
        $e = new EditorRenderExtension(new BlockRendererRegistry([]), null, debug: false);
        self::assertSame('<!-- ux-editor: missing renderer for "unknown" -->', $e->render(new BlockContent([['type' => 'unknown', 'data' => []]])));
    }

    public function testBlockContentMissingRendererVisibleInDebug(): void
    {
        $e = new EditorRenderExtension(new BlockRendererRegistry([]), null, debug: true);
        $out = $e->render(new BlockContent([['type' => 'unknown', 'data' => []]]));
        self::assertStringContainsString('Missing block renderer', $out);
        self::assertStringContainsString('unknown', $out);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`src/Bridge/Format/Block/BlockRendererInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

interface BlockRendererInterface
{
    public function getBlockType(): string;
    public function render(array $blockData, array $blockMeta = []): string;
}
```

`src/Bridge/Format/Block/BlockRendererRegistry.php`:
```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

final class BlockRendererRegistry
{
    /** @var array<string, BlockRendererInterface> */
    private array $byType = [];

    /** @param iterable<BlockRendererInterface> $renderers */
    public function __construct(iterable $renderers = [])
    {
        foreach ($renderers as $r) { $this->byType[$r->getBlockType()] = $r; }
    }

    public function get(string $type): ?BlockRendererInterface { return $this->byType[$type] ?? null; }
}
```

`src/Twig/EditorRenderExtension.php`:
```php
<?php
namespace Symfony\UX\Editor\Twig;

use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererRegistry;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Content\PageContent;
use Twig\Extension\AbstractExtension;
use Twig\TwigFunction;

final class EditorRenderExtension extends AbstractExtension
{
    public function __construct(
        private readonly BlockRendererRegistry $blockRenderers,
        private readonly ?HtmlSanitizerInterface $sanitizer = null,
        private readonly bool $debug = false,
    ) {}

    public function getFunctions(): array
    {
        return [new TwigFunction('ux_editor_render', $this->render(...), ['is_safe' => ['html']])];
    }

    public function render(?EditorContentInterface $content, array $options = []): string
    {
        if ($content === null || $content->isEmpty()) { return ''; }
        return match (true) {
            $content instanceof HtmlContent  => $this->renderHtml($content),
            $content instanceof BlockContent => $this->renderBlocks($content),
            $content instanceof PageContent  => $this->renderPage($content, $options),
            default                          => '',
        };
    }

    private function renderHtml(HtmlContent $c): string { return $c->getSanitized($this->sanitizer); }

    private function renderBlocks(BlockContent $c): string
    {
        $out = '';
        foreach ($c->blocks as $block) {
            $type = (string)($block['type'] ?? '');
            $r = $this->blockRenderers->get($type);
            if ($r === null) {
                $out .= $this->debug
                    ? sprintf('<div style="border:1px dashed red;padding:.5em;background:#fee">Missing block renderer for type "%s"</div>', htmlspecialchars($type))
                    : sprintf('<!-- ux-editor: missing renderer for "%s" -->', $type);
                continue;
            }
            $out .= $r->render($block['data'] ?? [], $block);
        }
        return $out;
    }

    private function renderPage(PageContent $c, array $options): string
    {
        // Implemented in Task 34
        return '';
    }
}
```

- [ ] **Step 4: Run — expect 6 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Twig/EditorRenderExtension.php src/Editor/src/Bridge/Format/Block/BlockRendererInterface.php src/Editor/src/Bridge/Format/Block/BlockRendererRegistry.php src/Editor/tests/Twig/EditorRenderExtensionTest.php
git commit -m "feat(editor): add EditorRenderExtension (Html + Blocks)"
```

---

## Task 34 — Page render via sandboxed iframe `srcdoc`

**Files:**
- Modify: `src/Editor/src/Twig/EditorRenderExtension.php`
- Test: `src/Editor/tests/Twig/EditorRenderExtensionPageTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Twig;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererRegistry;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Twig\EditorRenderExtension;

final class EditorRenderExtensionPageTest extends TestCase
{
    public function testPageProducesSandboxedIframe(): void
    {
        $out = (new EditorRenderExtension(new BlockRendererRegistry([]), null))->render(new PageContent('<h1>X</h1>', 'h1{color:red}'));
        self::assertStringContainsString('<iframe', $out);
        self::assertStringContainsString('sandbox="allow-same-origin"', $out);
        self::assertStringContainsString('srcdoc=', $out);
        self::assertStringContainsString('&lt;h1&gt;X&lt;/h1&gt;', $out);
        self::assertStringContainsString('h1{color:red}', $out);
    }

    public function testEmptyPageIsEmpty(): void
    {
        self::assertSame('', (new EditorRenderExtension(new BlockRendererRegistry([]), null))->render(new PageContent('')));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — replace `renderPage`**

```php
private function renderPage(PageContent $c, array $options): string
{
    $srcdoc = sprintf(
        '<!doctype html><html><head><style>%s</style></head><body>%s</body></html>',
        $c->css,
        $c->html,
    );
    return sprintf(
        '<iframe sandbox="allow-same-origin" srcdoc="%s"></iframe>',
        htmlspecialchars($srcdoc, \ENT_QUOTES | \ENT_HTML5, 'UTF-8'),
    );
}
```

- [ ] **Step 4: Run — expect 2 pass + Task 33 tests still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Twig/EditorRenderExtension.php src/Editor/tests/Twig/EditorRenderExtensionPageTest.php
git commit -m "feat(editor): render PageContent via sandboxed iframe"
```

---

## Task 35 — `WysiwygCapabilities::default()` factory

**Files:**
- Create: `src/Editor/src/Bridge/Format/Wysiwyg/WysiwygCapabilities.php`
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygCapabilitiesTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;

final class WysiwygCapabilitiesTest extends TestCase
{
    public function testDefaults(): void
    {
        $c = WysiwygCapabilities::default();
        self::assertTrue($c->supportsToolbar);
        self::assertTrue($c->supportsPlugins);
        self::assertTrue($c->supportsTheme);
        self::assertTrue($c->supportsLanguage);
        self::assertSame(['html'], $c->supportedFormats);
    }

    public function testWithOverrides(): void
    {
        $c = WysiwygCapabilities::default()->with(supportsTheme: false);
        self::assertFalse($c->supportsTheme);
        self::assertTrue($c->supportsToolbar);
        self::assertSame(['html'], $c->supportedFormats);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Wysiwyg;

use Symfony\UX\Editor\Config\BridgeCapabilities;

final class WysiwygCapabilities
{
    public static function default(): BridgeCapabilities
    {
        return new BridgeCapabilities(
            supportsToolbar:  true,
            supportsPlugins:  true,
            supportsTheme:    true,
            supportsLanguage: true,
            supportedFormats: ['html'],
        );
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Wysiwyg/WysiwygCapabilities.php src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygCapabilitiesTest.php
git commit -m "feat(editor): add WysiwygCapabilities factory"
```

---

## Task 36 — `AbstractWysiwygConfig`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygConfig.php`
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygConfig;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class AbstractWysiwygConfigTest extends TestCase
{
    public function testDefaultTranslateCommon(): void
    {
        $cfg = new class(new CommonOptions(placeholder: 'Write…', readOnly: true, language: 'fr', plugins: ['link'])) extends AbstractWysiwygConfig {
            public function getBridgeId(): string { return 'fake'; }
        };
        $n = $cfg->toNative();
        self::assertSame('Write…', $n['placeholder']);
        self::assertTrue($n['readOnly']);
        self::assertSame('fr', $n['language']);
        self::assertSame(['link'], $n['plugins']);
    }

    public function testCapabilitiesDefaultToWysiwyg(): void
    {
        $cfg = new class extends AbstractWysiwygConfig { public function getBridgeId(): string { return 'fake'; } };
        self::assertEquals(WysiwygCapabilities::default(), $cfg->getCapabilities());
        self::assertSame(['html'], $cfg->getCapabilities()->supportedFormats);
    }

    public function testSubclassCanExtendTranslateOwn(): void
    {
        $cfg = new class(new CommonOptions(placeholder: 'X')) extends AbstractWysiwygConfig {
            public function getBridgeId(): string { return 'extra'; }
            protected function translateOwn(): array { return ['custom' => 'value']; }
        };
        $n = $cfg->toNative();
        self::assertSame('X', $n['placeholder']);
        self::assertSame('value', $n['custom']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Wysiwyg;

use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

abstract class AbstractWysiwygConfig extends AbstractEditorConfig
{
    public function getCapabilities(): BridgeCapabilities { return WysiwygCapabilities::default(); }

    protected function translateCommon(CommonOptions $c): array
    {
        $out = [];
        if ($c->placeholder !== null) { $out['placeholder'] = $c->placeholder; }
        if ($c->readOnly)             { $out['readOnly']    = true; }
        if ($c->language !== null)    { $out['language']    = $c->language; }
        if ($c->plugins !== [])       { $out['plugins']     = $c->plugins; }
        if ($c->theme !== null)       { $out['theme']       = $c->theme; }
        if ($c->toolbar !== null)     { $out['toolbar']     = $c->toolbar; }
        if ($c->height !== null)      { $out['height']      = $c->height; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygConfig.php src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygConfigTest.php
git commit -m "feat(editor): add AbstractWysiwygConfig"
```

---

## Task 37 — `AbstractWysiwygTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygTransformer.php`
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygTransformer;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class AbstractWysiwygTransformerTest extends TestCase
{
    public function testStorageShapeIsScalar(): void
    {
        $t = $this->fake();
        self::assertSame(StorageShape::Scalar, $t->getStorageShape());
        self::assertSame(HtmlContent::class, $t->getContentClass());
    }

    public function testRoundTrip(): void
    {
        $t = $this->fake();
        self::assertSame('<p>hi</p>', $t->transform(new HtmlContent('<p>hi</p>')));
        self::assertNull($t->transform(null));
        $back = $t->reverseTransform('<p>hi</p>');
        self::assertInstanceOf(HtmlContent::class, $back);
        self::assertSame('<p>hi</p>', $back->html);
        self::assertNull($t->reverseTransform(null));
        self::assertNull($t->reverseTransform(''));
    }

    public function testMetadataAttachedOnReverse(): void
    {
        self::assertSame('fake', $this->fake()->reverseTransform('<p>x</p>')->getMetadata()['bridgeId']);
    }

    private function fake(): AbstractWysiwygTransformer
    {
        return new class extends AbstractWysiwygTransformer { public function getBridgeId(): string { return 'fake'; } };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Wysiwyg;

use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

abstract class AbstractWysiwygTransformer implements EditorContentTransformerInterface
{
    public function getContentClass(): string  { return HtmlContent::class; }
    public function getStorageShape(): StorageShape { return StorageShape::Scalar; }

    public function transform(?EditorContentInterface $content): ?string
    {
        if ($content === null) { return null; }
        if (!$content instanceof HtmlContent) {
            throw new \InvalidArgumentException(sprintf('Expected HtmlContent, got %s', get_debug_type($content)));
        }
        return $content->html;
    }

    public function reverseTransform(mixed $stored): ?HtmlContent
    {
        if ($stored === null || $stored === '') { return null; }
        return new HtmlContent((string)$stored, ['bridgeId' => $this->getBridgeId()]);
    }

    abstract public function getBridgeId(): string;
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygTransformer.php src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygTransformerTest.php
git commit -m "feat(editor): add AbstractWysiwygTransformer"
```

---

## Task 38 — `AbstractWysiwygBridge`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygBridge.php`
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygBridge;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygConfig;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygTransformer;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class AbstractWysiwygBridgeTest extends TestCase
{
    public function testDefaults(): void
    {
        $b = new class extends AbstractWysiwygBridge {
            public function getId(): string { return 'fakewy'; }
            public function getDefaultConfig(): EditorConfigInterface {
                return new class extends AbstractWysiwygConfig { public function getBridgeId(): string { return 'fakewy'; } };
            }
            public function createTransformer(): EditorContentTransformerInterface {
                return new class extends AbstractWysiwygTransformer { public function getBridgeId(): string { return 'fakewy'; } };
            }
        };
        self::assertSame('symfony--ux-editor--fakewy', $b->getControllerName());
        self::assertEquals(WysiwygCapabilities::default(), $b->getCapabilities());
        self::assertInstanceOf(AbstractWysiwygConfig::class, $b->getDefaultConfig());
        self::assertInstanceOf(EditorContentTransformerInterface::class, $b->createTransformer());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Wysiwyg;

use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Config\BridgeCapabilities;

abstract class AbstractWysiwygBridge extends AbstractBridge
{
    public function getCapabilities(): BridgeCapabilities { return WysiwygCapabilities::default(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Wysiwyg/AbstractWysiwygBridge.php src/Editor/tests/Bridge/Format/Wysiwyg/AbstractWysiwygBridgeTest.php
git commit -m "feat(editor): add AbstractWysiwygBridge"
```

---

## Task 39 — Wysiwyg end-to-end via `EditorType`

**Files:**
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygBridge;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygConfig;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygTransformer;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\EditorType;

final class WysiwygIntegrationTest extends TestCase
{
    public function testFullRoundTrip(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new FakeWysiwygBridge()]), new PresetRegistry([])))
            ->getFormFactory();
        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config' => new FakeWysiwygConfig(new CommonOptions(placeholder: 'Write…')),
        ])->getForm();

        $form->submit(['body' => '<p>hello</p>']);
        self::assertTrue($form->isSynchronized());
        $data = $form->get('body')->getData();
        self::assertInstanceOf(HtmlContent::class, $data);
        self::assertSame('<p>hello</p>', $data->html);
        self::assertSame('fakewy', $data->getMetadata()['bridgeId']);

        $attr = $form->get('body')->createView()->vars['attr'];
        $native = json_decode($attr['data-symfony--ux-editor--fakewy-config-value'], true);
        self::assertSame('Write…', $native['placeholder']);
    }
}

final class FakeWysiwygConfig extends AbstractWysiwygConfig
{
    public function getBridgeId(): string { return 'fakewy'; }
}

final class FakeWysiwygBridge extends AbstractWysiwygBridge
{
    public function getId(): string { return 'fakewy'; }
    public function getDefaultConfig(): EditorConfigInterface { return new FakeWysiwygConfig(); }
    public function createTransformer(): EditorContentTransformerInterface
    {
        return new class extends AbstractWysiwygTransformer { public function getBridgeId(): string { return 'fakewy'; } };
    }
}
```

- [ ] **Step 2: Run — expect 1 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygIntegrationTest.php
git commit -m "test(editor): end-to-end WYSIWYG round-trip"
```

---

## Task 40 — Capability warning for incompatible toolbar

**Files:**
- Test: `src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygCapabilityWarningTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Wysiwyg;

use PHPUnit\Framework\TestCase;
use Psr\Log\AbstractLogger;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\AbstractWysiwygConfig;
use Symfony\UX\Editor\Bridge\Format\Wysiwyg\WysiwygCapabilities;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class WysiwygCapabilityWarningTest extends TestCase
{
    public function testWarnsWhenToolbarUnsupported(): void
    {
        $logger = new class extends AbstractLogger {
            public array $log = [];
            public function log($level, \Stringable|string $msg, array $ctx = []): void { $this->log[] = [$level, (string)$msg]; }
        };
        $cfg = new class(new CommonOptions(toolbar: ['bold'])) extends AbstractWysiwygConfig {
            public function getBridgeId(): string { return 'notoolbar'; }
            public function getCapabilities(): BridgeCapabilities { return WysiwygCapabilities::default()->with(supportsToolbar: false); }
        };
        $cfg->setLogger($logger);
        $cfg->toNative();

        self::assertNotEmpty($logger->log);
        self::assertSame('warning', $logger->log[0][0]);
        self::assertStringContainsString('toolbar', $logger->log[0][1]);
    }
}
```

- [ ] **Step 2: Run — expect 1 pass** (Task 12 already wired the guard).

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Bridge/Format/Wysiwyg/WysiwygCapabilityWarningTest.php
git commit -m "test(editor): Wysiwyg config warns on unsupported toolbar"
```

---

## Task 41 — `BlockCapabilities::default()`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Block/BlockCapabilities.php`
- Test: `src/Editor/tests/Bridge/Format/Block/BlockCapabilitiesTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\BlockCapabilities;

final class BlockCapabilitiesTest extends TestCase
{
    public function testDefaults(): void
    {
        $c = BlockCapabilities::default();
        self::assertFalse($c->supportsToolbar);
        self::assertTrue($c->supportsPlugins);
        self::assertFalse($c->supportsTheme);
        self::assertTrue($c->supportsLanguage);
        self::assertSame(['blocks'], $c->supportedFormats);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

use Symfony\UX\Editor\Config\BridgeCapabilities;

final class BlockCapabilities
{
    public static function default(): BridgeCapabilities
    {
        return new BridgeCapabilities(
            supportsToolbar:  false,
            supportsPlugins:  true,
            supportsTheme:    false,
            supportsLanguage: true,
            supportedFormats: ['blocks'],
        );
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Block/BlockCapabilities.php src/Editor/tests/Bridge/Format/Block/BlockCapabilitiesTest.php
git commit -m "feat(editor): add BlockCapabilities factory"
```

---

## Task 42 — `AbstractBlockConfig`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Block/AbstractBlockConfig.php`
- Test: `src/Editor/tests/Bridge/Format/Block/AbstractBlockConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockConfig;
use Symfony\UX\Editor\Bridge\Format\Block\BlockCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class AbstractBlockConfigTest extends TestCase
{
    public function testDefaultTranslateCommon(): void
    {
        $cfg = new class(new CommonOptions(placeholder: 'P', autofocus: true, readOnly: true)) extends AbstractBlockConfig {
            public function getBridgeId(): string { return 'fb'; }
        };
        $n = $cfg->toNative();
        self::assertSame('P', $n['placeholder']);
        self::assertTrue($n['autofocus']);
        self::assertTrue($n['readOnly']);
    }

    public function testCapabilitiesAreBlockFamily(): void
    {
        $cfg = new class extends AbstractBlockConfig { public function getBridgeId(): string { return 'fb'; } };
        self::assertEquals(BlockCapabilities::default(), $cfg->getCapabilities());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

abstract class AbstractBlockConfig extends AbstractEditorConfig
{
    public function getCapabilities(): BridgeCapabilities { return BlockCapabilities::default(); }

    protected function translateCommon(CommonOptions $c): array
    {
        $out = [];
        if ($c->placeholder !== null) { $out['placeholder'] = $c->placeholder; }
        if ($c->readOnly)             { $out['readOnly']    = true; }
        if ($c->autofocus)            { $out['autofocus']   = true; }
        if ($c->language !== null)    { $out['language']    = $c->language; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Block/AbstractBlockConfig.php src/Editor/tests/Bridge/Format/Block/AbstractBlockConfigTest.php
git commit -m "feat(editor): add AbstractBlockConfig"
```

---

## Task 43 — Cover `BlockRendererRegistry` final shape

**Files:**
- Test: `src/Editor/tests/Bridge/Format/Block/BlockRendererRegistryTest.php`

> Registry final shape already created in Task 33. This task adds dedicated coverage.

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererInterface;
use Symfony\UX\Editor\Bridge\Format\Block\BlockRendererRegistry;

final class BlockRendererRegistryTest extends TestCase
{
    public function testGetByType(): void
    {
        $r = new class implements BlockRendererInterface {
            public function getBlockType(): string { return 'header'; }
            public function render(array $blockData, array $blockMeta = []): string { return '<h>'.($blockData['text'] ?? '').'</h>'; }
        };
        $reg = new BlockRendererRegistry([$r]);
        self::assertSame($r, $reg->get('header'));
        self::assertNull($reg->get('unknown'));
    }

    public function testLastWriterWins(): void
    {
        $a = new class implements BlockRendererInterface {
            public function getBlockType(): string { return 'p'; }
            public function render(array $blockData, array $blockMeta = []): string { return 'A'; }
        };
        $b = new class implements BlockRendererInterface {
            public function getBlockType(): string { return 'p'; }
            public function render(array $blockData, array $blockMeta = []): string { return 'B'; }
        };
        self::assertSame('B', (new BlockRendererRegistry([$a, $b]))->get('p')->render([]));
    }
}
```

- [ ] **Step 2: Run — expect 2 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Bridge/Format/Block/BlockRendererRegistryTest.php
git commit -m "test(editor): cover BlockRendererRegistry final shape"
```

---

## Task 44 — `AbstractBlockTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Block/AbstractBlockTransformer.php`
- Test: `src/Editor/tests/Bridge/Format/Block/AbstractBlockTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockTransformer;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Exception\ContentSchemaException;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class AbstractBlockTransformerTest extends TestCase
{
    public function testStorageShapeIsJson(): void
    {
        self::assertSame(StorageShape::Json, $this->fake()->getStorageShape());
        self::assertSame(BlockContent::class, $this->fake()->getContentClass());
    }

    public function testRoundTripFromArray(): void
    {
        $t = $this->fake();
        $bc = new BlockContent([['type' => 'p', 'data' => ['text' => 'hi']]], '2.0');
        $arr = $t->transform($bc);
        self::assertSame('2.0', $arr['version']);
        self::assertSame([['type' => 'p', 'data' => ['text' => 'hi']]], $arr['blocks']);

        $back = $t->reverseTransform($arr);
        self::assertInstanceOf(BlockContent::class, $back);
        self::assertSame('2.0', $back->schemaVersion);
        self::assertSame('fakeblock', $back->getMetadata()['bridgeId']);
    }

    public function testNullPaths(): void
    {
        $t = $this->fake();
        self::assertNull($t->transform(null));
        self::assertNull($t->reverseTransform(null));
        self::assertNull($t->reverseTransform([]));
    }

    public function testMalformedThrows(): void
    {
        $this->expectException(ContentSchemaException::class);
        $this->fake()->reverseTransform(['blocks' => 'not-an-array']);
    }

    private function fake(): AbstractBlockTransformer
    {
        return new class extends AbstractBlockTransformer { public function getBridgeId(): string { return 'fakeblock'; } };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Exception\ContentSchemaException;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

abstract class AbstractBlockTransformer implements EditorContentTransformerInterface
{
    public function getContentClass(): string { return BlockContent::class; }
    public function getStorageShape(): StorageShape { return StorageShape::Json; }

    public function transform(?EditorContentInterface $content): ?array
    {
        if ($content === null) { return null; }
        if (!$content instanceof BlockContent) {
            throw new \InvalidArgumentException(sprintf('Expected BlockContent, got %s', get_debug_type($content)));
        }
        return ['version' => $content->schemaVersion, 'blocks' => $content->blocks];
    }

    public function reverseTransform(mixed $stored): ?BlockContent
    {
        if ($stored === null || $stored === []) { return null; }
        if (!\is_array($stored)) {
            throw new ContentSchemaException(sprintf('Expected array, got %s', get_debug_type($stored)));
        }
        $blocks = $stored['blocks'] ?? [];
        if (!\is_array($blocks)) {
            throw new ContentSchemaException('"blocks" must be an array');
        }
        return new BlockContent(
            blocks: $blocks,
            schemaVersion: (string)($stored['version'] ?? '1.0'),
            metadata: ['bridgeId' => $this->getBridgeId()],
        );
    }

    abstract public function getBridgeId(): string;
}
```

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Block/AbstractBlockTransformer.php src/Editor/tests/Bridge/Format/Block/AbstractBlockTransformerTest.php
git commit -m "feat(editor): add AbstractBlockTransformer"
```

---

## Task 45 — `AbstractBlockBridge`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Block/AbstractBlockBridge.php`
- Test: `src/Editor/tests/Bridge/Format/Block/AbstractBlockBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockBridge;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockConfig;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockTransformer;
use Symfony\UX\Editor\Bridge\Format\Block\BlockCapabilities;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class AbstractBlockBridgeTest extends TestCase
{
    public function testDefaults(): void
    {
        $b = new class extends AbstractBlockBridge {
            public function getId(): string { return 'fb'; }
            public function getDefaultConfig(): EditorConfigInterface {
                return new class extends AbstractBlockConfig { public function getBridgeId(): string { return 'fb'; } };
            }
            public function createTransformer(): EditorContentTransformerInterface {
                return new class extends AbstractBlockTransformer { public function getBridgeId(): string { return 'fb'; } };
            }
        };
        self::assertSame('symfony--ux-editor--fb', $b->getControllerName());
        self::assertEquals(BlockCapabilities::default(), $b->getCapabilities());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Block;

use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Config\BridgeCapabilities;

abstract class AbstractBlockBridge extends AbstractBridge
{
    public function getCapabilities(): BridgeCapabilities { return BlockCapabilities::default(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Block/AbstractBlockBridge.php src/Editor/tests/Bridge/Format/Block/AbstractBlockBridgeTest.php
git commit -m "feat(editor): add AbstractBlockBridge"
```

---

## Task 46 — Block end-to-end via `EditorType`

**Files:**
- Test: `src/Editor/tests/Bridge/Format/Block/BlockIntegrationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Block;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockBridge;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockConfig;
use Symfony\UX\Editor\Bridge\Format\Block\AbstractBlockTransformer;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\EditorType;

final class BlockIntegrationTest extends TestCase
{
    public function testJsonRoundTripThroughEditorType(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new FakeBlockBridge()]), new PresetRegistry([])))
            ->getFormFactory();
        $form = $factory->createBuilder()->add('body', EditorType::class, ['config' => new FakeBlockConfig()])->getForm();
        $payload = json_encode(['version' => '1.0', 'blocks' => [['type' => 'paragraph', 'data' => ['text' => 'hi']]]], \JSON_THROW_ON_ERROR);
        $form->submit(['body' => $payload]);
        self::assertTrue($form->isSynchronized());
        $data = $form->get('body')->getData();
        self::assertInstanceOf(BlockContent::class, $data);
        self::assertSame('paragraph', $data->blocks[0]['type']);
    }
}

final class FakeBlockConfig extends AbstractBlockConfig
{
    public function getBridgeId(): string { return 'fakeblock'; }
}

final class FakeBlockBridge extends AbstractBlockBridge
{
    public function getId(): string { return 'fakeblock'; }
    public function getDefaultConfig(): EditorConfigInterface { return new FakeBlockConfig(); }
    public function createTransformer(): EditorContentTransformerInterface
    {
        return new class extends AbstractBlockTransformer { public function getBridgeId(): string { return 'fakeblock'; } };
    }
}
```

- [ ] **Step 2: Run — expect 1 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Bridge/Format/Block/BlockIntegrationTest.php
git commit -m "test(editor): end-to-end Block round-trip"
```

---

## Task 47 — `PageCapabilities::default()`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/PageCapabilities.php`
- Test: `src/Editor/tests/Bridge/Format/Page/PageCapabilitiesTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\PageCapabilities;

final class PageCapabilitiesTest extends TestCase
{
    public function testDefaults(): void
    {
        $c = PageCapabilities::default();
        self::assertFalse($c->supportsToolbar);
        self::assertTrue($c->supportsPlugins);
        self::assertTrue($c->supportsTheme);
        self::assertTrue($c->supportsLanguage);
        self::assertSame(['page'], $c->supportedFormats);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Config\BridgeCapabilities;

final class PageCapabilities
{
    public static function default(): BridgeCapabilities
    {
        return new BridgeCapabilities(
            supportsToolbar:  false,
            supportsPlugins:  true,
            supportsTheme:    true,
            supportsLanguage: true,
            supportedFormats: ['page'],
        );
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/PageCapabilities.php src/Editor/tests/Bridge/Format/Page/PageCapabilitiesTest.php
git commit -m "feat(editor): add PageCapabilities factory"
```

---

## Task 48 — `AbstractPageConfig`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/AbstractPageConfig.php`
- Test: `src/Editor/tests/Bridge/Format/Page/AbstractPageConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageConfig;
use Symfony\UX\Editor\Bridge\Format\Page\PageCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class AbstractPageConfigTest extends TestCase
{
    public function testCapabilitiesArePageFamily(): void
    {
        $cfg = new class extends AbstractPageConfig { public function getBridgeId(): string { return 'fp'; } };
        self::assertEquals(PageCapabilities::default(), $cfg->getCapabilities());
    }

    public function testTranslateCommonMinimal(): void
    {
        $cfg = new class(new CommonOptions(theme: 'dark', language: 'fr', placeholder: 'IGNORED')) extends AbstractPageConfig {
            public function getBridgeId(): string { return 'fp'; }
        };
        $n = $cfg->toNative();
        self::assertSame('dark', $n['theme']);
        self::assertSame('fr',   $n['language']);
        self::assertArrayNotHasKey('placeholder', $n);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

abstract class AbstractPageConfig extends AbstractEditorConfig
{
    public function getCapabilities(): BridgeCapabilities { return PageCapabilities::default(); }

    protected function translateCommon(CommonOptions $c): array
    {
        $out = [];
        if ($c->theme !== null)    { $out['theme']    = $c->theme; }
        if ($c->language !== null) { $out['language'] = $c->language; }
        return $out;
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/AbstractPageConfig.php src/Editor/tests/Bridge/Format/Page/AbstractPageConfigTest.php
git commit -m "feat(editor): add AbstractPageConfig"
```

---

## Task 49 — `PageAssetExtractor`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/PageAssetExtractor.php`
- Test: `src/Editor/tests/Bridge/Format/Page/PageAssetExtractorTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\PageAssetExtractor;
use Symfony\UX\Editor\Content\PageContent;

final class PageAssetExtractorTest extends TestCase
{
    public function testExtractsFromAssetsField(): void
    {
        $page = new PageContent('<p>x</p>', '', [
            ['type' => 'image', 'url' => '/a.png'],
            ['type' => 'image', 'url' => '/b.png'],
        ]);
        self::assertSame(['/a.png', '/b.png'], (new PageAssetExtractor())->extractUrls($page));
    }

    public function testWalksComponentTreeForSrc(): void
    {
        $page = new PageContent('', '', [], [
            ['type' => 'section', 'children' => [
                ['type' => 'image', 'src' => '/c.png'],
                ['type' => 'image', 'src' => '/d.png'],
            ]],
        ]);
        $urls = (new PageAssetExtractor())->extractUrls($page);
        sort($urls);
        self::assertSame(['/c.png', '/d.png'], $urls);
    }

    public function testDedupes(): void
    {
        $page = new PageContent('', '', [['type' => 'image', 'url' => '/x.png']], [['type' => 'image', 'src' => '/x.png']]);
        self::assertSame(['/x.png'], (new PageAssetExtractor())->extractUrls($page));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Content\PageContent;

final class PageAssetExtractor
{
    /** @return list<string> */
    public function extractUrls(PageContent $page): array
    {
        $urls = [];
        foreach ($page->assets as $asset) {
            if (\is_array($asset) && isset($asset['url']) && \is_string($asset['url'])) {
                $urls[$asset['url']] = true;
            }
        }
        $this->walk($page->components, $urls);
        return array_keys($urls);
    }

    /** @param array<mixed> $nodes */
    private function walk(array $nodes, array &$urls): void
    {
        foreach ($nodes as $n) {
            if (!\is_array($n)) { continue; }
            if (isset($n['src']) && \is_string($n['src']))  { $urls[$n['src']] = true; }
            if (isset($n['url']) && \is_string($n['url']))  { $urls[$n['url']] = true; }
            if (!empty($n['children']) && \is_array($n['children'])) {
                $this->walk($n['children'], $urls);
            }
        }
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/PageAssetExtractor.php src/Editor/tests/Bridge/Format/Page/PageAssetExtractorTest.php
git commit -m "feat(editor): add PageAssetExtractor"
```

---

## Task 50 — `PageSandboxRenderer`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/PageSandboxRenderer.php`
- Test: `src/Editor/tests/Bridge/Format/Page/PageSandboxRendererTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\PageSandboxRenderer;
use Symfony\UX\Editor\Content\PageContent;

final class PageSandboxRendererTest extends TestCase
{
    public function testSandboxedIframe(): void
    {
        $out = (new PageSandboxRenderer())->render(new PageContent('<h1>X</h1>', 'h1{color:red}'));
        self::assertStringContainsString('<iframe', $out);
        self::assertStringContainsString('sandbox="allow-same-origin"', $out);
        self::assertStringContainsString('srcdoc=', $out);
        self::assertStringContainsString('&lt;h1&gt;X&lt;/h1&gt;', $out);
        self::assertStringContainsString('h1{color:red}', $out);
    }

    public function testCustomSandbox(): void
    {
        $out = (new PageSandboxRenderer('allow-same-origin allow-scripts'))->render(new PageContent('<p>x</p>'));
        self::assertStringContainsString('sandbox="allow-same-origin allow-scripts"', $out);
    }

    public function testEmptyPageProducesEmptyString(): void
    {
        self::assertSame('', (new PageSandboxRenderer())->render(new PageContent('')));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Content\PageContent;

final class PageSandboxRenderer
{
    public function __construct(private readonly string $sandbox = 'allow-same-origin') {}

    public function render(PageContent $page): string
    {
        if ($page->isEmpty()) { return ''; }
        $srcdoc = sprintf(
            '<!doctype html><html><head><style>%s</style></head><body>%s</body></html>',
            $page->css,
            $page->html,
        );
        return sprintf(
            '<iframe sandbox="%s" srcdoc="%s"></iframe>',
            htmlspecialchars($this->sandbox, \ENT_QUOTES | \ENT_HTML5, 'UTF-8'),
            htmlspecialchars($srcdoc,        \ENT_QUOTES | \ENT_HTML5, 'UTF-8'),
        );
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/PageSandboxRenderer.php src/Editor/tests/Bridge/Format/Page/PageSandboxRendererTest.php
git commit -m "feat(editor): add PageSandboxRenderer"
```

---

## Checkpoint after Task 50

- [ ] **Run full suite**

```bash
cd src/Editor && vendor/bin/phpunit
```

Expected: all Tasks 2-50 pass; zero failures/errors/warnings.

- [ ] **Coverage spot-check**

```bash
vendor/bin/phpunit --coverage-text
```

Expected: ≥ 88% Tier 0 + Tier 1 PHP line coverage (excluding `AbstractPageTransformer` + `AbstractPageBuilderBridge` — Tasks 51-52).

---

## Self-review for Tasks 31-50

**1. Spec coverage:**
- §8 LiveComponent autosave → Tasks 31-32.
- §9 Twig render (Html + Blocks + Page) → Tasks 33-34.
- §3 Tier 1 Wysiwyg layer → Tasks 35-40.
- §3 Tier 1 Block layer → Tasks 41-46.
- §3 Tier 1 Page layer (capabilities, config, asset extractor, sandbox renderer) → Tasks 47-50.
- Remaining: `AbstractPageTransformer` (Task 51), `AbstractPageBuilderBridge` (Task 52), JS Tier 0 + Tier 1 (Tasks 53-65), bundle finalization (Tasks 66-72), docs/coverage (Tasks 73-78).

**2. Placeholder scan:** Task 33 introduces `BlockRendererInterface`/`Registry` at their final shape (not stubs); Task 43 adds dedicated coverage without modifying them. No `TODO`/`TBD`/handwave anywhere.

**3. Type consistency:**
- `BridgeCapabilities` ctor used identically in Tasks 35, 41, 47.
- `AbstractEditorConfig` extension chain: Wysiwyg/Block/Page configs all extend it, override `getCapabilities()` to return the family-specific `*Capabilities::default()`, implement `translateCommon`. ✓
- `EditorContentTransformerInterface` shape consistent in Tasks 37, 44 (Page transformer in Task 51).
- `PageContent`, `BlockContent`, `HtmlContent` shapes from Tasks 4-6 referenced consistently. ✓
- `StorageShape::Scalar/Json/Split` enum cases referenced uniformly. ✓

No inline fixes needed.

---

Next: `expand tasks 51-78` (AbstractPageTransformer + AbstractPageBuilderBridge + JS Tier 0/1 controllers + bundle finalization + docs/coverage).
