# UX Editor Core — Tasks 51-78 (final batch for Plan 1)

> **Parent plan:** [`2026-05-19-ux-editor-core.md`](2026-05-19-ux-editor-core.md). Continues from [`-tasks-31-50.md`](2026-05-19-ux-editor-core-tasks-31-50.md). Requires Tasks 1-50 complete.

Coverage: Tier 1 Page transformer/bridge, all JS (Tier 0 + Tier 1 controllers, upload client, live debouncer, content mirrors), bundle finalization (config schema, compile pass, debug command, profiler collector), docs, controllers.json, coverage gates, audit, tag.

Tasks 66-68 of the parent overview were partially executed in Task 30 (bundle, DI extension, services.php). They are folded into Tasks 66-69 below as additional work (config schema + compile pass + debug command + registry exposure).

---

## Task 51 — `AbstractPageTransformer`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/AbstractPageTransformer.php`
- Test: `src/Editor/tests/Bridge/Format/Page/AbstractPageTransformerTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageTransformer;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Exception\ContentSchemaException;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

final class AbstractPageTransformerTest extends TestCase
{
    public function testStorageShapeIsJsonByDefault(): void
    {
        $t = $this->fake();
        self::assertSame(StorageShape::Json, $t->getStorageShape());
        self::assertSame(PageContent::class, $t->getContentClass());
    }

    public function testRoundTripBundle(): void
    {
        $t = $this->fake();
        $pc = new PageContent('<h1>x</h1>', 'h1{}', [['type' => 'image', 'url' => '/x.png']], [['type' => 'h1']]);
        $arr = $t->transform($pc);
        self::assertSame('<h1>x</h1>', $arr['html']);
        self::assertSame('h1{}',       $arr['css']);
        self::assertCount(1, $arr['assets']);
        self::assertCount(1, $arr['components']);

        $back = $t->reverseTransform($arr);
        self::assertInstanceOf(PageContent::class, $back);
        self::assertSame('<h1>x</h1>', $back->html);
        self::assertSame('fakepage',   $back->getMetadata()['bridgeId']);
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
        $this->fake()->reverseTransform(['html' => ['not-a-string']]);
    }

    private function fake(): AbstractPageTransformer
    {
        return new class extends AbstractPageTransformer { public function getBridgeId(): string { return 'fakepage'; } };
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Content\EditorContentInterface;
use Symfony\UX\Editor\Content\PageContent;
use Symfony\UX\Editor\Exception\ContentSchemaException;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;
use Symfony\UX\Editor\Form\DataTransformer\StorageShape;

abstract class AbstractPageTransformer implements EditorContentTransformerInterface
{
    public function getContentClass(): string { return PageContent::class; }
    public function getStorageShape(): StorageShape { return StorageShape::Json; }

    public function transform(?EditorContentInterface $content): ?array
    {
        if ($content === null) { return null; }
        if (!$content instanceof PageContent) {
            throw new \InvalidArgumentException(sprintf('Expected PageContent, got %s', get_debug_type($content)));
        }
        return [
            'html'       => $content->html,
            'css'        => $content->css,
            'assets'     => $content->assets,
            'components' => $content->components,
        ];
    }

    public function reverseTransform(mixed $stored): ?PageContent
    {
        if ($stored === null || $stored === []) { return null; }
        if (!\is_array($stored)) {
            throw new ContentSchemaException(sprintf('Expected array, got %s', get_debug_type($stored)));
        }
        $html = $stored['html'] ?? '';
        $css  = $stored['css']  ?? '';
        if (!\is_string($html) || !\is_string($css)) {
            throw new ContentSchemaException('"html" and "css" must be strings');
        }
        return new PageContent(
            html: $html,
            css:  $css,
            assets:     \is_array($stored['assets']     ?? null) ? $stored['assets']     : [],
            components: \is_array($stored['components'] ?? null) ? $stored['components'] : [],
            metadata:   ['bridgeId' => $this->getBridgeId()],
        );
    }

    abstract public function getBridgeId(): string;
}
```

- [ ] **Step 4: Run — expect 4 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/AbstractPageTransformer.php src/Editor/tests/Bridge/Format/Page/AbstractPageTransformerTest.php
git commit -m "feat(editor): add AbstractPageTransformer"
```

---

## Task 52 — `AbstractPageBuilderBridge`

**Files:**
- Create: `src/Editor/src/Bridge/Format/Page/AbstractPageBuilderBridge.php`
- Test: `src/Editor/tests/Bridge/Format/Page/AbstractPageBuilderBridgeTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Bridge\Format\Page;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageBuilderBridge;
use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageConfig;
use Symfony\UX\Editor\Bridge\Format\Page\AbstractPageTransformer;
use Symfony\UX\Editor\Bridge\Format\Page\PageCapabilities;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Form\DataTransformer\EditorContentTransformerInterface;

final class AbstractPageBuilderBridgeTest extends TestCase
{
    public function testDefaults(): void
    {
        $b = new class extends AbstractPageBuilderBridge {
            public function getId(): string { return 'fp'; }
            public function getDefaultConfig(): EditorConfigInterface {
                return new class extends AbstractPageConfig { public function getBridgeId(): string { return 'fp'; } };
            }
            public function createTransformer(): EditorContentTransformerInterface {
                return new class extends AbstractPageTransformer { public function getBridgeId(): string { return 'fp'; } };
            }
        };
        self::assertSame('symfony--ux-editor--fp', $b->getControllerName());
        self::assertEquals(PageCapabilities::default(), $b->getCapabilities());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Bridge\Format\Page;

use Symfony\UX\Editor\Bridge\AbstractBridge;
use Symfony\UX\Editor\Config\BridgeCapabilities;

abstract class AbstractPageBuilderBridge extends AbstractBridge
{
    public function getCapabilities(): BridgeCapabilities { return PageCapabilities::default(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Bridge/Format/Page/AbstractPageBuilderBridge.php src/Editor/tests/Bridge/Format/Page/AbstractPageBuilderBridgeTest.php
git commit -m "feat(editor): add AbstractPageBuilderBridge"
```

---

## Task 53 — JS scaffold: `AbstractEditorController` skeleton + Vitest

**Files:**
- Create: `src/Editor/assets/src/controller.ts`
- Create: `src/Editor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, expect, it, beforeEach } from 'vitest';
import { Application } from '@hotwired/stimulus';
import { AbstractEditorController } from '../src/controller.js';

beforeEach(() => { document.body.innerHTML = ''; });

class DummyController extends AbstractEditorController<{ dom: HTMLElement; config: any }> {
  static values = { ...(AbstractEditorController as any).values };
  async createEditor(mount: HTMLElement, config: any) { return { dom: mount, config }; }
  serialize(i: { dom: HTMLElement }): string { return i.dom.textContent ?? ''; }
  async destroyEditor(_i: any): Promise<void> { /* noop */ }
  hotReloadable(): Set<string> { return new Set(); }
  applyConfig(_d: Record<string, unknown>, _i: any): void {}
}

describe('AbstractEditorController', () => {
  it('declares expected values + targets', () => {
    expect((DummyController as any).values).toHaveProperty('config');
    expect((DummyController as any).values).toHaveProperty('format');
    expect((DummyController as any).values).toHaveProperty('bridgeId');
    expect((DummyController as any).targets).toContain('input');
    expect((DummyController as any).targets).toContain('mount');
  });

  it('registers without throwing', () => {
    const app = Application.start();
    expect(() => app.register('dummy', DummyController as any)).not.toThrow();
    app.stop();
  });
});
```

- [ ] **Step 2: Run — expect fail**

```bash
cd src/Editor && npx vitest run assets/test/controller.test.ts
```

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/controller.ts
import { Controller } from '@hotwired/stimulus';

export abstract class AbstractEditorController<T = any> extends Controller<HTMLElement> {
  static values = {
    config: Object,
    format: String,
    bridgeId: String,
    uploadUrl: String,
  };
  static targets = ['input', 'mount'];

  declare readonly configValue: Record<string, unknown>;
  declare readonly formatValue: string;
  declare readonly bridgeIdValue: string;
  declare readonly uploadUrlValue: string;
  declare readonly inputTarget: HTMLInputElement | HTMLTextAreaElement;
  declare readonly mountTarget: HTMLElement;

  protected instance?: T;

  abstract createEditor(mount: HTMLElement, config: Record<string, unknown>): Promise<T>;
  abstract serialize(instance: T): string | object | Promise<string | object>;
  abstract destroyEditor(instance: T): Promise<void> | void;
  abstract hotReloadable(): Set<string>;
  abstract applyConfig(diff: Record<string, unknown>, instance: T): Promise<void> | void;
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/controller.ts src/Editor/assets/test/controller.test.ts
git commit -m "feat(editor): add AbstractEditorController skeleton"
```

---

## Task 54 — `connect()` lifecycle + dispatch events

**Files:**
- Modify: `src/Editor/assets/src/controller.ts`
- Modify: `src/Editor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (append)**

```ts
it('connect dispatches pre-connect then connect, creates instance', async () => {
  document.body.innerHTML = `
    <div data-controller="dummy"
         data-dummy-config-value='{"foo":1}'
         data-dummy-format-value='html'
         data-dummy-bridge-id-value='fake'>
      <textarea data-dummy-target="input"></textarea>
      <div data-dummy-target="mount">hi</div>
    </div>`;
  const root = document.querySelector('[data-controller="dummy"]')!;
  const events: string[] = [];
  ['ux:editor:pre-connect', 'ux:editor:connect'].forEach(n => root.addEventListener(n, () => events.push(n)));

  const app = Application.start();
  app.register('dummy', DummyController as any);
  await new Promise(r => setTimeout(r, 0));

  expect(events).toEqual(['ux:editor:pre-connect', 'ux:editor:connect']);
  app.stop();
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — append to `controller.ts`**

```ts
async connect(): Promise<void> {
  this.element.dispatchEvent(new CustomEvent('ux:editor:pre-connect', {
    bubbles: true,
    detail: { bridgeId: this.bridgeIdValue, format: this.formatValue, config: this.configValue },
  }));
  this.instance = await this.createEditor(this.mountTarget, this.configValue);
  this.element.dispatchEvent(new CustomEvent('ux:editor:connect', {
    bubbles: true,
    detail: { bridgeId: this.bridgeIdValue, instance: this.instance },
  }));
}
```

- [ ] **Step 4: Run — expect 1 pass + Task 53 still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/controller.ts src/Editor/assets/test/controller.test.ts
git commit -m "feat(editor): wire AbstractEditorController.connect"
```

---

## Task 55 — `syncInput()` + change events

**Files:**
- Modify: `src/Editor/assets/src/controller.ts`
- Modify: `src/Editor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (append)**

```ts
it('syncInput writes serialized value + dispatches ux:editor:change', async () => {
  document.body.innerHTML = `
    <div data-controller="dummy" data-dummy-config-value='{}' data-dummy-format-value='html' data-dummy-bridge-id-value='fake'>
      <textarea data-dummy-target="input"></textarea>
      <div data-dummy-target="mount">hello</div>
    </div>`;
  const root = document.querySelector('[data-controller="dummy"]')!;
  const events: any[] = [];
  root.addEventListener('ux:editor:change', (e: any) => events.push(e.detail));

  const app = Application.start();
  app.register('dummy', DummyController as any);
  await new Promise(r => setTimeout(r, 0));
  const ctrl = (app as any).getControllerForElementAndIdentifier(root, 'dummy');
  ctrl.syncInput();

  expect((root.querySelector('textarea') as HTMLTextAreaElement).value).toBe('hello');
  expect(events[0]).toMatchObject({ value: 'hello', format: 'html' });
  app.stop();
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — append to `controller.ts`**

```ts
syncInput(): void {
  if (this.instance === undefined) return;
  const value = this.serialize(this.instance);
  const finalize = (v: string | object) => {
    this.inputTarget.value = typeof v === 'string' ? v : JSON.stringify(v);
    this.element.dispatchEvent(new CustomEvent('ux:editor:change', {
      bubbles: true,
      detail: { value: v, format: this.formatValue, bridgeId: this.bridgeIdValue },
    }));
  };
  if (value && typeof (value as any).then === 'function') {
    (value as Promise<string | object>).then(finalize);
  } else {
    finalize(value as string | object);
  }
}
```

- [ ] **Step 4: Run — expect 1 pass + earlier tests green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/controller.ts src/Editor/assets/test/controller.test.ts
git commit -m "feat(editor): add syncInput + ux:editor:change"
```

---

## Task 56 — `configValueChanged()` hot vs remount

**Files:**
- Modify: `src/Editor/assets/src/controller.ts`
- Modify: `src/Editor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (append)**

```ts
it('hot diff calls applyConfig, no remount', async () => {
  let applied: any = null;
  let created = 0;
  class HotDummy extends AbstractEditorController<{}> {
    static values = { ...(AbstractEditorController as any).values };
    async createEditor(): Promise<{}> { created++; return {}; }
    serialize(): string { return ''; }
    async destroyEditor(): Promise<void> {}
    hotReloadable(): Set<string> { return new Set(['readOnly']); }
    applyConfig(diff: Record<string, unknown>): void { applied = diff; }
  }
  document.body.innerHTML = `<div data-controller="hot" data-hot-config-value='{"readOnly":false}' data-hot-format-value='html' data-hot-bridge-id-value='fake'><textarea data-hot-target="input"></textarea><div data-hot-target="mount"></div></div>`;
  const app = Application.start();
  app.register('hot', HotDummy as any);
  await new Promise(r => setTimeout(r, 0));
  const ctrl = (app as any).getControllerForElementAndIdentifier(document.querySelector('[data-controller="hot"]'), 'hot');
  await ctrl.configValueChanged({ readOnly: true }, { readOnly: false });
  expect(applied).toEqual({ readOnly: true });
  expect(created).toBe(1);
  app.stop();
});

it('non-hot diff destroys + recreates', async () => {
  let destroyed = 0, created = 0;
  class ColdDummy extends AbstractEditorController<{}> {
    static values = { ...(AbstractEditorController as any).values };
    async createEditor(): Promise<{}> { created++; return {}; }
    serialize(): string { return ''; }
    async destroyEditor(): Promise<void> { destroyed++; }
    hotReloadable(): Set<string> { return new Set(['readOnly']); }
    applyConfig(): void {}
  }
  document.body.innerHTML = `<div data-controller="cold" data-cold-config-value='{"toolbar":[]}' data-cold-format-value='html' data-cold-bridge-id-value='fake'><textarea data-cold-target="input"></textarea><div data-cold-target="mount"></div></div>`;
  const app = Application.start();
  app.register('cold', ColdDummy as any);
  await new Promise(r => setTimeout(r, 0));
  const ctrl = (app as any).getControllerForElementAndIdentifier(document.querySelector('[data-controller="cold"]'), 'cold');
  await ctrl.configValueChanged({ toolbar: ['bold'] }, { toolbar: [] });
  expect(destroyed).toBe(1);
  expect(created).toBe(2);
  app.stop();
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — append to `controller.ts`**

```ts
async configValueChanged(newCfg: Record<string, unknown>, oldCfg?: Record<string, unknown>): Promise<void> {
  if (!this.instance || oldCfg === undefined) return;
  const diff = this.diff(newCfg, oldCfg);
  if (Object.keys(diff).length === 0) return;

  const hot = this.hotReloadable();
  const allHot = Object.keys(diff).every(k => hot.has(k));
  if (allHot) {
    await this.applyConfig(diff, this.instance);
    return;
  }
  await this.destroyEditor(this.instance);
  this.element.dispatchEvent(new CustomEvent('ux:editor:remount', { bubbles: true, detail: { reason: 'non-hot-keys', diff } }));
  this.instance = await this.createEditor(this.mountTarget, newCfg);
}

protected diff(a: Record<string, unknown>, b: Record<string, unknown>): Record<string, unknown> {
  const out: Record<string, unknown> = {};
  const keys = new Set([...Object.keys(a), ...Object.keys(b)]);
  for (const k of keys) {
    if (JSON.stringify(a[k]) !== JSON.stringify(b[k])) {
      out[k] = a[k];
    }
  }
  return out;
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/controller.ts src/Editor/assets/test/controller.test.ts
git commit -m "feat(editor): add configValueChanged hot vs remount"
```

---

## Task 57 — `disconnect()` lifecycle

**Files:**
- Modify: `src/Editor/assets/src/controller.ts`
- Modify: `src/Editor/assets/test/controller.test.ts`

- [ ] **Step 1: Failing test (append)**

```ts
it('disconnect destroys + dispatches ux:editor:destroy', async () => {
  let destroyed = 0;
  class D extends AbstractEditorController<{}> {
    static values = { ...(AbstractEditorController as any).values };
    async createEditor(): Promise<{}> { return {}; }
    serialize(): string { return ''; }
    async destroyEditor(): Promise<void> { destroyed++; }
    hotReloadable(): Set<string> { return new Set(); }
    applyConfig(): void {}
  }
  document.body.innerHTML = `<div data-controller="d" data-d-config-value='{}' data-d-format-value='html' data-d-bridge-id-value='fake'><textarea data-d-target="input"></textarea><div data-d-target="mount"></div></div>`;
  const events: string[] = [];
  document.querySelector('[data-controller="d"]')!.addEventListener('ux:editor:destroy', () => events.push('destroy'));
  const app = Application.start();
  app.register('d', D as any);
  await new Promise(r => setTimeout(r, 0));
  document.body.innerHTML = '';
  await new Promise(r => setTimeout(r, 0));
  expect(destroyed).toBe(1);
  expect(events).toEqual(['destroy']);
  app.stop();
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation — append to `controller.ts`**

```ts
async disconnect(): Promise<void> {
  if (this.instance !== undefined) {
    await this.destroyEditor(this.instance);
    this.element.dispatchEvent(new CustomEvent('ux:editor:destroy', { bubbles: true, detail: { bridgeId: this.bridgeIdValue } }));
    this.instance = undefined;
  }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/controller.ts src/Editor/assets/test/controller.test.ts
git commit -m "feat(editor): add disconnect lifecycle"
```

---

## Task 58 — `SignedUploadClient`

**Files:**
- Create: `src/Editor/assets/src/upload/SignedUploadClient.ts`
- Create: `src/Editor/assets/test/upload.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { SignedUploadClient } from '../src/upload/SignedUploadClient.js';

describe('SignedUploadClient', () => {
  beforeEach(() => { vi.stubGlobal('fetch', vi.fn()); });
  afterEach(()  => { vi.unstubAllGlobals(); });

  it('POSTs multipart and returns JSON on 200', async () => {
    (fetch as any).mockResolvedValue(new Response(JSON.stringify({ url: '/u/abc.png' }), { status: 200 }));
    const c = new SignedUploadClient('/_ux_editor/upload/body?token=t', { field: 'body' });
    const r = await c.upload(new Blob(['xx'], { type: 'image/png' }), 'i.png');
    expect(r.url).toBe('/u/abc.png');
    const call = (fetch as any).mock.calls[0];
    expect(call[0]).toBe('/_ux_editor/upload/body?token=t');
    expect(call[1].method).toBe('POST');
    expect(call[1].body).toBeInstanceOf(FormData);
  });

  it('throws on non-2xx with structured error', async () => {
    (fetch as any).mockResolvedValue(new Response(JSON.stringify({ error: 'unsupported_file', message: 'bad mime' }), { status: 422 }));
    const c = new SignedUploadClient('/u?token=t', { field: 'body' });
    await expect(c.upload(new Blob(['x']), 'x.exe'))
      .rejects.toMatchObject({ status: 422, code: 'unsupported_file', message: 'bad mime' });
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/upload/SignedUploadClient.ts
export interface UploadResult { url: string; size?: number; type?: string; width?: number; height?: number }
export interface UploadOptions { field: string }

export class SignedUploadClient {
  constructor(private readonly url: string, private readonly options: UploadOptions) {}

  async upload(file: Blob, filename: string): Promise<UploadResult> {
    const fd = new FormData();
    fd.append('file', file, filename);
    fd.append('field', this.options.field);

    const res = await fetch(this.url, { method: 'POST', body: fd });
    const text = await res.text();
    let payload: any = {};
    try { payload = text ? JSON.parse(text) : {}; } catch { /* server returned non-JSON */ }

    if (!res.ok) {
      const err: any = new Error(payload.message ?? `Upload failed: ${res.status}`);
      err.status = res.status;
      err.code = payload.error ?? 'unknown_error';
      throw err;
    }
    return payload as UploadResult;
  }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/upload/SignedUploadClient.ts src/Editor/assets/test/upload.test.ts
git commit -m "feat(editor): add SignedUploadClient"
```

---

## Task 59 — `setupAutosave` debouncer

**Files:**
- Create: `src/Editor/assets/src/live/live-editor.ts`
- Create: `src/Editor/assets/test/live.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect, vi } from 'vitest';
import { setupAutosave } from '../src/live/live-editor.js';

describe('setupAutosave', () => {
  it('debounces ux:editor:change, dispatches with field + last value', async () => {
    vi.useFakeTimers();
    const dispatched: any[] = [];
    const root = document.createElement('div');
    document.body.append(root);
    setupAutosave(root, {
      field: 'body',
      debounceMs: 100,
      dispatch: (field, content) => { dispatched.push({ field, content }); return Promise.resolve(); },
    });

    root.dispatchEvent(new CustomEvent('ux:editor:change', { bubbles: true, detail: { value: 'a' } }));
    root.dispatchEvent(new CustomEvent('ux:editor:change', { bubbles: true, detail: { value: 'b' } }));
    expect(dispatched).toEqual([]);
    vi.advanceTimersByTime(99);
    expect(dispatched).toEqual([]);
    vi.advanceTimersByTime(1);
    await Promise.resolve();
    expect(dispatched).toEqual([{ field: 'body', content: 'b' }]);
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/live/live-editor.ts
export interface AutosaveOptions {
  field: string;
  debounceMs?: number;
  dispatch: (field: string, content: unknown) => Promise<void>;
}

export function setupAutosave(root: HTMLElement, opts: AutosaveOptions): () => void {
  const debounceMs = opts.debounceMs ?? 800;
  let timer: ReturnType<typeof setTimeout> | undefined;
  let lastValue: unknown;

  const handler = (e: Event) => {
    const ev = e as CustomEvent<{ value: unknown }>;
    lastValue = ev.detail?.value;
    if (timer !== undefined) clearTimeout(timer);
    timer = setTimeout(() => {
      timer = undefined;
      opts.dispatch(opts.field, lastValue).catch(() => { /* host surfaces errors */ });
    }, debounceMs);
  };

  root.addEventListener('ux:editor:change', handler);
  return () => {
    root.removeEventListener('ux:editor:change', handler);
    if (timer !== undefined) clearTimeout(timer);
  };
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/live/live-editor.ts src/Editor/assets/test/live.test.ts
git commit -m "feat(editor): add setupAutosave debouncer"
```

---

## Task 60 — Content JS mirrors

**Files:**
- Create: `src/Editor/assets/src/content/EditorContent.ts`
- Create: `src/Editor/assets/src/content/HtmlContent.ts`
- Create: `src/Editor/assets/src/content/BlockContent.ts`
- Create: `src/Editor/assets/src/content/PageContent.ts`
- Create: `src/Editor/assets/test/content.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect } from 'vitest';
import { HtmlContent } from '../src/content/HtmlContent.js';
import { BlockContent } from '../src/content/BlockContent.js';
import { PageContent } from '../src/content/PageContent.js';
import { EditorContentFormat } from '../src/content/EditorContent.js';

describe('content mirrors', () => {
  it('HtmlContent', () => {
    const c = HtmlContent.from('<p>x</p>', { bridgeId: 'ck' });
    expect(c.format).toBe(EditorContentFormat.Html);
    expect(c.getRaw()).toBe('<p>x</p>');
    expect(c.metadata.bridgeId).toBe('ck');
  });
  it('BlockContent', () => {
    const c = BlockContent.from({ version: '2.0', blocks: [{ type: 'p', data: {} }] });
    expect(c.format).toBe(EditorContentFormat.Blocks);
    expect(c.schemaVersion).toBe('2.0');
    expect(c.blocks.length).toBe(1);
  });
  it('PageContent', () => {
    const c = PageContent.from({ html: '<h1>x</h1>', css: 'h1{}', assets: [], components: [] });
    expect(c.format).toBe(EditorContentFormat.Page);
    expect(c.html).toBe('<h1>x</h1>');
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`EditorContent.ts`:
```ts
export enum EditorContentFormat { Html = 'html', Blocks = 'blocks', Page = 'page' }

export abstract class EditorContent<T = unknown> {
  constructor(public readonly format: EditorContentFormat, public readonly metadata: Record<string, unknown> = {}) {}
  abstract getRaw(): T;
  abstract isEmpty(): boolean;
}
```

`HtmlContent.ts`:
```ts
import { EditorContent, EditorContentFormat } from './EditorContent.js';

export class HtmlContent extends EditorContent<string> {
  constructor(public readonly html: string, metadata: Record<string, unknown> = {}) {
    super(EditorContentFormat.Html, metadata);
  }
  getRaw(): string  { return this.html; }
  isEmpty(): boolean { return this.html.replace(/<[^>]*>/g, '').trim() === ''; }
  static from(html: string, metadata: Record<string, unknown> = {}): HtmlContent { return new HtmlContent(html, metadata); }
}
```

`BlockContent.ts`:
```ts
import { EditorContent, EditorContentFormat } from './EditorContent.js';

export interface Block { type: string; data: Record<string, unknown>; id?: string }

export class BlockContent extends EditorContent<Block[]> {
  constructor(
    public readonly blocks: Block[],
    public readonly schemaVersion: string = '1.0',
    metadata: Record<string, unknown> = {},
  ) { super(EditorContentFormat.Blocks, metadata); }
  getRaw(): Block[] { return this.blocks; }
  isEmpty(): boolean { return this.blocks.length === 0; }
  static from(payload: { version?: string; blocks?: Block[] }, metadata: Record<string, unknown> = {}): BlockContent {
    return new BlockContent(payload.blocks ?? [], payload.version ?? '1.0', metadata);
  }
}
```

`PageContent.ts`:
```ts
import { EditorContent, EditorContentFormat } from './EditorContent.js';

export interface PageBundle { html: string; css: string; assets: unknown[]; components: unknown[] }

export class PageContent extends EditorContent<PageBundle> {
  constructor(
    public readonly html: string,
    public readonly css: string = '',
    public readonly assets: unknown[] = [],
    public readonly components: unknown[] = [],
    metadata: Record<string, unknown> = {},
  ) { super(EditorContentFormat.Page, metadata); }
  getRaw(): PageBundle { return { html: this.html, css: this.css, assets: this.assets, components: this.components }; }
  isEmpty(): boolean { return this.html === '' && this.components.length === 0; }
  static from(bundle: Partial<PageBundle>, metadata: Record<string, unknown> = {}): PageContent {
    return new PageContent(bundle.html ?? '', bundle.css ?? '', bundle.assets ?? [], bundle.components ?? [], metadata);
  }
}
```

- [ ] **Step 4: Run — expect 3 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/content/ src/Editor/assets/test/content.test.ts
git commit -m "feat(editor): add JS content mirrors"
```

---

## Task 61 — `AbstractWysiwygController`

**Files:**
- Create: `src/Editor/assets/src/format/wysiwyg_controller.ts`
- Create: `src/Editor/assets/test/format/wysiwyg.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect } from 'vitest';
import { Application } from '@hotwired/stimulus';
import { AbstractWysiwygController, type WysiwygInstance } from '../../src/format/wysiwyg_controller.js';

class Fake extends AbstractWysiwygController {
  static values = { ...(AbstractWysiwygController as any).values };
  async createEditor(): Promise<WysiwygInstance> { return { getHTML: () => '<p>x</p>' }; }
  hotReloadable(): Set<string> { return new Set(['readOnly']); }
  applyConfig(): void {}
  async destroyEditor(): Promise<void> {}
}

describe('AbstractWysiwygController', () => {
  it('serialize returns instance.getHTML()', async () => {
    document.body.innerHTML = `<div data-controller="w" data-w-config-value='{}' data-w-format-value='html' data-w-bridge-id-value='fake'><textarea data-w-target="input"></textarea><div data-w-target="mount"></div></div>`;
    const app = Application.start();
    app.register('w', Fake as any);
    await new Promise(r => setTimeout(r, 0));
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(document.querySelector('[data-controller="w"]'), 'w');
    expect(ctrl.serialize(ctrl.instance)).toBe('<p>x</p>');
    app.stop();
  });

  it('sets aria-multiline on mount target', async () => {
    document.body.innerHTML = `<div data-controller="w" data-w-config-value='{}' data-w-format-value='html' data-w-bridge-id-value='fake'><textarea data-w-target="input"></textarea><div data-w-target="mount"></div></div>`;
    const app = Application.start();
    app.register('w', Fake as any);
    await new Promise(r => setTimeout(r, 0));
    expect(document.querySelector('[data-w-target="mount"]')!.getAttribute('aria-multiline')).toBe('true');
    app.stop();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/format/wysiwyg_controller.ts
import { AbstractEditorController } from '../controller.js';

export interface WysiwygInstance { getHTML(): string; destroy?(): void }

export abstract class AbstractWysiwygController extends AbstractEditorController<WysiwygInstance> {
  static values = { ...(AbstractEditorController as any).values, sanitizeOnPaste: Boolean };

  serialize(instance: WysiwygInstance): string { return instance.getHTML(); }

  async connect(): Promise<void> {
    this.mountTarget.setAttribute('aria-multiline', 'true');
    await super.connect();
  }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/format/wysiwyg_controller.ts src/Editor/assets/test/format/wysiwyg.test.ts
git commit -m "feat(editor): add AbstractWysiwygController"
```

---

## Task 62 — `AbstractBlockController`

**Files:**
- Create: `src/Editor/assets/src/format/block_controller.ts`
- Create: `src/Editor/assets/test/format/block.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect } from 'vitest';
import { Application } from '@hotwired/stimulus';
import { AbstractBlockController } from '../../src/format/block_controller.js';

class Fake extends AbstractBlockController {
  static values = { ...(AbstractBlockController as any).values };
  async createEditor(): Promise<any> { return { save: async () => ({ version: '1.0', blocks: [{ type: 'p', data: { text: 'hi' } }] }) }; }
  hotReloadable(): Set<string> { return new Set(); }
  applyConfig(): void {}
  async destroyEditor(): Promise<void> {}
}

describe('AbstractBlockController', () => {
  it('serialize awaits editor.save()', async () => {
    document.body.innerHTML = `<div data-controller="b" data-b-config-value='{}' data-b-format-value='blocks' data-b-bridge-id-value='fb'><textarea data-b-target="input"></textarea><div data-b-target="mount"></div></div>`;
    const app = Application.start();
    app.register('b', Fake as any);
    await new Promise(r => setTimeout(r, 0));
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(document.querySelector('[data-controller="b"]'), 'b');
    const out: any = await ctrl.serialize(ctrl.instance);
    expect(out.blocks[0].type).toBe('p');
    app.stop();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/format/block_controller.ts
import { AbstractEditorController } from '../controller.js';

export interface BlockInstance {
  save(): Promise<{ version?: string; blocks: Array<{ type: string; data: Record<string, unknown> }> }>;
  destroy?(): void;
}

export abstract class AbstractBlockController extends AbstractEditorController<BlockInstance> {
  async serialize(instance: BlockInstance): Promise<object> { return await instance.save(); }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/format/block_controller.ts src/Editor/assets/test/format/block.test.ts
git commit -m "feat(editor): add AbstractBlockController"
```

---

## Task 63 — `AbstractPageBuilderController`

**Files:**
- Create: `src/Editor/assets/src/format/page_builder_controller.ts`
- Create: `src/Editor/assets/test/format/page.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect } from 'vitest';
import { Application } from '@hotwired/stimulus';
import { AbstractPageBuilderController } from '../../src/format/page_builder_controller.js';

class Fake extends AbstractPageBuilderController {
  static values = { ...(AbstractPageBuilderController as any).values };
  async createEditor(): Promise<any> {
    return {
      getHtml: () => '<h1>X</h1>',
      getCss:  () => 'h1{color:red}',
      getComponents: () => [{ type: 'h1' }],
      getAssets: () => [{ type: 'image', url: '/a.png' }],
    };
  }
  hotReloadable(): Set<string> { return new Set(); }
  applyConfig(): void {}
  async destroyEditor(): Promise<void> {}
}

describe('AbstractPageBuilderController', () => {
  it('serialize returns bundle shape', async () => {
    document.body.innerHTML = `<div data-controller="p" data-p-config-value='{}' data-p-format-value='page' data-p-bridge-id-value='fp'><textarea data-p-target="input"></textarea><div data-p-target="mount"></div></div>`;
    const app = Application.start();
    app.register('p', Fake as any);
    await new Promise(r => setTimeout(r, 0));
    const ctrl: any = (app as any).getControllerForElementAndIdentifier(document.querySelector('[data-controller="p"]'), 'p');
    const out: any = ctrl.serialize(ctrl.instance);
    expect(out.html).toBe('<h1>X</h1>');
    expect(out.css).toBe('h1{color:red}');
    expect(out.components[0].type).toBe('h1');
    expect(out.assets[0].url).toBe('/a.png');
    app.stop();
  });
});
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```ts
// src/Editor/assets/src/format/page_builder_controller.ts
import { AbstractEditorController } from '../controller.js';

export interface PageInstance {
  getHtml(): string;
  getCss(): string;
  getComponents(): unknown[];
  getAssets(): unknown[];
  destroy?(): void;
}

export abstract class AbstractPageBuilderController extends AbstractEditorController<PageInstance> {
  serialize(instance: PageInstance): object {
    return { html: instance.getHtml(), css: instance.getCss(), components: instance.getComponents(), assets: instance.getAssets() };
  }
}
```

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/src/format/page_builder_controller.ts src/Editor/assets/test/format/page.test.ts
git commit -m "feat(editor): add AbstractPageBuilderController"
```

---

## Task 64 — Cross-format event consistency test

**Files:**
- Create: `src/Editor/assets/test/format/events.test.ts`

- [ ] **Step 1: Failing test**

```ts
import { describe, it, expect } from 'vitest';
import { Application } from '@hotwired/stimulus';
import { AbstractWysiwygController } from '../../src/format/wysiwyg_controller.js';
import { AbstractBlockController } from '../../src/format/block_controller.js';
import { AbstractPageBuilderController } from '../../src/format/page_builder_controller.js';

async function lifecycle(cls: any, id: string, fmt: string): Promise<string[]> {
  document.body.innerHTML = `<div data-controller="${id}" data-${id}-config-value='{}' data-${id}-format-value='${fmt}' data-${id}-bridge-id-value='${id}'><textarea data-${id}-target="input"></textarea><div data-${id}-target="mount"></div></div>`;
  const root = document.querySelector(`[data-controller="${id}"]`)!;
  const events: string[] = [];
  ['ux:editor:pre-connect', 'ux:editor:connect', 'ux:editor:destroy'].forEach(n =>
    root.addEventListener(n, () => events.push(n))
  );
  const app = Application.start();
  app.register(id, cls);
  await new Promise(r => setTimeout(r, 0));
  document.body.innerHTML = '';
  await new Promise(r => setTimeout(r, 0));
  app.stop();
  return events;
}

describe('cross-format event consistency', () => {
  it('all Tier 1 controllers emit pre-connect, connect, destroy', async () => {
    class W extends AbstractWysiwygController {
      static values = { ...(AbstractWysiwygController as any).values };
      async createEditor(): Promise<any> { return { getHTML: () => '' }; }
      hotReloadable() { return new Set<string>(); } applyConfig() {} async destroyEditor() {}
    }
    class B extends AbstractBlockController {
      static values = { ...(AbstractBlockController as any).values };
      async createEditor(): Promise<any> { return { save: async () => ({ blocks: [] }) }; }
      hotReloadable() { return new Set<string>(); } applyConfig() {} async destroyEditor() {}
    }
    class P extends AbstractPageBuilderController {
      static values = { ...(AbstractPageBuilderController as any).values };
      async createEditor(): Promise<any> { return { getHtml: () => '', getCss: () => '', getComponents: () => [], getAssets: () => [] }; }
      hotReloadable() { return new Set<string>(); } applyConfig() {} async destroyEditor() {}
    }
    for (const [cls, id, fmt] of [[W, 'w1', 'html'], [B, 'b1', 'blocks'], [P, 'p1', 'page']] as const) {
      expect(await lifecycle(cls, id, fmt)).toEqual(['ux:editor:pre-connect', 'ux:editor:connect', 'ux:editor:destroy']);
    }
  });
});
```

- [ ] **Step 2: Run — expect 1 pass**

- [ ] **Step 3: Implementation** — none.

- [ ] **Step 4: Run — expect 1 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/test/format/events.test.ts
git commit -m "test(editor): cross-format event consistency"
```

---

## Task 65 — TypeScript build

**Files:**
- Build verification only

- [ ] **Step 1: Typecheck**

```bash
cd src/Editor && npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 2: Emit `dist/`**

```bash
npx tsc
ls dist/
```

Expected: `controller.js`, `controller.d.ts`, `content/`, `format/`, `upload/`, `live/`.

- [ ] **Step 3: Smoke import from emitted output**

```bash
node -e "import('./dist/controller.js').then(m => console.log(typeof m.AbstractEditorController))"
```

Expected: `function`.

- [ ] **Step 4: If `tsc` reports errors — fix smallest diff in `tsconfig.json`** (likely `"moduleResolution": "bundler"`, `"target": "ES2022"`, `"declaration": true`, `"strict": true`) and commit.

```bash
git add src/Editor/tsconfig.json
git commit -m "chore(editor): tsconfig adjustments for clean build"
```

- [ ] **Step 5: Verify `dist/` ignored** (added in Task 1)

```bash
grep -q '^/dist/' src/Editor/.gitignore && echo OK
```

---

## Task 66 — Bundle config schema

**Files:**
- Create: `src/Editor/src/DependencyInjection/Configuration.php`
- Modify: `src/Editor/src/DependencyInjection/UXEditorExtension.php`
- Modify: `src/Editor/config/services.php`
- Test: `src/Editor/tests/DependencyInjection/ConfigurationTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\DependencyInjection;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Config\Definition\Processor;
use Symfony\UX\Editor\DependencyInjection\Configuration;

final class ConfigurationTest extends TestCase
{
    public function testDefaults(): void
    {
        $cfg = (new Processor())->processConfiguration(new Configuration(), [[]]);
        self::assertTrue($cfg['html']['sanitize_required']);
        self::assertSame('default', $cfg['upload']['default_profile']);
        self::assertSame(3600, $cfg['upload']['ttl_seconds']);
    }

    public function testOverrides(): void
    {
        $cfg = (new Processor())->processConfiguration(new Configuration(), [[
            'html'   => ['sanitize_required' => false],
            'upload' => ['default_profile' => 'images', 'ttl_seconds' => 600],
        ]]);
        self::assertFalse($cfg['html']['sanitize_required']);
        self::assertSame('images', $cfg['upload']['default_profile']);
        self::assertSame(600, $cfg['upload']['ttl_seconds']);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

`Configuration.php`:
```php
<?php
namespace Symfony\UX\Editor\DependencyInjection;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

final class Configuration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $tb = new TreeBuilder('ux_editor');
        $tb->getRootNode()
            ->children()
                ->arrayNode('html')->addDefaultsIfNotSet()
                    ->children()
                        ->booleanNode('sanitize_required')->defaultTrue()->end()
                    ->end()
                ->end()
                ->arrayNode('upload')->addDefaultsIfNotSet()
                    ->children()
                        ->scalarNode('default_profile')->defaultValue('default')->end()
                        ->integerNode('ttl_seconds')->defaultValue(3600)->end()
                    ->end()
                ->end()
            ->end();
        return $tb;
    }
}
```

Replace `UXEditorExtension::load()`:
```php
public function load(array $configs, ContainerBuilder $container): void
{
    $config = $this->processConfiguration(new Configuration(), $configs);

    $loader = new PhpFileLoader($container, new FileLocator(\dirname(__DIR__, 2).'/config'));
    $loader->load('services.php');

    $container->setParameter('ux_editor.html.sanitize_required', $config['html']['sanitize_required']);
    $container->setParameter('ux_editor.upload.default_profile', $config['upload']['default_profile']);
    $container->setParameter('ux_editor.upload.ttl_seconds',     $config['upload']['ttl_seconds']);
}

public function getConfiguration(array $config, ContainerBuilder $container): ?\Symfony\Component\Config\Definition\ConfigurationInterface
{
    return new Configuration();
}
```

Update `services.php` to consume parameters:
```php
$s->set(SignedUploadUrlGenerator::class)->args(['%kernel.secret%', '%ux_editor.upload.ttl_seconds%']);
$s->set(EditorUploadController::class)->public()->arg('$defaultProfile', '%ux_editor.upload.default_profile%');
```

- [ ] **Step 4: Run — expect 2 pass + Task 30 extension test still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/DependencyInjection/Configuration.php src/Editor/src/DependencyInjection/UXEditorExtension.php src/Editor/config/services.php src/Editor/tests/DependencyInjection/ConfigurationTest.php
git commit -m "feat(editor): add bundle config schema"
```

---

## Task 67 — Boot-time sanitizer check (compile pass)

**Files:**
- Create: `src/Editor/src/DependencyInjection/Compiler/AssertSanitizerPass.php`
- Modify: `src/Editor/src/UXEditorBundle.php`
- Test: `src/Editor/tests/DependencyInjection/AssertSanitizerPassTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\DependencyInjection;

use PHPUnit\Framework\TestCase;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;
use Symfony\UX\Editor\DependencyInjection\Compiler\AssertSanitizerPass;

final class AssertSanitizerPassTest extends TestCase
{
    public function testThrowsWhenRequiredButMissing(): void
    {
        $c = new ContainerBuilder();
        $c->setParameter('ux_editor.html.sanitize_required', true);
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessageMatches('/html_sanitizer.sanitizer.default/');
        (new AssertSanitizerPass())->process($c);
    }

    public function testPassesWhenSanitizerRegistered(): void
    {
        $c = new ContainerBuilder();
        $c->setParameter('ux_editor.html.sanitize_required', true);
        $c->setDefinition('html_sanitizer.sanitizer.default', new Definition(\stdClass::class));
        (new AssertSanitizerPass())->process($c);
        $this->addToAssertionCount(1);
    }

    public function testSilentWhenNotRequired(): void
    {
        $c = new ContainerBuilder();
        $c->setParameter('ux_editor.html.sanitize_required', false);
        (new AssertSanitizerPass())->process($c);
        $this->addToAssertionCount(1);
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

final class AssertSanitizerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->hasParameter('ux_editor.html.sanitize_required') || !$container->getParameter('ux_editor.html.sanitize_required')) {
            return;
        }
        if (!$container->hasDefinition('html_sanitizer.sanitizer.default') && !$container->hasAlias('html_sanitizer.sanitizer.default')) {
            throw new \RuntimeException('ux_editor.html.sanitize_required is true but no "html_sanitizer.sanitizer.default" service is registered. Install symfony/html-sanitizer or set sanitize_required: false.');
        }
    }
}
```

Update `UXEditorBundle.php` — add `build()`:
```php
public function build(\Symfony\Component\DependencyInjection\ContainerBuilder $container): void
{
    parent::build($container);
    $container->addCompilerPass(new \Symfony\UX\Editor\DependencyInjection\Compiler\AssertSanitizerPass());
}
```

- [ ] **Step 4: Run — expect 3 pass + existing extension tests still green**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/DependencyInjection/Compiler/AssertSanitizerPass.php src/Editor/src/UXEditorBundle.php src/Editor/tests/DependencyInjection/AssertSanitizerPassTest.php
git commit -m "feat(editor): boot-time check for required HtmlSanitizer"
```

---

## Task 68 — `bin/console debug:ux-editor`

**Files:**
- Create: `src/Editor/src/Command/DebugEditorCommand.php`
- Test: `src/Editor/tests/Command/DebugEditorCommandTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\Command;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Console\Tester\CommandTester;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Command\DebugEditorCommand;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

final class DebugEditorCommandTest extends TestCase
{
    public function testListsSections(): void
    {
        $cmd = new DebugEditorCommand(new BridgeRegistry([]), new PresetRegistry([]), new ContentConverterRegistry([]), new UploadHandlerRegistry([]));
        $tester = new CommandTester($cmd);
        $tester->execute([]);
        $out = $tester->getDisplay();
        self::assertStringContainsString('Bridges',            $out);
        self::assertStringContainsString('Presets',            $out);
        self::assertStringContainsString('Content converters', $out);
        self::assertStringContainsString('Upload handlers',    $out);
        self::assertSame(0, $tester->getStatusCode());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;
use Symfony\UX\Editor\Bridge\BridgeRegistry;
use Symfony\UX\Editor\Config\Preset\PresetRegistry;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

#[AsCommand(name: 'debug:ux-editor', description: 'List registered ux-editor bridges, presets, converters, upload handlers')]
final class DebugEditorCommand extends Command
{
    public function __construct(
        private readonly BridgeRegistry $bridges,
        private readonly PresetRegistry $presets,
        private readonly ContentConverterRegistry $converters,
        private readonly UploadHandlerRegistry $uploadHandlers,
    ) { parent::__construct(); }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $io->section('Bridges');
        $rows = [];
        foreach ($this->bridges->all() as $id => $b) {
            $cap = $b->getCapabilities();
            $rows[] = [$id, $b->getControllerName(), implode(',', $cap->supportedFormats)];
        }
        $io->table(['ID', 'Controller', 'Formats'], $rows);

        $io->section('Presets');
        $io->listing(array_keys($this->presets->all()) ?: ['(none)']);

        $io->section('Content converters');
        $pairs = $this->converters->pairs();
        $io->listing($pairs === [] ? ['(none)'] : array_map(fn(array $p) => sprintf('%s -> %s', $p['from'], $p['to']), $pairs));

        $io->section('Upload handlers');
        $io->listing(array_keys($this->uploadHandlers->all()) ?: ['(none)']);

        return Command::SUCCESS;
    }
}
```

> This command calls `ContentConverterRegistry::pairs()` and `UploadHandlerRegistry::all()`. Both are added in Task 69 (preceding this test run will fail until then). Implementation order within this task: Task 69 first if executing strictly, OR add `pairs()`/`all()` here alongside command implementation. Recommended path: combine Tasks 68 + 69 into one PR if implementing sequentially.

- [ ] **Step 4: Run — expect 1 pass** (after Task 69 lands; if running 68 alone, stub `pairs()`/`all()` first).

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Command/DebugEditorCommand.php src/Editor/tests/Command/DebugEditorCommandTest.php
git commit -m "feat(editor): add debug:ux-editor console command"
```

---

## Task 69 — `pairs()` on converter registry + `all()` on upload handler registry

**Files:**
- Modify: `src/Editor/src/Content/Converter/ContentConverterRegistry.php`
- Modify: `src/Editor/src/Upload/UploadHandlerRegistry.php`
- Test: `src/Editor/tests/Content/Converter/ContentConverterRegistryPairsTest.php`
- Test: `src/Editor/tests/Upload/UploadHandlerRegistryAllTest.php`

- [ ] **Step 1: Failing tests**

```php
<?php
namespace Symfony\UX\Editor\Tests\Content\Converter;

use PHPUnit\Framework\TestCase;
use Symfony\UX\Editor\Content\Converter\ContentConverterInterface;
use Symfony\UX\Editor\Content\Converter\ContentConverterRegistry;
use Symfony\UX\Editor\Content\EditorContentInterface;

final class ContentConverterRegistryPairsTest extends TestCase
{
    public function testPairsListsRegistered(): void
    {
        $conv = new class implements ContentConverterInterface {
            public function getFrom(): string { return 'a'; }
            public function getTo(): string   { return 'b'; }
            public function convert(EditorContentInterface $c): EditorContentInterface { return $c; }
        };
        self::assertSame([['from' => 'a', 'to' => 'b']], (new ContentConverterRegistry([$conv]))->pairs());
        self::assertSame([], (new ContentConverterRegistry([]))->pairs());
    }
}
```

```php
<?php
namespace Symfony\UX\Editor\Tests\Upload;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\UX\Editor\Upload\EditorUploadHandlerInterface;
use Symfony\UX\Editor\Upload\UploadHandlerRegistry;

final class UploadHandlerRegistryAllTest extends TestCase
{
    public function testAllListsRegistered(): void
    {
        $h = new class implements EditorUploadHandlerInterface {
            public function handle(UploadedFile $f, array $c = []): array { return ['url' => '/x', 'size' => 1]; }
        };
        self::assertSame(['default'], array_keys((new UploadHandlerRegistry(['default' => $h]))->all()));
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

Add to `ContentConverterRegistry`:
```php
/** @return list<array{from:string,to:string}> */
public function pairs(): array
{
    $out = [];
    foreach (array_keys($this->byPair) as $key) {
        [$from, $to] = explode('::', $key, 2);
        $out[] = ['from' => $from, 'to' => $to];
    }
    return $out;
}
```

Add to `UploadHandlerRegistry`:
```php
/** @return array<string, EditorUploadHandlerInterface> */
public function all(): array { return $this->handlers; }
```

- [ ] **Step 4: Run — expect 2 pass + Task 68 test now also pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/Content/Converter/ContentConverterRegistry.php src/Editor/src/Upload/UploadHandlerRegistry.php src/Editor/tests/Content/Converter/ContentConverterRegistryPairsTest.php src/Editor/tests/Upload/UploadHandlerRegistryAllTest.php
git commit -m "feat(editor): expose pairs/all on registries"
```

---

## Task 70 — WebProfiler data collector

**Files:**
- Create: `src/Editor/src/DataCollector/UXEditorDataCollector.php`
- Test: `src/Editor/tests/DataCollector/UXEditorDataCollectorTest.php`

- [ ] **Step 1: Failing test**

```php
<?php
namespace Symfony\UX\Editor\Tests\DataCollector;

use PHPUnit\Framework\TestCase;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\UX\Editor\DataCollector\UXEditorDataCollector;

final class UXEditorDataCollectorTest extends TestCase
{
    public function testRecordsBridgeUseAndWarnings(): void
    {
        $c = new UXEditorDataCollector();
        $c->recordBridgeUse('ckeditor', 'html');
        $c->recordCapabilityWarning('Bridge x lacks toolbar');
        $c->collect(new Request(), new Response());
        self::assertSame(1, $c->getBridgeUseCount());
        self::assertSame(['ckeditor' => ['html' => 1]], $c->getBridges());
        self::assertSame(['Bridge x lacks toolbar'], $c->getWarnings());
        self::assertSame('ux_editor', $c->getName());
    }

    public function testReset(): void
    {
        $c = new UXEditorDataCollector();
        $c->recordBridgeUse('q', 'html');
        $c->reset();
        self::assertSame(0, $c->getBridgeUseCount());
    }
}
```

- [ ] **Step 2: Run — expect fail**

- [ ] **Step 3: Implementation**

```php
<?php
namespace Symfony\UX\Editor\DataCollector;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\DataCollector\DataCollector;

final class UXEditorDataCollector extends DataCollector
{
    public function __construct() { $this->reset(); }

    public function recordBridgeUse(string $bridgeId, string $format): void
    {
        $this->data['bridges'][$bridgeId][$format] = ($this->data['bridges'][$bridgeId][$format] ?? 0) + 1;
        $this->data['count']++;
    }

    public function recordCapabilityWarning(string $message): void { $this->data['warnings'][] = $message; }

    public function collect(Request $request, Response $response, ?\Throwable $exception = null): void {}

    public function reset(): void { $this->data = ['bridges' => [], 'warnings' => [], 'count' => 0]; }

    public function getName(): string { return 'ux_editor'; }
    public function getBridges(): array { return $this->data['bridges'] ?? []; }
    public function getWarnings(): array { return $this->data['warnings'] ?? []; }
    public function getBridgeUseCount(): int { return $this->data['count'] ?? 0; }
}
```

- [ ] **Step 4: Run — expect 2 pass**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/src/DataCollector/UXEditorDataCollector.php src/Editor/tests/DataCollector/UXEditorDataCollectorTest.php
git commit -m "feat(editor): add UXEditorDataCollector"
```

---

## Task 71 — Register debug command + data collector + collector template

**Files:**
- Modify: `src/Editor/config/services.php`
- Create: `src/Editor/templates/Collector/template.html.twig`

- [ ] **Step 1: Edit `services.php` — append** (after existing closures):

```php
use Symfony\UX\Editor\Command\DebugEditorCommand;
use Symfony\UX\Editor\DataCollector\UXEditorDataCollector;

$s->set(DebugEditorCommand::class)->tag('console.command');

$s->set(UXEditorDataCollector::class)
    ->tag('data_collector', ['id' => 'ux_editor', 'template' => '@UXEditor/Collector/template.html.twig']);
```

- [ ] **Step 2: Create `templates/Collector/template.html.twig`**

```twig
{% extends '@WebProfiler/Profiler/layout_data_collector.html.twig' %}

{% block toolbar %}
    {% set icon %}<span class="sf-toolbar-value">{{ collector.bridgeUseCount }}</span>{% endset %}
    {% set text %}
        <div class="sf-toolbar-info-piece"><b>UX Editor</b><span>{{ collector.bridgeUseCount }} forms</span></div>
    {% endset %}
    {{ include('@WebProfiler/Profiler/toolbar_item.html.twig', { 'link': false }) }}
{% endblock %}

{% block menu %}<span class="label">UX Editor</span>{% endblock %}

{% block panel %}
    <h2>Bridges in use</h2>
    <table>
        {% for bridge, formats in collector.bridges %}
            {% for fmt, count in formats %}
                <tr><td>{{ bridge }}</td><td>{{ fmt }}</td><td>{{ count }}</td></tr>
            {% endfor %}
        {% else %}
            <tr><td colspan="3">(none)</td></tr>
        {% endfor %}
    </table>
    <h2>Capability warnings</h2>
    <ul>{% for w in collector.warnings %}<li>{{ w }}</li>{% else %}<li>(none)</li>{% endfor %}</ul>
{% endblock %}
```

- [ ] **Step 3: Run extension test (Task 30)** — expect green.

- [ ] **Step 4: Run** — `vendor/bin/phpunit tests/DependencyInjection/`. Expected green.

- [ ] **Step 5: Commit**

```bash
git add src/Editor/config/services.php src/Editor/templates/Collector/template.html.twig
git commit -m "feat(editor): register debug command + data collector template"
```

---

## Task 72 — `assets/controllers.json`

**Files:**
- Create: `src/Editor/assets/controllers.json`

- [ ] **Step 1: Write file**

```json
{
    "controllers": {
        "@symfony/ux-editor": {
            "controller": {
                "enabled": false,
                "fetch": "lazy"
            },
            "format--wysiwyg": {
                "enabled": false,
                "fetch": "lazy"
            },
            "format--block": {
                "enabled": false,
                "fetch": "lazy"
            },
            "format--page": {
                "enabled": false,
                "fetch": "lazy"
            }
        }
    },
    "entrypoints": []
}
```

> All Tier 1 abstracts are `enabled: false`. Tier 2 bridges register concrete controllers in their own `controllers.json`.

- [ ] **Step 2: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('src/Editor/assets/controllers.json', 'utf8'))" && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/assets/controllers.json
git commit -m "feat(editor): add controllers.json (Tier 1 abstracts disabled by default)"
```

---

## Task 73 — Sphinx docs skeleton

**Files:**
- Create: `src/Editor/doc/index.rst`

- [ ] **Step 1: Write file**

```rst
Symfony UX Editor
=================

`symfony/ux-editor`_ provides one Symfony Form field, ``EditorType``, that integrates
multiple content-authoring editors (WYSIWYG, block, page builder) through a single API.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/ux-editor

Then install a bridge package for the editor you want to use, for example:

.. code-block:: terminal

    $ composer require symfony/ux-editor-editorjs
    $ composer require symfony/ux-editor-ckeditor
    $ composer require symfony/ux-editor-grapesjs

Quick start
-----------

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
                    common: new CommonOptions(toolbar: ['bold','italic','link'], placeholder: 'Write…'),
                ),
            ]);
        }
    }

Architecture
------------

Three-tier stack:

* **Tier 0 — Core**: ``EditorType``, value objects (``HtmlContent`` / ``BlockContent`` / ``PageContent``),
  Doctrine custom types, upload pipeline, ``LiveEditor`` trait, Twig ``ux_editor_render`` function.
* **Tier 1 — Format abstracts**: ``AbstractWysiwygBridge`` / ``AbstractBlockBridge`` /
  ``AbstractPageBuilderBridge`` plus matching configs, transformers, controllers, capability factories.
  Shipped inside ``symfony/ux-editor``.
* **Tier 2 — Specific bridges**: CKEditor, Quill, TinyMCE, TipTap, EditorJS, BlockNote, GrapesJS, VvvebJS.
  Each ships as its own composer + npm sub-package.

See ``docs/superpowers/specs/2026-05-19-ux-editor-design.md`` in the source repository for the full design specification.

.. _`symfony/ux-editor`: https://github.com/symfony/ux
```

- [ ] **Step 2: Verify non-empty**

```bash
test -s src/Editor/doc/index.rst && echo OK
```

- [ ] **Step 3-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/doc/index.rst
git commit -m "docs(editor): add Sphinx index.rst"
```

---

## Task 74 — Coverage gate: PHPUnit ≥ 90% Tier 0/1

- [ ] **Step 1: Run with coverage**

```bash
cd src/Editor && XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text --coverage-clover=coverage.xml
```

Expected: line coverage ≥ 90% across `src/Content/`, `src/Config/`, `src/Bridge/`, `src/Form/`, `src/Doctrine/`, `src/Upload/`, `src/Live/`, `src/Twig/`, `src/Exception/`, `src/DependencyInjection/`, `src/DataCollector/`, `src/Command/`.

- [ ] **Step 2: If under 90% — add focused tests**

For each file under threshold, add one test exercising the missing branch. Commit per added test:

```bash
git add src/Editor/tests/...
git commit -m "test(editor): cover <ClassName> <branch>"
```

- [ ] **Step 3: Verify gate**

Re-run Step 1 until threshold met.

- [ ] **Step 4: Ignore coverage artifact**

```bash
grep -q '^coverage.xml$' src/Editor/.gitignore || echo coverage.xml >> src/Editor/.gitignore
```

- [ ] **Step 5: Commit `.gitignore`** (if changed)

```bash
git add src/Editor/.gitignore
git commit -m "chore(editor): ignore coverage.xml"
```

---

## Task 75 — Coverage gate: Vitest ≥ 85%

- [ ] **Step 1: Run with coverage**

```bash
cd src/Editor && npx vitest run --coverage
```

Expected: ≥ 85% line coverage across `assets/src/controller.ts`, `assets/src/format/*.ts`, `assets/src/upload/*.ts`, `assets/src/live/*.ts`, `assets/src/content/*.ts`.

- [ ] **Step 2-3: If under 85% — add focused tests + re-run**

Same iteration pattern as Task 74.

- [ ] **Step 4-5: N/A**

---

## Task 76 — `composer audit` + `npm audit` clean

- [ ] **Step 1: PHP audit**

```bash
cd src/Editor && composer audit
```

Expected: zero advisories.

- [ ] **Step 2: NPM audit**

```bash
cd src/Editor && npm audit --omit=dev
```

Expected: zero advisories.

- [ ] **Step 3: If advisories — bump pinned versions**

Update `composer.json` / `package.json` constraints, re-install, re-run audits, commit per fix:

```bash
git add src/Editor/composer.json src/Editor/package.json
git commit -m "chore(editor): bump <dep> to address <CVE>"
```

- [ ] **Step 4-5: N/A**

---

## Task 77 — Finalize `CHANGELOG.md`

**Files:**
- Modify: `src/Editor/CHANGELOG.md`

- [ ] **Step 1: Replace content**

```markdown
# CHANGELOG

## 0.1.0 — 2026-05-19

Initial release of `symfony/ux-editor` core package.

### Added

- Tier 0 core: `EditorType`, `EditorContentInterface` + polymorphic value objects (`HtmlContent`, `BlockContent`, `PageContent`),
  `EditorContentFormat` enum, `BridgeInterface` + `BridgeRegistry`, `EditorConfigInterface` + `AbstractEditorConfig` + `CommonOptions` +
  `BridgeCapabilities`, preset registry, content converter registry.
- Tier 1 format abstracts: `AbstractWysiwygBridge` + `AbstractWysiwygConfig` + `AbstractWysiwygTransformer` + `WysiwygCapabilities`
  (same shape for Block and Page families).
- Doctrine custom types: `editor_html`, `editor_blocks`, `editor_page`.
- Upload pipeline: `EditorUploadController`, `SignedUploadUrlGenerator`, `EditorUploadHandlerInterface`,
  `DefaultLocalUploadHandler`, `UploadHandlerRegistry`.
- Twig: `ux_editor_render` function (HTML sanitized, blocks via registry, page in sandboxed iframe).
- LiveComponent: `LiveEditor` trait (debounced `saveDraft` + dirty/lastSavedAt tracking).
- JS Tier 0/1: `AbstractEditorController` + format-specific abstract controllers, `SignedUploadClient`, `setupAutosave`, content mirrors.
- Bundle config: `ux_editor.html.sanitize_required`, `ux_editor.upload.default_profile`, `ux_editor.upload.ttl_seconds`.
- Compile pass `AssertSanitizerPass` (boot-time check for required HTML sanitizer).
- Console command `debug:ux-editor`.
- WebProfiler data collector `ux_editor`.

### Notes

This release ships abstractions only. Specific editor bridges (CKEditor, EditorJS, GrapesJS) ship as separate composer + npm sub-packages in follow-up releases.
```

- [ ] **Step 2-4: N/A**

- [ ] **Step 5: Commit**

```bash
git add src/Editor/CHANGELOG.md
git commit -m "docs(editor): finalize CHANGELOG 0.1.0"
```

---

## Task 78 — Annotated tag `editor/v0.1.0` (local only, no push)

- [ ] **Step 1: Verify clean working tree**

```bash
git status
```

Expected: clean.

- [ ] **Step 2: Create annotated tag**

```bash
git tag -a editor/v0.1.0 -m "symfony/ux-editor 0.1.0 — Tier 0 + Tier 1 abstracts"
```

> Do not push. Release engineering decides when to push tags upstream; splitsh tooling tags split repos separately.

- [ ] **Step 3: Verify**

```bash
git tag -l 'editor/v0.1.0'
```

Expected: `editor/v0.1.0`.

- [ ] **Step 4-5: N/A**

---

## Final checkpoint after Task 78

- [ ] **Full PHPUnit + Vitest**

```bash
cd src/Editor && vendor/bin/phpunit && npx vitest run
```

Expected: both green.

- [ ] **Final coverage**

```bash
XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-text
npx vitest run --coverage
```

Expected: PHPUnit ≥ 90% Tier 0/1 lines, Vitest ≥ 85% lines.

- [ ] **Audit**

```bash
composer audit && npm audit --omit=dev
```

Expected: zero advisories.

---

## Self-review for Tasks 51-78

**1. Spec coverage:**
- §3 Tier 1 Page transformer + bridge → Tasks 51-52.
- §3 JS Tier 0/1 controllers + utilities → Tasks 53-65.
- §3/§7 Bundle wiring (config schema, compile pass, command, collector, controllers.json) → Tasks 66-72.
- §13/release (docs, coverage gates, audit, tag) → Tasks 73-78.

**2. Placeholder scan:**
- Task 65 step 4 is conditional ("if `tsc` reports errors") — quality gate, not a TODO.
- Task 68 has a note about ordering with Task 69; combining the two when implementing sequentially is explicit, not handwave.
- Tasks 74-75 have iteration steps ("add tests until threshold met") — bounded quality gate, every iteration produces a test + commit.
- Task 76 step 3 is conditional bump path — bounded fix.
- No `TBD`/`FIXME`/`handle errors`/etc.

**3. Type consistency:**
- `AbstractEditorController` shape stable across Tasks 53-57, 61-64. Abstract methods `createEditor`, `serialize`, `destroyEditor`, `hotReloadable`, `applyConfig` referenced uniformly.
- `AbstractWysiwygController.serialize` returns string; `AbstractBlockController.serialize` returns object (awaited); `AbstractPageBuilderController.serialize` returns bundle object. Each consistent with PHP-side `HtmlContent`/`BlockContent`/`PageContent` `getRaw()` types.
- `SignedUploadClient` error shape `{status, code, message}` consistent with `EditorUploadController` JSON response shape (Task 29 in tasks-7-30).
- `Configuration` parameter names (`ux_editor.html.sanitize_required`, `ux_editor.upload.default_profile`, `ux_editor.upload.ttl_seconds`) consistent across Tasks 66, 67, services.php update.
- `UploadHandlerRegistry::all()` and `ContentConverterRegistry::pairs()` follow existing `BridgeRegistry::all()` (Task 18) and `PresetRegistry::all()` (Task 13) patterns.

No inline fixes needed.

---

## Plan 1 — complete

Plan 1 (Core + Tier 1) fully expanded across:
- `2026-05-19-ux-editor-core.md` (Tasks 1-6 + 7-78 summary index)
- `2026-05-19-ux-editor-core-tasks-7-30.md` (exceptions, config, bridge, EditorType, Doctrine, upload, bundle wiring)
- `2026-05-19-ux-editor-core-tasks-31-50.md` (Live trait, Twig, Tier 1 Wysiwyg/Block/Page PHP abstracts)
- `2026-05-19-ux-editor-core-tasks-51-78.md` (Page transformer/bridge, JS Tier 0/1, bundle finalization, docs, coverage, audit, tag) — this file

Plans 2-5 (deferred until Plan 1 lands):
- Plan 2: `2026-05-19-ux-editor-bridge-editorjs.md` — first Tier 2 bridge (EditorJS)
- Plan 3: `2026-05-19-ux-editor-bridge-ckeditor.md` — CKEditor bridge
- Plan 4: `2026-05-19-ux-editor-bridge-grapesjs.md` — GrapesJS bridge
- Plan 5: `2026-05-19-ux-editor-demo.md` — `ux.symfony.com` demo + LiveComponent showcase

## Execution handoff

Plan saved across four files in `docs/superpowers/plans/`. Two execution options:

1. **Subagent-Driven (recommended)** — `superpowers:subagent-driven-development` dispatches one subagent per task. Each subagent reads the parent overview + relevant tasks file. Two-stage review between tasks.

2. **Inline Execution** — `superpowers:executing-plans` walks the four files in this session, batched with checkpoints (after Tasks 6, 15, 30, 50, 65, 78).

Which approach?
