# UX Editor Core — Tasks 7-30 (expanded)

> **Parent plan:** [`2026-05-19-ux-editor-core.md`](2026-05-19-ux-editor-core.md). Same TDD 5-step cycle as Tasks 1-6. Requires Tasks 1-6 complete.

---

## Task 7 — Exception hierarchy

**Files:**
- Create: `src/Editor/src/Exception/EditorExceptionInterface.php`
- Create: `src/Editor/src/Exception/UnknownBridgeException.php`
- Create: `src/Editor/src/Exception/BridgeConfigMismatchException.php`
- Create: `src/Editor/src/Exception/IncompatibleConfigException.php`
- Create: `src/Editor/src/Exception/UnsupportedConversionException.php`
- Create: `src/Editor/src/Exception/ContentSchemaException.php`
- Create: `src/Editor/src/Exception/Upload/UploadException.php`
- Create: `src/Editor/src/Exception/Upload/InvalidSignatureException.php`
- Create: `src/Editor/src/Exception/Upload/UnsupportedFileException.php`
- Create: `src/Editor/src/Exception/Upload/UploadHandlerException.php`
- Test: `src/Editor/tests/Exception/ExceptionHierarchyTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Exception;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Exception\BridgeConfigMismatchException;
use Symfony\UX\Editor\Exception\ContentSchemaException;
use Symfony\UX\Editor\Exception\EditorExceptionInterface;
use Symfony\UX\Editor\Exception\IncompatibleConfigException;
use Symfony\UX\Editor\Exception\UnknownBridgeException;
use Symfony\UX\Editor\Exception\UnsupportedConversionException;
use Symfony\UX\Editor\Exception\Upload\InvalidSignatureException;
use Symfony\UX\Editor\Exception\Upload\UnsupportedFileException;
use Symfony\UX\Editor\Exception\Upload\UploadException;
use Symfony\UX\Editor\Exception\Upload\UploadHandlerException;

final class ExceptionHierarchyTest extends TestCase
{
    /** @dataProvider exceptionClasses */
    public function testAllExceptionsImplementMarker(string $class): void
    {
        $e = new $class('msg');
        self::assertInstanceOf(EditorExceptionInterface::class, $e);
        self::assertInstanceOf(\Throwable::class, $e);
    }

    public static function exceptionClasses(): array
    {
        return [
            [UnknownBridgeException::class],
            [BridgeConfigMismatchException::class],
            [IncompatibleConfigException::class],
            [UnsupportedConversionException::class],
            [ContentSchemaException::class],
            [UploadException::class],
            [InvalidSignatureException::class],
            [UnsupportedFileException::class],
            [UploadHandlerException::class],
        ];
    }

    public function testUploadExceptionsExtendUploadException(): void
    {
        self::assertInstanceOf(UploadException::class, new InvalidSignatureException('x'));
        self::assertInstanceOf(UploadException::class, new UnsupportedFileException('x'));
        self::assertInstanceOf(UploadException::class, new UploadHandlerException('x'));
    }
}
```

- [ ] **Step 2: Run — expect "Class not found"**

```bash
cd src/Editor && vendor/bin/phpunit tests/Exception/ExceptionHierarchyTest.php
```

- [ ] **Step 3: Implementation**

`EditorExceptionInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

interface EditorExceptionInterface extends \Throwable {}
```

`UnknownBridgeException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

final class UnknownBridgeException extends \InvalidArgumentException implements EditorExceptionInterface {}
```

`BridgeConfigMismatchException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

final class BridgeConfigMismatchException extends \LogicException implements EditorExceptionInterface {}
```

`IncompatibleConfigException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

final class IncompatibleConfigException extends \RuntimeException implements EditorExceptionInterface {}
```

`UnsupportedConversionException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

final class UnsupportedConversionException extends \RuntimeException implements EditorExceptionInterface {}
```

`ContentSchemaException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception;

final class ContentSchemaException extends \RuntimeException implements EditorExceptionInterface {}
```

`Upload/UploadException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception\Upload;

use Symfony\UX\Editor\Exception\EditorExceptionInterface;

class UploadException extends \RuntimeException implements EditorExceptionInterface {}
```

`Upload/InvalidSignatureException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception\Upload;

final class InvalidSignatureException extends UploadException {}
```

`Upload/UnsupportedFileException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception\Upload;

final class UnsupportedFileException extends UploadException {}
```

`Upload/UploadHandlerException.php`:
```php
<?php
namespace Symfony\UX\Editor\Exception\Upload;

final class UploadHandlerException extends UploadException {}
```

- [ ] **Step 4: Run — expect 10 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Exception/ src/Editor/tests/Exception/
git commit -m "feat(editor): add exception hierarchy"
```

---

## Task 8 — `CommonOptions` DTO

**Files:**
- Create: `src/Editor/src/Config/CommonOptions.php`
- Test: `src/Editor/tests/Config/CommonOptionsTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Config\CommonOptions;

final class CommonOptionsTest extends TestCase
{
    public function testDefaults(): void
    {
        $o = new CommonOptions();
        self::assertNull($o->toolbar);
        self::assertNull($o->placeholder);
        self::assertFalse($o->readOnly);
        self::assertNull($o->height);
        self::assertNull($o->theme);
        self::assertNull($o->language);
        self::assertSame([], $o->plugins);
        self::assertFalse($o->autofocus);
        self::assertTrue($o->spellcheck);
    }

    public function testNamedArgsConstruction(): void
    {
        $o = new CommonOptions(
            toolbar: ['bold', 'italic'],
            placeholder: 'Write…',
            readOnly: true,
            height: '400px',
            theme: 'dark',
            language: 'fr',
            plugins: ['image', 'link'],
            autofocus: true,
            spellcheck: false,
        );
        self::assertSame(['bold', 'italic'], $o->toolbar);
        self::assertSame('Write…', $o->placeholder);
        self::assertTrue($o->readOnly);
        self::assertSame('400px', $o->height);
        self::assertSame('dark', $o->theme);
        self::assertSame('fr', $o->language);
        self::assertSame(['image', 'link'], $o->plugins);
        self::assertTrue($o->autofocus);
        self::assertFalse($o->spellcheck);
    }

    public function testFromArrayMapsKeys(): void
    {
        $o = CommonOptions::fromArray(['toolbar' => ['bold'], 'placeholder' => 'x']);
        self::assertSame(['bold'], $o->toolbar);
        self::assertSame('x', $o->placeholder);
        self::assertFalse($o->readOnly);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Config;

final class CommonOptions
{
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

    public static function fromArray(array $a): self
    {
        return new self(
            toolbar:     $a['toolbar']     ?? null,
            placeholder: $a['placeholder'] ?? null,
            readOnly:    (bool)($a['readOnly'] ?? false),
            height:      $a['height']      ?? null,
            theme:       $a['theme']       ?? null,
            language:    $a['language']    ?? null,
            plugins:     $a['plugins']     ?? [],
            autofocus:   (bool)($a['autofocus'] ?? false),
            spellcheck:  (bool)($a['spellcheck'] ?? true),
        );
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/CommonOptions.php src/Editor/tests/Config/CommonOptionsTest.php
git commit -m "feat(editor): add CommonOptions DTO"
```

---

## Task 9 — `BridgeCapabilities` DTO + `with()`

**Files:**
- Create: `src/Editor/src/Config/BridgeCapabilities.php`
- Test: `src/Editor/tests/Config/BridgeCapabilitiesTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Config\BridgeCapabilities;

final class BridgeCapabilitiesTest extends TestCase
{
    public function testFields(): void
    {
        $c = new BridgeCapabilities(true, true, false, true, ['html']);
        self::assertTrue($c->supportsToolbar);
        self::assertTrue($c->supportsPlugins);
        self::assertFalse($c->supportsTheme);
        self::assertTrue($c->supportsLanguage);
        self::assertSame(['html'], $c->supportedFormats);
    }

    public function testWithClonesAndOverrides(): void
    {
        $a = new BridgeCapabilities(true, true, true, true, ['html']);
        $b = $a->with(supportsTheme: false, supportedFormats: ['blocks']);
        self::assertTrue($a->supportsTheme);
        self::assertFalse($b->supportsTheme);
        self::assertSame(['blocks'], $b->supportedFormats);
        self::assertTrue($b->supportsToolbar);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Config;

final class BridgeCapabilities
{
    public function __construct(
        public readonly bool  $supportsToolbar,
        public readonly bool  $supportsPlugins,
        public readonly bool  $supportsTheme,
        public readonly bool  $supportsLanguage,
        /** @var list<'html'|'blocks'|'page'> */
        public readonly array $supportedFormats,
    ) {}

    public function with(
        ?bool  $supportsToolbar = null,
        ?bool  $supportsPlugins = null,
        ?bool  $supportsTheme = null,
        ?bool  $supportsLanguage = null,
        ?array $supportedFormats = null,
    ): self {
        return new self(
            $supportsToolbar  ?? $this->supportsToolbar,
            $supportsPlugins  ?? $this->supportsPlugins,
            $supportsTheme    ?? $this->supportsTheme,
            $supportsLanguage ?? $this->supportsLanguage,
            $supportedFormats ?? $this->supportedFormats,
        );
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/BridgeCapabilities.php src/Editor/tests/Config/BridgeCapabilitiesTest.php
git commit -m "feat(editor): add BridgeCapabilities DTO with clone-with"
```

---

## Task 10 — `EditorConfigInterface`

**Files:**
- Create: `src/Editor/src/Config/EditorConfigInterface.php`
- Test: `src/Editor/tests/Config/EditorConfigInterfaceTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;

final class EditorConfigInterfaceTest extends TestCase
{
    public function testContractMethodsExist(): void
    {
        $stub = new class implements EditorConfigInterface {
            public function getBridgeId(): string { return 'fake'; }
            public function getCommon(): CommonOptions { return new CommonOptions(); }
            public function getNativeOverrides(): array { return []; }
            public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
            public function toNative(): array { return []; }
        };
        self::assertSame('fake', $stub->getBridgeId());
        self::assertInstanceOf(CommonOptions::class, $stub->getCommon());
        self::assertInstanceOf(BridgeCapabilities::class, $stub->getCapabilities());
        self::assertSame([], $stub->getNativeOverrides());
        self::assertSame([], $stub->toNative());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Config;

interface EditorConfigInterface
{
    public function getBridgeId(): string;
    public function getCommon(): CommonOptions;
    public function getNativeOverrides(): array;
    public function getCapabilities(): BridgeCapabilities;
    public function toNative(): array;
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/EditorConfigInterface.php src/Editor/tests/Config/EditorConfigInterfaceTest.php
git commit -m "feat(editor): add EditorConfigInterface contract"
```

---

## Task 11 — `AbstractEditorConfig` + merge order

**Files:**
- Create: `src/Editor/src/Config/AbstractEditorConfig.php`
- Test: `src/Editor/tests/Config/AbstractEditorConfigTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;

final class AbstractEditorConfigTest extends TestCase
{
    public function testMergeOrderCommonOwnOverrides(): void
    {
        $c = new class(
            common: new CommonOptions(placeholder: 'common-ph'),
            nativeOverrides: ['placeholder' => 'override-ph', 'extra' => true],
        ) extends AbstractEditorConfig {
            public function getBridgeId(): string { return 'fake'; }
            public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
            protected function translateCommon(CommonOptions $c): array {
                return ['placeholder' => $c->placeholder, 'origin' => 'common'];
            }
            protected function translateOwn(): array { return ['origin' => 'own']; }
        };
        $native = $c->toNative();
        self::assertSame('override-ph', $native['placeholder']);
        self::assertSame('own', $native['origin']);
        self::assertTrue($native['extra']);
    }

    public function testCommonReturnedAsGiven(): void
    {
        $c = new class(common: new CommonOptions(language: 'fr')) extends AbstractEditorConfig {
            public function getBridgeId(): string { return 'fake'; }
            public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
            protected function translateCommon(CommonOptions $c): array { return []; }
        };
        self::assertSame('fr', $c->getCommon()->language);
        self::assertSame([], $c->getNativeOverrides());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Config;

abstract class AbstractEditorConfig implements EditorConfigInterface
{
    public function __construct(
        protected CommonOptions $common = new CommonOptions(),
        protected array $nativeOverrides = [],
    ) {}

    public function getCommon(): CommonOptions { return $this->common; }
    public function getNativeOverrides(): array { return $this->nativeOverrides; }

    public function toNative(): array
    {
        $base = $this->translateCommon($this->common);
        $own  = $this->translateOwn();
        return array_replace_recursive($base, $own, $this->nativeOverrides);
    }

    abstract public function getBridgeId(): string;
    abstract public function getCapabilities(): BridgeCapabilities;
    abstract protected function translateCommon(CommonOptions $c): array;

    protected function translateOwn(): array { return []; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/AbstractEditorConfig.php src/Editor/tests/Config/AbstractEditorConfigTest.php
git commit -m "feat(editor): add AbstractEditorConfig with merge order"
```

---

## Task 12 — Capability guard (`assertCapabilities`)

**Files:**
- Modify: `src/Editor/src/Config/AbstractEditorConfig.php`
- Test: `src/Editor/tests/Config/CapabilityGuardTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config;

use PHPUnit\Framework\TestCase;
use Psr\Log\AbstractLogger;
use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Exception\IncompatibleConfigException;

final class CapabilityGuardTest extends TestCase
{
    public function testWarnsWhenIncompatibleAndNotStrict(): void
    {
        $logger = $this->arrayLogger();
        $c = $this->configWithToolbar(false);
        $c->setLogger($logger);
        $c->toNative();
        self::assertNotEmpty($logger->log);
        self::assertSame('warning', $logger->log[0][0]);
        self::assertStringContainsString('toolbar', $logger->log[0][1]);
    }

    public function testThrowsWhenIncompatibleAndStrict(): void
    {
        $c = $this->configWithToolbar(false);
        $c->setStrict(true);
        $this->expectException(IncompatibleConfigException::class);
        $c->toNative();
    }

    public function testSilentWhenCompatible(): void
    {
        $logger = $this->arrayLogger();
        $c = $this->configWithToolbar(true);
        $c->setLogger($logger);
        $c->toNative();
        self::assertSame([], $logger->log);
    }

    private function arrayLogger(): AbstractLogger
    {
        return new class extends AbstractLogger {
            public array $log = [];
            public function log($level, \Stringable|string $msg, array $ctx = []): void { $this->log[] = [$level, (string)$msg, $ctx]; }
        };
    }

    private function configWithToolbar(bool $supportsToolbar): AbstractEditorConfig
    {
        return new class($supportsToolbar) extends AbstractEditorConfig {
            public function __construct(private bool $supportsToolbar) { parent::__construct(new CommonOptions(toolbar: ['bold'])); }
            public function getBridgeId(): string { return 'fake'; }
            public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities($this->supportsToolbar, true, true, true, ['html']); }
            protected function translateCommon(CommonOptions $c): array { return []; }
        };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — replace `AbstractEditorConfig.php`**

```php
<?php
namespace Symfony\UX\Editor\Config;

use Psr\Log\LoggerAwareInterface;
use Psr\Log\LoggerInterface;
use Psr\Log\NullLogger;
use Symfony\UX\Editor\Exception\IncompatibleConfigException;

abstract class AbstractEditorConfig implements EditorConfigInterface, LoggerAwareInterface
{
    private LoggerInterface $logger;
    private bool $strict = false;

    public function __construct(
        protected CommonOptions $common = new CommonOptions(),
        protected array $nativeOverrides = [],
    ) {
        $this->logger = new NullLogger();
    }

    public function setLogger(LoggerInterface $logger): void { $this->logger = $logger; }
    public function setStrict(bool $strict): void { $this->strict = $strict; }

    public function getCommon(): CommonOptions { return $this->common; }
    public function getNativeOverrides(): array { return $this->nativeOverrides; }

    public function toNative(): array
    {
        $this->assertCapabilities();
        $base = $this->translateCommon($this->common);
        $own  = $this->translateOwn();
        return array_replace_recursive($base, $own, $this->nativeOverrides);
    }

    abstract public function getBridgeId(): string;
    abstract public function getCapabilities(): BridgeCapabilities;
    abstract protected function translateCommon(CommonOptions $c): array;

    protected function translateOwn(): array { return []; }

    protected function assertCapabilities(): void
    {
        $cap   = $this->getCapabilities();
        $issues = [];
        if ($this->common->toolbar !== null && !$cap->supportsToolbar)   $issues[] = 'toolbar';
        if ($this->common->plugins  !== [] && !$cap->supportsPlugins)    $issues[] = 'plugins';
        if ($this->common->theme    !== null && !$cap->supportsTheme)    $issues[] = 'theme';
        if ($this->common->language !== null && !$cap->supportsLanguage) $issues[] = 'language';

        if ($issues === []) {
            return;
        }
        $msg = sprintf('Bridge "%s" does not support common option(s): %s', $this->getBridgeId(), implode(', ', $issues));
        if ($this->strict) {
            throw new IncompatibleConfigException($msg);
        }
        $this->logger->warning($msg);
    }
}
```

- [ ] **Step 4: Run all Config tests — expect 5 pass (3 new + 2 from Task 11)**

```bash
vendor/bin/phpunit tests/Config/
```

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/AbstractEditorConfig.php src/Editor/tests/Config/CapabilityGuardTest.php
git commit -m "feat(editor): add capability guard (warn/strict throw)"
```

---

## Task 13 — `EditorPresetInterface` + `PresetRegistry`

**Files:**
- Create: `src/Editor/src/Config/Preset/EditorPresetInterface.php`
- Create: `src/Editor/src/Config/Preset/PresetRegistry.php`
- Test: `src/Editor/tests/Config/Preset/PresetRegistryTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Config\Preset;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Exception\UnknownBridgeException;

final class PresetRegistryTest extends TestCase
{
    public function testGetByName(): void
    {
        $reg = new PresetRegistry(['blog.standard' => $this->fakePreset('eid')]);
        self::assertSame('eid', $reg->get('blog.standard')->build()->getBridgeId());
    }

    public function testUnknownPresetThrows(): void
    {
        $this->expectException(UnknownBridgeException::class);
        (new PresetRegistry([]))->get('does.not.exist');
    }

    public function testAll(): void
    {
        $reg = new PresetRegistry(['a' => $this->fakePreset('a'), 'b' => $this->fakePreset('b')]);
        self::assertSame(['a', 'b'], array_keys($reg->all()));
    }

    private function fakePreset(string $bridgeId): EditorPresetInterface
    {
        return new class($bridgeId) implements EditorPresetInterface {
            public function __construct(private string $bridgeId) {}
            public function build(): EditorConfigInterface {
                $bid = $this->bridgeId;
                return new class($bid) implements EditorConfigInterface {
                    public function __construct(private string $bid) {}
                    public function getBridgeId(): string { return $this->bid; }
                    public function getCommon(): CommonOptions { return new CommonOptions(); }
                    public function getNativeOverrides(): array { return []; }
                    public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
                    public function toNative(): array { return []; }
                };
            }
        };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`EditorPresetInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Config\Preset;

use Symfony\UX\Editor\Config\EditorConfigInterface;

interface EditorPresetInterface
{
    public function build(): EditorConfigInterface;
}
```

`PresetRegistry.php`:
```php
<?php
namespace Symfony\UX\Editor\Config\Preset;

use Symfony\UX\Editor\Exception\UnknownBridgeException;

final class PresetRegistry
{
    /** @var array<string, EditorPresetInterface> */
    private array $presets = [];

    /** @param iterable<string, EditorPresetInterface> $presets */
    public function __construct(iterable $presets = [])
    {
        foreach ($presets as $name => $preset) {
            $this->presets[$name] = $preset;
        }
    }

    public function get(string $name): EditorPresetInterface
    {
        return $this->presets[$name] ?? throw new UnknownBridgeException(sprintf('Unknown preset "%s". Registered: %s', $name, implode(', ', array_keys($this->presets)) ?: '(none)'));
    }

    /** @return array<string, EditorPresetInterface> */
    public function all(): array { return $this->presets; }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Config/Preset/ src/Editor/tests/Config/Preset/
git commit -m "feat(editor): add preset interface + registry"
```

---

## Task 14 — `ContentConverterInterface` + `ContentConverterRegistry`

**Files:**
- Create: `src/Editor/src/Content/Converter/ContentConverterInterface.php`
- Create: `src/Editor/src/Content/Converter/ContentConverterRegistry.php`
- Test: `src/Editor/tests/Content/Converter/ContentConverterRegistryTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content\Converter;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\Converter\ContentConverterInterface;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Exception\UnsupportedConversionException;

final class ContentConverterRegistryTest extends TestCase
{
    public function testIdentityConversion(): void
    {
        $reg = new ContentConverterRegistry([]);
        $in  = new HtmlContent('<p>x</p>');
        self::assertSame($in, $reg->convert($in, 'ckeditor', 'ckeditor'));
    }

    public function testRegisteredConverterUsed(): void
    {
        $conv = new class implements ContentConverterInterface {
            public function getFrom(): string { return 'a'; }
            public function getTo(): string   { return 'b'; }
            public function convert(EditorContentInterface $c): EditorContentInterface {
                return new HtmlContent('converted:' . $c->getRaw());
            }
        };
        $reg = new ContentConverterRegistry([$conv]);
        self::assertSame('converted:hi', $reg->convert(new HtmlContent('hi'), 'a', 'b')->getRaw());
    }

    public function testUnknownPairThrows(): void
    {
        $this->expectException(UnsupportedConversionException::class);
        (new ContentConverterRegistry([]))->convert(new HtmlContent(''), 'a', 'b');
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`ContentConverterInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Content\Converter;

use Symfony\UX\Editor\Content\EditorContentInterface;

interface ContentConverterInterface
{
    public function getFrom(): string;
    public function getTo(): string;
    public function convert(EditorContentInterface $content): EditorContentInterface;
}
```

`ContentConverterRegistry.php`:
```php
<?php
namespace Symfony\UX\Editor\Content\Converter;

use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Exception\UnsupportedConversionException;

final class ContentConverterRegistry
{
    /** @var array<string, ContentConverterInterface> */
    private array $byPair = [];

    /** @param iterable<ContentConverterInterface> $converters */
    public function __construct(iterable $converters = [])
    {
        foreach ($converters as $c) {
            $this->byPair[$this->key($c->getFrom(), $c->getTo())] = $c;
        }
    }

    public function convert(EditorContentInterface $content, string $from, string $to): EditorContentInterface
    {
        if ($from === $to) { return $content; }
        $c = $this->byPair[$this->key($from, $to)] ?? throw new UnsupportedConversionException(sprintf('No converter registered for "%s" -> "%s"', $from, $to));
        return $c->convert($content);
    }

    private function key(string $from, string $to): string { return $from . '::' . $to; }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/Converter/ src/Editor/tests/Content/Converter/
git commit -m "feat(editor): add content converter interface + registry"
```

---

## Task 15 — Green checkpoint (no code change expected)

- [ ] **Step 1: Run full PHPUnit suite**

```bash
cd src/Editor && vendor/bin/phpunit
```

Expected: all tests from Tasks 2-14 pass. Zero failures/errors/warnings.

- [ ] **Step 2: No commit** (skip if no diff). If anything stale, fix in-place and commit `test(editor): green checkpoint`.

---

## Task 16 — `BridgeInterface`

**Files:**
- Create: `src/Editor/src/Bridge/BridgeInterface.php`
- Create: `src/Editor/src/Form/DataTransformer/EditorContentTransformerInterface.php` (placeholder, will be replaced in Task 19)
- Test: `src/Editor/tests/Bridge/BridgeInterfaceTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\BridgeInterface;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class BridgeInterfaceTest extends TestCase
{
    public function testContract(): void
    {
        $b = new class implements BridgeInterface {
            public function getId(): string { return 'fake'; }
            public function getControllerName(): string { return 'symfony--ux-editor--fake'; }
            public function getDefaultConfig(): EditorConfigInterface {
                return new class implements EditorConfigInterface {
                    public function getBridgeId(): string { return 'fake'; }
                    public function getCommon(): CommonOptions { return new CommonOptions(); }
                    public function getNativeOverrides(): array { return []; }
                    public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
                    public function toNative(): array { return []; }
                };
            }
            public function getCapabilities(): BridgeCapabilities { return $this->getDefaultConfig()->getCapabilities(); }
            public function createTransformer(): EditorContentTransformerInterface { throw new \LogicException('not used'); }
        };
        self::assertSame('fake', $b->getId());
        self::assertSame('symfony--ux-editor--fake', $b->getControllerName());
        self::assertContains('html', $b->getCapabilities()->supportedFormats);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

Placeholder `EditorContentTransformerInterface.php` (will be replaced in Task 19):
```php
<?php
namespace Symfony\UX\Editor\Form\DataTransformer;

interface EditorContentTransformerInterface {}
```

`BridgeInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Bridge;

use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

interface BridgeInterface
{
    public function getId(): string;
    public function getControllerName(): string;
    public function getDefaultConfig(): EditorConfigInterface;
    public function getCapabilities(): BridgeCapabilities;
    public function createTransformer(): EditorContentTransformerInterface;
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/BridgeInterface.php src/Editor/src/Form/DataTransformer/EditorContentTransformerInterface.php src/Editor/tests/Bridge/BridgeInterfaceTest.php
git commit -m "feat(editor): add BridgeInterface + transformer placeholder"
```

---

## Task 17 — `AbstractBridge`

**Files:**
- Create: `src/Editor/src/Bridge/AbstractBridge.php`
- Test: `src/Editor/tests/Bridge/AbstractBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class AbstractBridgeTest extends TestCase
{
    public function testDefaultControllerNameDerivedFromId(): void
    {
        $b = new class extends AbstractBridge {
            public function getId(): string { return 'fakebridge'; }
            public function getDefaultConfig(): EditorConfigInterface {
                return new class implements EditorConfigInterface {
                    public function getBridgeId(): string { return 'fakebridge'; }
                    public function getCommon(): CommonOptions { return new CommonOptions(); }
                    public function getNativeOverrides(): array { return []; }
                    public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
                    public function toNative(): array { return []; }
                };
            }
            public function getCapabilities(): BridgeCapabilities { return $this->getDefaultConfig()->getCapabilities(); }
            public function createTransformer(): EditorContentTransformerInterface { throw new \LogicException('not used'); }
        };
        self::assertSame('symfony--ux-editor--fakebridge', $b->getControllerName());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge;

abstract class AbstractBridge implements BridgeInterface
{
    public function getControllerName(): string
    {
        return 'symfony--ux-editor--' . $this->getId();
    }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/AbstractBridge.php src/Editor/tests/Bridge/AbstractBridgeTest.php
git commit -m "feat(editor): add AbstractBridge with default controller name"
```

---

## Task 18 — `BridgeRegistry`

**Files:**
- Create: `src/Editor/src/Bridge/BridgeRegistry.php`
- Test: `src/Editor/tests/Bridge/BridgeRegistryTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\BridgeInterface;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Exception\UnknownBridgeException;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class BridgeRegistryTest extends TestCase
{
    public function testGetAndAll(): void
    {
        $a = $this->fakeBridge('a');
        $b = $this->fakeBridge('b');
        $reg = new BridgeRegistry([$a, $b]);
        self::assertSame($a, $reg->get('a'));
        self::assertSame($b, $reg->get('b'));
        self::assertSame(['a', 'b'], array_keys($reg->all()));
    }

    public function testUnknownThrows(): void
    {
        $this->expectException(UnknownBridgeException::class);
        (new BridgeRegistry([]))->get('missing');
    }

    public function testDuplicateIdThrows(): void
    {
        $this->expectException(\LogicException::class);
        new BridgeRegistry([$this->fakeBridge('a'), $this->fakeBridge('a')]);
    }

    private function fakeBridge(string $id): BridgeInterface
    {
        return new class($id) implements BridgeInterface {
            public function __construct(private string $id) {}
            public function getId(): string { return $this->id; }
            public function getControllerName(): string { return 'symfony--ux-editor--' . $this->id; }
            public function getDefaultConfig(): EditorConfigInterface { throw new \LogicException('not used'); }
            public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
            public function createTransformer(): EditorContentTransformerInterface { throw new \LogicException('not used'); }
        };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge;

use Symfony\UX\Editor\Exception\UnknownBridgeException;

final class BridgeRegistry
{
    /** @var array<string, BridgeInterface> */
    private array $bridges = [];

    /** @param iterable<BridgeInterface> $bridges */
    public function __construct(iterable $bridges = [])
    {
        foreach ($bridges as $b) {
            if (isset($this->bridges[$b->getId()])) {
                throw new \LogicException(sprintf('Duplicate bridge id "%s"', $b->getId()));
            }
            $this->bridges[$b->getId()] = $b;
        }
    }

    public function get(string $id): BridgeInterface
    {
        return $this->bridges[$id] ?? throw new UnknownBridgeException(sprintf('Unknown bridge "%s". Registered: %s', $id, implode(', ', array_keys($this->bridges)) ?: '(none)'));
    }

    /** @return array<string, BridgeInterface> */
    public function all(): array { return $this->bridges; }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/BridgeRegistry.php src/Editor/tests/Bridge/BridgeRegistryTest.php
git commit -m "feat(editor): add BridgeRegistry"
```

---

## Task 19 — `EditorContentTransformerInterface` + `StorageShape`

**Files:**
- Modify: `src/Editor/src/Form/DataTransformer/EditorContentTransformerInterface.php` (replace placeholder)
- Create: `src/Editor/src/Form/DataTransformer/StorageShape.php`
- Test: `src/Editor/tests/Form/DataTransformer/EditorContentTransformerInterfaceTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Form\DataTransformer;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class EditorContentTransformerInterfaceTest extends TestCase
{
    public function testEnumCases(): void
    {
        self::assertSame('scalar', StorageShape::Scalar->value);
        self::assertSame('json',   StorageShape::Json->value);
        self::assertSame('split',  StorageShape::Split->value);
    }

    public function testContract(): void
    {
        $t = new class implements EditorContentTransformerInterface {
            public function getBridgeId(): string { return 'fake'; }
            public function getContentClass(): string { return HtmlContent::class; }
            public function getStorageShape(): StorageShape { return StorageShape::Scalar; }
            public function transform(?EditorContentInterface $c): mixed { return $c?->getRaw(); }
            public function reverseTransform(mixed $v): ?EditorContentInterface { return $v === null ? null : new HtmlContent((string)$v); }
        };
        self::assertSame('fake', $t->getBridgeId());
        self::assertSame(HtmlContent::class, $t->getContentClass());
        self::assertSame(StorageShape::Scalar, $t->getStorageShape());
        self::assertSame('hi', $t->transform(new HtmlContent('hi')));
        self::assertNull($t->transform(null));
        self::assertInstanceOf(HtmlContent::class, $t->reverseTransform('hi'));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`StorageShape.php`:
```php
<?php
namespace Symfony\UX\Editor\Form\DataTransformer;

enum StorageShape: string
{
    case Scalar = 'scalar';
    case Json   = 'json';
    case Split  = 'split';
}
```

Replace `EditorContentTransformerInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Form\DataTransformer;

use Symfony\UX\Editor\Content\EditorContentInterface;

interface EditorContentTransformerInterface
{
    public function getBridgeId(): string;
    public function getContentClass(): string;
    public function getStorageShape(): StorageShape;
    public function transform(?EditorContentInterface $content): mixed;
    public function reverseTransform(mixed $stored): ?EditorContentInterface;
}
```

- [ ] **Step 4: Run — expect 2 pass + Tasks 16/17/18 still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Form/DataTransformer/ src/Editor/tests/Form/DataTransformer/
git commit -m "feat(editor): add EditorContentTransformerInterface + StorageShape enum"
```

---

## Task 20 — `EditorType` (option resolver + transformer wiring + sanitize)

**Files:**
- Create: `src/Editor/src/Form/EditorType.php`
- Create: `src/Editor/src/Form/DataTransformer/TransformerAdapter.php` (private adapter to Symfony `DataTransformerInterface`)
- Test: `src/Editor/tests/Form/EditorTypeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Form;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;
use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;
use Symfony\UX\Editor\Form\EditorType;

final class EditorTypeTest extends TestCase
{
    public function testTypedConfigModeRoundTrips(): void
    {
        $form = $this->factory()->createBuilder()
            ->add('body', EditorType::class, ['config' => new EditorTypeTestFakeConfig()])
            ->getForm();
        $form->submit(['body' => '<p>hi</p>']);
        self::assertTrue($form->isSynchronized());
        $data = $form->get('body')->getData();
        self::assertInstanceOf(HtmlContent::class, $data);
        self::assertSame('<p>hi</p>', $data->html);
    }

    public function testBridgePlusArrayMode(): void
    {
        $form = $this->factory()->createBuilder()
            ->add('body', EditorType::class, ['bridge' => 'fake', 'common' => ['placeholder' => 'x']])
            ->getForm();
        $form->submit(['body' => 'plain']);
        self::assertSame('plain', $form->get('body')->getData()->html);
    }

    public function testSanitizeOnSubmitWhenEnabled(): void
    {
        $sanitizer = new HtmlSanitizer((new HtmlSanitizerConfig())->allowSafeElements());
        $form = $this->factory($sanitizer)->createBuilder()
            ->add('body', EditorType::class, ['config' => new EditorTypeTestFakeConfig(), 'sanitize' => true])
            ->getForm();
        $form->submit(['body' => '<script>x</script><p>ok</p>']);
        self::assertStringNotContainsString('<script>', $form->get('body')->getData()->html);
    }

    public function testSanitizeOffWhenDisabled(): void
    {
        $sanitizer = new HtmlSanitizer((new HtmlSanitizerConfig())->allowSafeElements());
        $form = $this->factory($sanitizer)->createBuilder()
            ->add('body', EditorType::class, ['config' => new EditorTypeTestFakeConfig(), 'sanitize' => false])
            ->getForm();
        $form->submit(['body' => '<script>x</script>']);
        self::assertSame('<script>x</script>', $form->get('body')->getData()->html);
    }

    private function factory(?HtmlSanitizer $sanitizer = null): \Symfony\Component\Form\FormFactoryInterface
    {
        $bridges = new BridgeRegistry([new EditorTypeTestFakeBridge()]);
        $presets = new PresetRegistry([]);
        return Forms::createFormFactoryBuilder()
            ->addType(new EditorType($bridges, $presets, $sanitizer))
            ->getFormFactory();
    }
}

final class EditorTypeTestFakeConfig implements EditorConfigInterface {
    public function getBridgeId(): string { return 'fake'; }
    public function getCommon(): CommonOptions { return new CommonOptions(); }
    public function getNativeOverrides(): array { return []; }
    public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(true, true, true, true, ['html']); }
    public function toNative(): array { return []; }
}

final class EditorTypeTestFakeBridge extends AbstractBridge {
    public function getId(): string { return 'fake'; }
    public function getDefaultConfig(): EditorConfigInterface { return new EditorTypeTestFakeConfig(); }
    public function getCapabilities(): BridgeCapabilities { return (new EditorTypeTestFakeConfig())->getCapabilities(); }
    public function createTransformer(): EditorContentTransformerInterface {
        return new class implements EditorContentTransformerInterface {
            public function getBridgeId(): string { return 'fake'; }
            public function getContentClass(): string { return HtmlContent::class; }
            public function getStorageShape(): StorageShape { return StorageShape::Scalar; }
            public function transform(?EditorContentInterface $c): mixed { return $c?->getRaw(); }
            public function reverseTransform(mixed $v): ?EditorContentInterface { return $v === null ? null : new HtmlContent((string)$v); }
        };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`src/Form/DataTransformer/TransformerAdapter.php`:
```php
<?php
namespace Symfony\UX\Editor\Form\DataTransformer;

use Symfony\Component\Form\DataTransformerInterface;
use Symfony\Component\Form\Exception\TransformationFailedException;
use Symfony\UX\Editor\Content\EditorContentInterface;

/** @internal */
final class TransformerAdapter implements DataTransformerInterface
{
    public function __construct(private readonly EditorContentTransformerInterface $inner) {}

    public function transform(mixed $value): mixed
    {
        if ($value === null) { return ''; }
        if (!$value instanceof EditorContentInterface) {
            throw new TransformationFailedException('Expected EditorContentInterface, got '.get_debug_type($value));
        }
        $raw = $this->inner->transform($value);
        return \is_string($raw) ? $raw : json_encode($raw, \JSON_THROW_ON_ERROR);
    }

    public function reverseTransform(mixed $value): mixed
    {
        if ($value === null || $value === '') { return null; }
        if ($this->inner->getStorageShape() !== StorageShape::Scalar && \is_string($value)) {
            try {
                $value = json_decode($value, true, 512, \JSON_THROW_ON_ERROR);
            } catch (\JsonException $e) {
                throw new TransformationFailedException('Invalid JSON for editor content', 0, $e);
            }
        }
        return $this->inner->reverseTransform($value);
    }
}
```

`src/Form/EditorType.php`:
```php
<?php
namespace Symfony\UX\Editor\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Exception\BridgeConfigMismatchException;
use Symfony\UX\Editor\Form\DataTransformer\TransformerAdapter;

class EditorType extends AbstractType
{
    public function __construct(
        private readonly BridgeRegistry $bridges,
        private readonly PresetRegistry $presets,
        private readonly ?HtmlSanitizerInterface $sanitizer = null,
    ) {}

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'config' => null,
            'preset' => null,
            'bridge' => null,
            'common' => null,
            'native' => null,
            'sanitize'           => true,
            'strictCapabilities' => false,
            'upload_url'         => null,
            'compound'           => false,
        ]);
        $resolver->setAllowedTypes('config', ['null', EditorConfigInterface::class]);
        $resolver->setAllowedTypes('preset', ['null', 'string']);
        $resolver->setAllowedTypes('bridge', ['null', 'string']);
        $resolver->setAllowedTypes('common', ['null', 'array']);
        $resolver->setAllowedTypes('native', ['null', 'array']);
        $resolver->setAllowedTypes('sanitize', 'bool');
        $resolver->setAllowedTypes('strictCapabilities', 'bool');
        $resolver->setAllowedTypes('upload_url', ['null', 'string']);
    }

    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $config = $this->resolveConfig($options);
        $bridge = $this->bridges->get($config->getBridgeId());

        if ($config instanceof AbstractEditorConfig && $options['strictCapabilities']) {
            $config->setStrict(true);
        }

        $builder->addModelTransformer(new TransformerAdapter($bridge->createTransformer()));
        $builder->setAttribute('ux_editor_config', $config);
        $builder->setAttribute('ux_editor_bridge', $bridge);

        if ($options['sanitize'] && $this->sanitizer !== null) {
            $sanitizer = $this->sanitizer;
            $builder->addEventListener(FormEvents::SUBMIT, function (FormEvent $event) use ($sanitizer): void {
                $data = $event->getData();
                if ($data instanceof HtmlContent) {
                    $event->setData(new HtmlContent($sanitizer->sanitize($data->html), $data->getMetadata()));
                }
            });
        }
    }

    public function getParent(): string { return TextareaType::class; }
    public function getBlockPrefix(): string { return 'ux_editor'; }

    private function resolveConfig(array $options): EditorConfigInterface
    {
        if ($options['config'] instanceof EditorConfigInterface) {
            return $options['config'];
        }
        if (\is_string($options['preset'])) {
            return $this->presets->get($options['preset'])->build();
        }
        if (\is_string($options['bridge'])) {
            $bridge = $this->bridges->get($options['bridge']);
            $config = $bridge->getDefaultConfig();
            if (!$config instanceof AbstractEditorConfig) {
                throw new BridgeConfigMismatchException(sprintf('Bridge "%s" default config must extend AbstractEditorConfig to accept array shorthand.', $options['bridge']));
            }
            $common = $options['common'] !== null ? CommonOptions::fromArray($options['common']) : $config->getCommon();
            $native = $options['native'] ?? $config->getNativeOverrides();
            $r = new \ReflectionObject($config);
            $r->getProperty('common')->setValue($config, $common);
            $r->getProperty('nativeOverrides')->setValue($config, $native);
            return $config;
        }
        throw new \InvalidArgumentException('EditorType requires one of: "config", "preset", or "bridge" option.');
    }
}
```

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Form/EditorType.php src/Editor/src/Form/DataTransformer/TransformerAdapter.php src/Editor/tests/Form/EditorTypeTest.php
git commit -m "feat(editor): add EditorType with three input modes + sanitize wiring"
```

---

## Task 21 — `EditorType::buildView` (Stimulus attrs)

**Files:**
- Modify: `src/Editor/src/Form/EditorType.php`
- Test: `src/Editor/tests/Form/EditorTypeViewTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Form;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Form\EditorType;

final class EditorTypeViewTest extends TestCase
{
    public function testStimulusAttrsOnView(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new EditorTypeTestFakeBridge()]), new PresetRegistry([])))
            ->getFormFactory();
        $form = $factory->createBuilder()->add('body', EditorType::class, ['config' => new EditorTypeTestFakeConfig()])->getForm();
        $view = $form->get('body')->createView();
        $attrs = $view->vars['attr'];
        self::assertStringContainsString('symfony--ux-editor--fake', $attrs['data-controller']);
        self::assertSame('html', $attrs['data-symfony--ux-editor--fake-format-value']);
        self::assertJson($attrs['data-symfony--ux-editor--fake-config-value']);
    }
}
```

(Reuses fixtures `EditorTypeTestFakeConfig` and `EditorTypeTestFakeBridge` from Task 20.)

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — append to `EditorType`**

Add method to `EditorType` class:
```php
public function buildView(\Symfony\Component\Form\FormView $view, \Symfony\Component\Form\FormInterface $form, array $options): void
{
    parent::buildView($view, $form, $options);

    /** @var \Symfony\UX\Editor\Bridge\BridgeInterface $bridge */
    $bridge = $form->getConfig()->getAttribute('ux_editor_bridge');
    /** @var \Symfony\UX\Editor\Config\EditorConfigInterface $config */
    $config = $form->getConfig()->getAttribute('ux_editor_config');

    $controller = $bridge->getControllerName();
    $format     = $config->getCapabilities()->supportedFormats[0] ?? 'html';

    $attr = $view->vars['attr'] ?? [];
    $attr['data-controller'] = trim(($attr['data-controller'] ?? '') . ' ' . $controller);
    $attr["data-{$controller}-config-value"]    = json_encode($config->toNative(), \JSON_THROW_ON_ERROR);
    $attr["data-{$controller}-format-value"]    = $format;
    $attr["data-{$controller}-bridge-id-value"] = $config->getBridgeId();
    if ($options['upload_url'] !== null) {
        $attr["data-{$controller}-upload-url-value"] = $options['upload_url'];
    }
    $view->vars['attr'] = $attr;
}
```

- [ ] **Step 4: Run — expect 1 pass + Task 20 tests still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Form/EditorType.php src/Editor/tests/Form/EditorTypeViewTest.php
git commit -m "feat(editor): add Stimulus attrs in EditorType::buildView"
```

---

## Task 22 — `strictCapabilities` end-to-end

**Files:**
- Test: `src/Editor/tests/Form/EditorTypeStrictTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Form;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Form\Forms;
use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\AbstractEditorConfig;
use Symfony\UX\Editor\Config\BridgeCapabilities;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Exception\IncompatibleConfigException;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;
use Symfony\UX\Editor\Form\EditorType;

final class EditorTypeStrictTest extends TestCase
{
    public function testStrictCapabilitiesThrowsOnIncompatibleToolbar(): void
    {
        $factory = Forms::createFormFactoryBuilder()
            ->addType(new EditorType(new BridgeRegistry([new NoToolbarBridge()]), new PresetRegistry([])))
            ->getFormFactory();
        $form = $factory->createBuilder()->add('body', EditorType::class, [
            'config' => new NoToolbarConfig(new CommonOptions(toolbar: ['bold'])),
            'strictCapabilities' => true,
        ])->getForm();
        $this->expectException(IncompatibleConfigException::class);
        $form->get('body')->createView();   // toNative() in buildView triggers assertCapabilities
    }
}

final class NoToolbarConfig extends AbstractEditorConfig {
    public function getBridgeId(): string { return 'notb'; }
    public function getCapabilities(): BridgeCapabilities { return new BridgeCapabilities(false, true, true, true, ['html']); }
    protected function translateCommon(CommonOptions $c): array { return []; }
}

final class NoToolbarBridge extends AbstractBridge {
    public function getId(): string { return 'notb'; }
    public function getDefaultConfig(): EditorConfigInterface { return new NoToolbarConfig(); }
    public function getCapabilities(): BridgeCapabilities { return (new NoToolbarConfig())->getCapabilities(); }
    public function createTransformer(): EditorContentTransformerInterface {
        return new class implements EditorContentTransformerInterface {
            public function getBridgeId(): string { return 'notb'; }
            public function getContentClass(): string { return HtmlContent::class; }
            public function getStorageShape(): StorageShape { return StorageShape::Scalar; }
            public function transform(?EditorContentInterface $c): mixed { return $c?->getRaw(); }
            public function reverseTransform(mixed $v): ?EditorContentInterface { return $v === null ? null : new HtmlContent((string)$v); }
        };
    }
}
```

- [ ] **Step 2: Run — expect 1 pass** (behavior wired in Task 12 + Task 20; this verifies the integration end-to-end).

- [ ] **Step 3: Implementation (only if Step 2 fails)**

If failing: confirm `setStrict(true)` runs before `toNative()` in `buildView`. Fix by moving `setStrict` invocation to top of `buildView` before any access to `toNative()`. No expected change.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/tests/Form/EditorTypeStrictTest.php
git commit -m "test(editor): strictCapabilities end-to-end"
```

---

## Task 23 — `EditorContentHtmlType` (Doctrine)

**Files:**
- Create: `src/Editor/src/Doctrine/EditorContentHtmlType.php`
- Test: `src/Editor/tests/Doctrine/EditorContentHtmlTypeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Doctrine;

use Doctrine\DBAL\Platforms\SqlitePlatform;
use Doctrine\DBAL\Types\Type;
use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\HtmlContent;
use Symfony\UX\Editor\Doctrine\EditorContentHtmlType;

final class EditorContentHtmlTypeTest extends TestCase
{
    protected function setUp(): void
    {
        if (!Type::hasType('editor_html')) {
            Type::addType('editor_html', EditorContentHtmlType::class);
        }
    }

    public function testConvertToDatabaseValue(): void
    {
        $t = Type::getType('editor_html');
        $p = new SqlitePlatform();
        self::assertSame('<p>hi</p>', $t->convertToDatabaseValue(new HtmlContent('<p>hi</p>'), $p));
        self::assertNull($t->convertToDatabaseValue(null, $p));
    }

    public function testConvertToPHPValue(): void
    {
        $t = Type::getType('editor_html');
        $p = new SqlitePlatform();
        $php = $t->convertToPHPValue('<p>hi</p>', $p);
        self::assertInstanceOf(HtmlContent::class, $php);
        self::assertSame('<p>hi</p>', $php->html);
        self::assertNull($t->convertToPHPValue(null, $p));
    }

    public function testTypeName(): void
    {
        self::assertSame('editor_html', Type::getType('editor_html')->getName());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Doctrine;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\TextType;
use Symfony\UX\Editor\Content\HtmlContent;

final class EditorContentHtmlType extends TextType
{
    public const NAME = 'editor_html';

    public function getName(): string { return self::NAME; }

    public function convertToDatabaseValue(mixed $value, AbstractPlatform $platform): ?string
    {
        if ($value === null) { return null; }
        if (!$value instanceof HtmlContent) {
            throw new \InvalidArgumentException('Expected HtmlContent, got '.get_debug_type($value));
        }
        return $value->html;
    }

    public function convertToPHPValue(mixed $value, AbstractPlatform $platform): ?HtmlContent
    {
        return $value === null ? null : new HtmlContent((string)$value);
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform): bool { return true; }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Doctrine/EditorContentHtmlType.php src/Editor/tests/Doctrine/EditorContentHtmlTypeTest.php
git commit -m "feat(editor): add EditorContentHtmlType Doctrine Type"
```

---

## Task 24 — `EditorContentBlocksType` (Doctrine)

**Files:**
- Create: `src/Editor/src/Doctrine/EditorContentBlocksType.php`
- Test: `src/Editor/tests/Doctrine/EditorContentBlocksTypeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Doctrine;

use Doctrine\DBAL\Platforms\SqlitePlatform;
use Doctrine\DBAL\Types\Type;
use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Doctrine\EditorContentBlocksType;
use Symfony\UX\Editor\Exception\ContentSchemaException;

final class EditorContentBlocksTypeTest extends TestCase
{
    protected function setUp(): void
    {
        if (!Type::hasType('editor_blocks')) {
            Type::addType('editor_blocks', EditorContentBlocksType::class);
        }
    }

    public function testRoundTrip(): void
    {
        $t = Type::getType('editor_blocks');
        $p = new SqlitePlatform();
        $bc = new BlockContent([['type' => 'p', 'data' => ['text' => 'x']]], '2.0');
        $json = $t->convertToDatabaseValue($bc, $p);
        self::assertJson($json);
        $back = $t->convertToPHPValue($json, $p);
        self::assertInstanceOf(BlockContent::class, $back);
        self::assertSame('2.0', $back->schemaVersion);
        self::assertSame([['type' => 'p', 'data' => ['text' => 'x']]], $back->blocks);
    }

    public function testNullRoundTrip(): void
    {
        $t = Type::getType('editor_blocks');
        $p = new SqlitePlatform();
        self::assertNull($t->convertToDatabaseValue(null, $p));
        self::assertNull($t->convertToPHPValue(null, $p));
    }

    public function testMalformedJsonThrows(): void
    {
        $this->expectException(ContentSchemaException::class);
        Type::getType('editor_blocks')->convertToPHPValue('{not json', new SqlitePlatform());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Doctrine;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\JsonType;
use Symfony\UX\Editor\Content\BlockContent;
use Symfony\UX\Editor\Exception\ContentSchemaException;

final class EditorContentBlocksType extends JsonType
{
    public const NAME = 'editor_blocks';

    public function getName(): string { return self::NAME; }

    public function convertToDatabaseValue(mixed $value, AbstractPlatform $platform): ?string
    {
        if ($value === null) { return null; }
        if (!$value instanceof BlockContent) {
            throw new \InvalidArgumentException('Expected BlockContent, got '.get_debug_type($value));
        }
        try {
            return json_encode([
                'version'  => $value->schemaVersion,
                'blocks'   => $value->blocks,
                'metadata' => $value->getMetadata(),
            ], \JSON_THROW_ON_ERROR);
        } catch (\JsonException $e) {
            throw new ContentSchemaException('Could not encode BlockContent', 0, $e);
        }
    }

    public function convertToPHPValue(mixed $value, AbstractPlatform $platform): ?BlockContent
    {
        if ($value === null) { return null; }
        try {
            $arr = json_decode((string)$value, true, 512, \JSON_THROW_ON_ERROR);
        } catch (\JsonException $e) {
            throw new ContentSchemaException('Malformed BlockContent JSON', 0, $e);
        }
        return new BlockContent(
            blocks: $arr['blocks'] ?? [],
            schemaVersion: (string)($arr['version'] ?? '1.0'),
            metadata: $arr['metadata'] ?? [],
        );
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform): bool { return true; }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Doctrine/EditorContentBlocksType.php src/Editor/tests/Doctrine/EditorContentBlocksTypeTest.php
git commit -m "feat(editor): add EditorContentBlocksType Doctrine Type"
```

---

## Task 25 — `EditorContentPageType` (Doctrine)

**Files:**
- Create: `src/Editor/src/Doctrine/EditorContentPageType.php`
- Test: `src/Editor/tests/Doctrine/EditorContentPageTypeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Doctrine;

use Doctrine\DBAL\Platforms\SqlitePlatform;
use Doctrine\DBAL\Types\Type;
use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Doctrine\EditorContentPageType;
use Symfony\UX\Editor\Exception\ContentSchemaException;

final class EditorContentPageTypeTest extends TestCase
{
    protected function setUp(): void
    {
        if (!Type::hasType('editor_page')) {
            Type::addType('editor_page', EditorContentPageType::class);
        }
    }

    public function testRoundTrip(): void
    {
        $t = Type::getType('editor_page');
        $p = new SqlitePlatform();
        $pc = new PageContent('<h1>x</h1>', 'h1{}', [['type' => 'image', 'url' => '/x.png']], [['type' => 'h1']]);
        $back = $t->convertToPHPValue($t->convertToDatabaseValue($pc, $p), $p);
        self::assertInstanceOf(PageContent::class, $back);
        self::assertSame('<h1>x</h1>', $back->html);
        self::assertSame('h1{}', $back->css);
        self::assertCount(1, $back->assets);
        self::assertCount(1, $back->components);
    }

    public function testMalformedThrows(): void
    {
        $this->expectException(ContentSchemaException::class);
        Type::getType('editor_page')->convertToPHPValue('not json', new SqlitePlatform());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Doctrine;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\JsonType;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Exception\ContentSchemaException;

final class EditorContentPageType extends JsonType
{
    public const NAME = 'editor_page';

    public function getName(): string { return self::NAME; }

    public function convertToDatabaseValue(mixed $value, AbstractPlatform $platform): ?string
    {
        if ($value === null) { return null; }
        if (!$value instanceof PageContent) {
            throw new \InvalidArgumentException('Expected PageContent, got '.get_debug_type($value));
        }
        try {
            return json_encode([
                'html'       => $value->html,
                'css'        => $value->css,
                'assets'     => $value->assets,
                'components' => $value->components,
                'metadata'   => $value->getMetadata(),
            ], \JSON_THROW_ON_ERROR);
        } catch (\JsonException $e) {
            throw new ContentSchemaException('Could not encode PageContent', 0, $e);
        }
    }

    public function convertToPHPValue(mixed $value, AbstractPlatform $platform): ?PageContent
    {
        if ($value === null) { return null; }
        try {
            $arr = json_decode((string)$value, true, 512, \JSON_THROW_ON_ERROR);
        } catch (\JsonException $e) {
            throw new ContentSchemaException('Malformed PageContent JSON', 0, $e);
        }
        return new PageContent(
            html: (string)($arr['html'] ?? ''),
            css:  (string)($arr['css']  ?? ''),
            assets: $arr['assets'] ?? [],
            components: $arr['components'] ?? [],
            metadata: $arr['metadata'] ?? [],
        );
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform): bool { return true; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Doctrine/EditorContentPageType.php src/Editor/tests/Doctrine/EditorContentPageTypeTest.php
git commit -m "feat(editor): add EditorContentPageType Doctrine Type"
```

---

## Task 26 — `SignedUploadUrlGenerator`

**Files:**
- Create: `src/Editor/src/Upload/SignedUploadUrlGenerator.php`
- Test: `src/Editor/tests/Upload/SignedUploadUrlGeneratorTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Upload;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Exception\Upload\InvalidSignatureException;
use Symfony\UX\Editor\Upload\SignedUploadUrlGenerator;

final class SignedUploadUrlGeneratorTest extends TestCase
{
    public function testHappyRoundTrip(): void
    {
        $g = new SignedUploadUrlGenerator(secret: 's3cret', ttlSeconds: 3600);
        $token = $g->sign(field: 'body', profile: 'default');
        $g->verify($token, field: 'body', profile: 'default');
        $this->addToAssertionCount(1);
    }

    public function testTamperedTokenThrows(): void
    {
        $g = new SignedUploadUrlGenerator('s3cret', 3600);
        $token = $g->sign('body', 'default');
        $this->expectException(InvalidSignatureException::class);
        $g->verify($token.'x', 'body', 'default');
    }

    public function testFieldMismatchThrows(): void
    {
        $g = new SignedUploadUrlGenerator('s3cret', 3600);
        $token = $g->sign('body', 'default');
        $this->expectException(InvalidSignatureException::class);
        $g->verify($token, 'other-field', 'default');
    }

    public function testExpiredTokenThrows(): void
    {
        $g = new SignedUploadUrlGenerator('s3cret', ttlSeconds: -1);
        $token = $g->sign('body', 'default');
        $this->expectException(InvalidSignatureException::class);
        $g->verify($token, 'body', 'default');
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Upload;

use Symfony\UX\Editor\Exception\Upload\InvalidSignatureException;

final class SignedUploadUrlGenerator
{
    public function __construct(
        #[\SensitiveParameter] private readonly string $secret,
        private readonly int $ttlSeconds = 3600,
    ) {}

    public function sign(string $field, string $profile, ?int $now = null): string
    {
        $now ??= time();
        $exp = $now + $this->ttlSeconds;
        $payload = sprintf('%s|%s|%d', $field, $profile, $exp);
        $sig = hash_hmac('sha256', $payload, $this->secret);
        return base64_encode($payload.'|'.$sig);
    }

    public function verify(string $token, string $field, string $profile, ?int $now = null): void
    {
        $now ??= time();
        $decoded = base64_decode($token, true);
        if ($decoded === false) {
            throw new InvalidSignatureException('Invalid token encoding');
        }
        $parts = explode('|', $decoded);
        if (\count($parts) !== 4) {
            throw new InvalidSignatureException('Invalid token shape');
        }
        [$f, $p, $exp, $sig] = $parts;
        $payload = sprintf('%s|%s|%s', $f, $p, $exp);
        $expected = hash_hmac('sha256', $payload, $this->secret);
        if (!hash_equals($expected, $sig)) {
            throw new InvalidSignatureException('Signature mismatch');
        }
        if ($f !== $field) {
            throw new InvalidSignatureException('Field mismatch');
        }
        if ($p !== $profile) {
            throw new InvalidSignatureException('Profile mismatch');
        }
        if ((int)$exp <= $now) {
            throw new InvalidSignatureException('Token expired');
        }
    }
}
```

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Upload/SignedUploadUrlGenerator.php src/Editor/tests/Upload/SignedUploadUrlGeneratorTest.php
git commit -m "feat(editor): add SignedUploadUrlGenerator (HMAC, expiry)"
```

---

## Task 27 — `EditorUploadHandlerInterface` + `DefaultLocalUploadHandler`

**Files:**
- Create: `src/Editor/src/Upload/EditorUploadHandlerInterface.php`
- Create: `src/Editor/src/Upload/DefaultLocalUploadHandler.php`
- Test: `src/Editor/tests/Upload/DefaultLocalUploadHandlerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Upload;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\UX\Editor\Exception\Upload\UnsupportedFileException;
use Symfony\UX\Editor\Upload\DefaultLocalUploadHandler;

final class DefaultLocalUploadHandlerTest extends TestCase
{
    private string $tmp;

    protected function setUp(): void
    {
        $this->tmp = sys_get_temp_dir().'/ux-editor-test-'.uniqid();
        mkdir($this->tmp);
    }
    protected function tearDown(): void
    {
        foreach (glob($this->tmp.'/*') ?: [] as $f) { @unlink($f); }
        @rmdir($this->tmp);
    }

    public function testHappyUpload(): void
    {
        $h = new DefaultLocalUploadHandler($this->tmp, '/uploads', ['image/png'], 2_000_000);
        $file = $this->makeUploadedFile('img.png', 'image/png', 100);
        $r = $h->handle($file, ['profile' => 'default', 'field' => 'body']);
        self::assertStringStartsWith('/uploads/', $r['url']);
        self::assertSame(100, $r['size']);
        self::assertFileExists($this->tmp.'/'.basename($r['url']));
    }

    public function testRejectsUnsupportedMime(): void
    {
        $h = new DefaultLocalUploadHandler($this->tmp, '/uploads', ['image/png'], 2_000_000);
        $file = $this->makeUploadedFile('evil.exe', 'application/x-msdownload', 10);
        $this->expectException(UnsupportedFileException::class);
        $h->handle($file, []);
    }

    public function testRejectsOversize(): void
    {
        $h = new DefaultLocalUploadHandler($this->tmp, '/uploads', ['image/png'], 50);
        $file = $this->makeUploadedFile('big.png', 'image/png', 1000);
        $this->expectException(UnsupportedFileException::class);
        $h->handle($file, []);
    }

    private function makeUploadedFile(string $name, string $mime, int $size): UploadedFile
    {
        $tmp = tempnam(sys_get_temp_dir(), 'ux');
        file_put_contents($tmp, str_repeat('x', $size));
        return new UploadedFile($tmp, $name, $mime, null, true);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`EditorUploadHandlerInterface.php`:
```php
<?php
namespace Symfony\UX\Editor\Upload;

use Symfony\Component\HttpFoundation\File\UploadedFile;

interface EditorUploadHandlerInterface
{
    /**
     * @param array<string,mixed> $context
     * @return array{url:string, size:int, type?:string, width?:int, height?:int}
     */
    public function handle(UploadedFile $file, array $context = []): array;
}
```

`DefaultLocalUploadHandler.php`:
```php
<?php
namespace Symfony\UX\Editor\Upload;

use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\UX\Editor\Exception\Upload\UnsupportedFileException;
use Symfony\UX\Editor\Exception\Upload\UploadHandlerException;

final class DefaultLocalUploadHandler implements EditorUploadHandlerInterface
{
    public function __construct(
        private readonly string $targetDir,
        private readonly string $publicUrlPrefix = '/uploads',
        /** @var list<string> */
        private readonly array $allowedMimes = ['image/png', 'image/jpeg', 'image/gif', 'image/webp'],
        private readonly int $maxBytes = 5_000_000,
    ) {}

    public function handle(UploadedFile $file, array $context = []): array
    {
        if (!\in_array($file->getMimeType() ?? $file->getClientMimeType(), $this->allowedMimes, true)) {
            throw new UnsupportedFileException('Mime type not allowed: '.($file->getMimeType() ?? 'unknown'));
        }
        $size = $file->getSize();
        if ($size === false || $size > $this->maxBytes) {
            throw new UnsupportedFileException('File too large: '.($size ?: '?').' > '.$this->maxBytes);
        }
        $ext = $file->guessExtension() ?: 'bin';
        $name = bin2hex(random_bytes(8)).'.'.$ext;
        try {
            $file->move($this->targetDir, $name);
        } catch (\Throwable $e) {
            throw new UploadHandlerException('Could not store upload', 0, $e);
        }
        return [
            'url'  => rtrim($this->publicUrlPrefix, '/').'/'.$name,
            'size' => $size,
            'type' => $file->getMimeType() ?? $file->getClientMimeType(),
        ];
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Upload/EditorUploadHandlerInterface.php src/Editor/src/Upload/DefaultLocalUploadHandler.php src/Editor/tests/Upload/DefaultLocalUploadHandlerTest.php
git commit -m "feat(editor): add upload handler interface + DefaultLocalUploadHandler"
```

---

## Task 28 — `UploadHandlerRegistry`

**Files:**
- Create: `src/Editor/src/Upload/UploadHandlerRegistry.php`
- Test: `src/Editor/tests/Upload/UploadHandlerRegistryTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Upload;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\UX\Editor\Exception\Upload\UploadHandlerException;
use Symfony\UX\Editor\Upload\EditorUploadHandlerInterface;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

final class UploadHandlerRegistryTest extends TestCase
{
    public function testGetByProfile(): void
    {
        $h = $this->fakeHandler();
        $r = new UploadHandlerRegistry(['default' => $h]);
        self::assertSame($h, $r->get('default'));
    }

    public function testUnknownProfileThrows(): void
    {
        $this->expectException(UploadHandlerException::class);
        (new UploadHandlerRegistry([]))->get('missing');
    }

    private function fakeHandler(): EditorUploadHandlerInterface
    {
        return new class implements EditorUploadHandlerInterface {
            public function handle(UploadedFile $f, array $c = []): array { return ['url' => '/u', 'size' => 1]; }
        };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Upload;

use Symfony\UX\Editor\Exception\Upload\UploadHandlerException;

final class UploadHandlerRegistry
{
    /** @var array<string, EditorUploadHandlerInterface> */
    private array $handlers = [];

    /** @param iterable<string, EditorUploadHandlerInterface> $handlers */
    public function __construct(iterable $handlers = [])
    {
        foreach ($handlers as $name => $h) {
            $this->handlers[$name] = $h;
        }
    }

    public function get(string $profile): EditorUploadHandlerInterface
    {
        return $this->handlers[$profile] ?? throw new UploadHandlerException(sprintf('No upload handler for profile "%s". Registered: %s', $profile, implode(', ', array_keys($this->handlers)) ?: '(none)'));
    }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Upload/UploadHandlerRegistry.php src/Editor/tests/Upload/UploadHandlerRegistryTest.php
git commit -m "feat(editor): add upload handler registry"
```

---

## Task 29 — `EditorUploadController`

**Files:**
- Create: `src/Editor/src/Upload/EditorUploadController.php`
- Test: `src/Editor/tests/Upload/EditorUploadControllerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Upload;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\HttpFoundation\Request;
use Symfony\UX\Editor\Upload\DefaultLocalUploadHandler;
use Symfony\UX\Editor\Upload\EditorUploadController;
use Symfony\UX\Editor\Upload\SignedUploadUrlGenerator;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

final class EditorUploadControllerTest extends TestCase
{
    private string $tmp;

    protected function setUp(): void
    {
        $this->tmp = sys_get_temp_dir().'/ux-editor-ctl-'.uniqid();
        mkdir($this->tmp);
    }
    protected function tearDown(): void
    {
        foreach (glob($this->tmp.'/*') ?: [] as $f) { @unlink($f); }
        @rmdir($this->tmp);
    }

    public function testHappyUpload(): void
    {
        $signer  = new SignedUploadUrlGenerator('s3cret', 3600);
        $handler = new DefaultLocalUploadHandler($this->tmp, '/u', ['image/png'], 1_000_000);
        $registry= new UploadHandlerRegistry(['default' => $handler]);
        $ctl     = new EditorUploadController($signer, $registry, 'default');

        $token = $signer->sign('body', 'default');
        $tmp = tempnam(sys_get_temp_dir(), 'ux');
        file_put_contents($tmp, 'PNGdata');
        $upload = new UploadedFile($tmp, 'img.png', 'image/png', null, true);
        $req = Request::create('/_ux_editor/upload/body?token='.urlencode($token), 'POST', [], [], ['file' => $upload]);

        $resp = $ctl('body', $req);
        self::assertSame(200, $resp->getStatusCode());
        $data = json_decode($resp->getContent(), true);
        self::assertStringStartsWith('/u/', $data['url']);
    }

    public function testBadSignatureReturns403(): void
    {
        $ctl = new EditorUploadController(new SignedUploadUrlGenerator('s3cret', 3600), new UploadHandlerRegistry([]), 'default');
        $req = Request::create('/_ux_editor/upload/body?token=garbage', 'POST');
        self::assertSame(403, $ctl('body', $req)->getStatusCode());
    }

    public function testBadMimeReturns422(): void
    {
        $signer  = new SignedUploadUrlGenerator('s3cret', 3600);
        $handler = new DefaultLocalUploadHandler($this->tmp, '/u', ['image/png'], 1_000_000);
        $registry= new UploadHandlerRegistry(['default' => $handler]);
        $ctl     = new EditorUploadController($signer, $registry, 'default');
        $token = $signer->sign('body', 'default');
        $tmp = tempnam(sys_get_temp_dir(), 'ux');
        file_put_contents($tmp, 'evil');
        $upload = new UploadedFile($tmp, 'e.exe', 'application/x-msdownload', null, true);
        $req = Request::create('/_ux_editor/upload/body?token='.urlencode($token), 'POST', [], [], ['file' => $upload]);
        self::assertSame(422, $ctl('body', $req)->getStatusCode());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Upload;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\UX\Editor\Exception\Upload\InvalidSignatureException;
use Symfony\UX\Editor\Exception\Upload\UnsupportedFileException;
use Symfony\UX\Editor\Exception\Upload\UploadHandlerException;

final class EditorUploadController
{
    public function __construct(
        private readonly SignedUploadUrlGenerator $signer,
        private readonly UploadHandlerRegistry $handlers,
        private readonly string $defaultProfile = 'default',
    ) {}

    public function __invoke(string $field, Request $request): Response
    {
        $token   = (string)$request->query->get('token', '');
        $profile = (string)$request->query->get('profile', $this->defaultProfile);
        try {
            $this->signer->verify($token, $field, $profile);
        } catch (InvalidSignatureException $e) {
            return new JsonResponse(['error' => 'invalid_signature', 'message' => $e->getMessage()], Response::HTTP_FORBIDDEN);
        }
        $file = $request->files->get('file');
        if (!$file) {
            return new JsonResponse(['error' => 'missing_file'], Response::HTTP_UNPROCESSABLE_ENTITY);
        }
        try {
            $result = $this->handlers->get($profile)->handle($file, ['field' => $field, 'profile' => $profile]);
        } catch (UnsupportedFileException $e) {
            return new JsonResponse(['error' => 'unsupported_file', 'message' => $e->getMessage()], Response::HTTP_UNPROCESSABLE_ENTITY);
        } catch (UploadHandlerException $e) {
            return new JsonResponse(['error' => 'handler_error'], Response::HTTP_INTERNAL_SERVER_ERROR);
        }
        return new JsonResponse($result);
    }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Upload/EditorUploadController.php src/Editor/tests/Upload/EditorUploadControllerTest.php
git commit -m "feat(editor): add EditorUploadController (signed URL + JSON responses)"
```

---

## Task 30 — Bundle + DI extension + route

**Files:**
- Create: `src/Editor/src/UXEditorBundle.php`
- Create: `src/Editor/src/DependencyInjection/UXEditorExtension.php`
- Create: `src/Editor/config/services.php`
- Create: `src/Editor/config/routes.php`
- Test: `src/Editor/tests/DependencyInjection/UXEditorExtensionTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\DependencyInjection;

use PHPUnit\Framework\TestCase;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\DependencyInjection\UXEditorExtension;
use Symfony\UX\Editor\Form\EditorType;
use Symfony\UX\Editor\Upload\EditorUploadController;
use Symfony\UX\Editor\Upload\SignedUploadUrlGenerator;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

final class UXEditorExtensionTest extends TestCase
{
    public function testServicesRegistered(): void
    {
        $c = new ContainerBuilder();
        $c->setParameter('kernel.debug', true);
        $c->setParameter('kernel.secret', 'test-secret');
        $c->setParameter('kernel.project_dir', sys_get_temp_dir());
        (new UXEditorExtension())->load([], $c);

        foreach ([
            BridgeRegistry::class,
            PresetRegistry::class,
            ContentConverterRegistry::class,
            UploadHandlerRegistry::class,
            SignedUploadUrlGenerator::class,
            EditorUploadController::class,
            EditorType::class,
        ] as $id) {
            self::assertTrue($c->hasDefinition($id) || $c->hasAlias($id), "Missing service $id");
        }
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`src/UXEditorBundle.php`:
```php
<?php
namespace Symfony\UX\Editor;

use Symfony\Component\DependencyInjection\Extension\ExtensionInterface;
use Symfony\Component\HttpKernel\Bundle\Bundle;
use Symfony\UX\Editor\DependencyInjection\UXEditorExtension;

final class UXEditorBundle extends Bundle
{
    public function getContainerExtension(): ?ExtensionInterface
    {
        return $this->extension ??= new UXEditorExtension();
    }
    public function getPath(): string { return \dirname(__DIR__); }
}
```

`src/DependencyInjection/UXEditorExtension.php`:
```php
<?php
namespace Symfony\UX\Editor\DependencyInjection;

use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

final class UXEditorExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        $loader = new PhpFileLoader($container, new FileLocator(\dirname(__DIR__, 2).'/config'));
        $loader->load('services.php');
    }

    public function getAlias(): string { return 'ux_editor'; }
}
```

`config/services.php`:
```php
<?php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\UX\Editor\Bridge\BridgeInterface;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\Preset\EditorPresetInterface;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\Converter\ContentConverterInterface;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\Form\EditorType;
use Symfony\UX\Editor\Upload\DefaultLocalUploadHandler;
use Symfony\UX\Editor\Upload\EditorUploadController;
use Symfony\UX\Editor\Upload\EditorUploadHandlerInterface;
use Symfony\UX\Editor\Upload\SignedUploadUrlGenerator;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

return static function (ContainerConfigurator $c): void {
    $s = $c->services()->defaults()->autowire()->autoconfigure();

    $s->instanceof(BridgeInterface::class)->tag('ux.editor.bridge');
    $s->instanceof(EditorPresetInterface::class)->tag('ux.editor.preset');
    $s->instanceof(ContentConverterInterface::class)->tag('ux.editor.content_converter');
    $s->instanceof(EditorUploadHandlerInterface::class)->tag('ux.editor.upload_handler');

    $s->set(BridgeRegistry::class)->args([tagged_iterator('ux.editor.bridge')]);
    $s->set(PresetRegistry::class)->args([tagged_iterator('ux.editor.preset', indexAttribute: 'name')]);
    $s->set(ContentConverterRegistry::class)->args([tagged_iterator('ux.editor.content_converter')]);
    $s->set(UploadHandlerRegistry::class)->args([tagged_iterator('ux.editor.upload_handler', indexAttribute: 'profile')]);

    $s->set(SignedUploadUrlGenerator::class)->args(['%kernel.secret%', 3600]);
    $s->set(EditorUploadController::class)->public()->arg('$defaultProfile', 'default');

    $s->set(EditorType::class)
        ->arg('$sanitizer', service('html_sanitizer.sanitizer.default')->nullOnInvalid());

    $s->set('symfony.ux_editor.upload_handler.default', DefaultLocalUploadHandler::class)
        ->args(['%kernel.project_dir%/public/uploads', '/uploads'])
        ->tag('ux.editor.upload_handler', ['profile' => 'default']);
};
```

`config/routes.php`:
```php
<?php
use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;
use Symfony\UX\Editor\Upload\EditorUploadController;

return static function (RoutingConfigurator $r): void {
    $r->add('ux_editor_upload', '/_ux_editor/upload/{field}')
      ->controller(EditorUploadController::class)
      ->methods(['POST'])
      ->requirements(['field' => '[a-zA-Z0-9_.-]+']);
};
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/config/ src/Editor/src/UXEditorBundle.php src/Editor/src/DependencyInjection/ src/Editor/tests/DependencyInjection/
git commit -m "feat(editor): wire bundle, DI extension, services, upload route"
```

---

## Checkpoint after Task 30

- [ ] **Run full suite**

```bash
cd src/Editor && vendor/bin/phpunit
```

Expected: all tests Tasks 2-30 pass; zero failures/errors/warnings.

- [ ] **Coverage spot-check**

```bash
vendor/bin/phpunit --coverage-text
```

Expected: ≥ 80% line coverage across Content/, Config/, Bridge/, Form/, Doctrine/, Upload/, Exception/, DependencyInjection/. Remaining gaps in Live/, Twig/, Bridge/Format/ — covered in Tasks 31-72.

---

## Self-review for Tasks 7-30

**1. Spec coverage:** Tasks 7-15 cover Spec §11 (Exceptions), §5 (Config/Common/Capabilities/Preset), §4.4 (Converter). Tasks 16-22 cover §3 (Bridge tier) + §5.4 (EditorType input modes) + §5.6 (capability guard). Tasks 23-25 cover §4.3 (Doctrine Types). Tasks 26-29 cover §7.2 (Upload pipeline). Task 30 covers §3 (bundle wiring) and §7.2 (route).

**2. Placeholder scan:** all tasks contain runnable PHP code; no `TODO`/`TBD`/`handle errors` handwaves. Task 22 contains a conditional "Step 3 (only if Step 2 fails)" — this is intentional since Step 2 should pass given Tasks 12 + 20 are already wired correctly; the conditional is a safety hatch, not a placeholder.

**3. Type consistency:**
- `EditorContentInterface::getRaw()` returns `string|array` consistently (Task 3 + value objects).
- `EditorContentTransformerInterface` signature stable: Task 16 placeholder (empty), Task 19 final (`transform/reverseTransform`), Task 20 uses final shape, Task 22 uses final shape. ✓
- `StorageShape` enum referenced in Tasks 19, 20, 22 with same `Scalar|Json|Split` cases. ✓
- `BridgeCapabilities` ctor signature stable across Tasks 9, 10, 11, 12, 16-22. ✓
- `CommonOptions` constructor positional args stable across Tasks 8, 11, 12, 22. ✓
- Exception class names stable across Tasks 7, 12, 13, 14, 18, 24, 25, 26, 28, 29.

No fixes needed inline.

---

Next: continue with `expand tasks 31-50` (Live, Twig, Tier 1 Wysiwyg/Block/Page abstracts) or proceed to execution against Tasks 1-30.
