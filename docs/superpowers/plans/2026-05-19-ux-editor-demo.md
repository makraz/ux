# UX Editor — Demo App + LiveComponent Showcase Implementation Plan (Plan 5 / 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the `ux.symfony.com` demo app — four routes (`/editor/{wysiwyg,block,page,live}`) showcasing each v1 bridge plus a LiveComponent integration showcase (hot config reload + autosave). Also: make the Playwright specs deferred in Plans 2-4 runnable, expose `__demoCkEditor` / `__demoGrapes` runtime handles, register EditorJS tool classes on `window.UXEditorJSTools`, aggregate per-bridge Playwright projects into the root config.

**Architecture:** Adds routes + controllers + Form types + templates + JS entrypoint to `ux.symfony.com/` (existing demo Symfony app). No new packages — app-level wiring of the four packages from Plans 1-4.

**Tech Stack:** Symfony 7.4/8.0, AssetMapper, `symfony/ux-live-component`, `symfony/ux-twig-component`, four packages from Plans 1-4.

**Depends on:** Plans 1-4 fully merged.

**Spec:** [`docs/superpowers/specs/2026-05-19-ux-editor-design.md`](../specs/2026-05-19-ux-editor-design.md)
**Roadmap:** [`ROADMAP-UX-EDITOR.md`](../../../ROADMAP-UX-EDITOR.md) — Plan 5 closes v1.0 by enabling Playwright E2E across all v1 bridges.

---

## File structure

```
ux.symfony.com/
├── composer.json                           # MODIFY: add 4 ux-editor packages
├── importmap.php                           # MODIFY: add @symfony/ux-editor + 3 bridges + upstream libs
├── assets/
│   ├── app.js                              # MODIFY: register EditorJS tools on window.UXEditorJSTools
│   └── controllers.json                    # MODIFY: enable ux-editor controllers
├── config/
│   └── routes/editor.yaml                  # CREATE
├── src/
│   ├── Controller/Editor/{Wysiwyg,Block,Page,Live}DemoController.php
│   ├── Entity/Editor/{DemoArticle,DemoNote,DemoLanding}.php
│   ├── Form/Editor/{DemoArticleType,DemoNoteType,DemoLandingType}.php
│   └── Twig/Components/Editor/LiveEditorDemo.php
├── templates/editor/
│   ├── _layout.html.twig
│   ├── wysiwyg.html.twig
│   ├── block.html.twig
│   ├── page.html.twig
│   └── live.html.twig
├── templates/components/Editor/LiveEditorDemo.html.twig
└── migrations/Version<TIMESTAMP>_AddEditorDemoTables.php
```

Order: composer install → entity + migration → form types → controllers → templates → live component → asset wiring → Playwright orchestration → run E2E → docs → tag.

---

## Task 1 — Install the four packages

**Files:**
- Modify: `ux.symfony.com/composer.json`
- Modify: `ux.symfony.com/importmap.php`

- [ ] **Step 1: Edit `composer.json` — append to `require`**

```json
"symfony/ux-editor": "@dev",
"symfony/ux-editor-editorjs": "@dev",
"symfony/ux-editor-ckeditor": "@dev",
"symfony/ux-editor-grapesjs": "@dev"
```

Add `repositories` block (if absent) so composer resolves them locally:

```json
"repositories": [
    {"type": "path", "url": "../src/Editor", "options": {"symlink": true}},
    {"type": "path", "url": "../src/Editor/src/Bridge/EditorJS", "options": {"symlink": true}},
    {"type": "path", "url": "../src/Editor/src/Bridge/CKEditor", "options": {"symlink": true}},
    {"type": "path", "url": "../src/Editor/src/Bridge/GrapesJS", "options": {"symlink": true}}
]
```

- [ ] **Step 2: Install**

```bash
cd ux.symfony.com && composer update --no-interaction \
  symfony/ux-editor symfony/ux-editor-editorjs symfony/ux-editor-ckeditor symfony/ux-editor-grapesjs
```

Expected: 4 packages installed, no errors.

- [ ] **Step 3: Edit `importmap.php` — append**

```php
'@symfony/ux-editor'           => ['version' => '0.1.0'],
'@symfony/ux-editor-editorjs'  => ['version' => '0.1.0'],
'@symfony/ux-editor-ckeditor'  => ['version' => '0.1.0'],
'@symfony/ux-editor-grapesjs'  => ['version' => '0.1.0'],
'@editorjs/editorjs'           => ['version' => '2.30.0'],
'@editorjs/header'             => ['version' => '2.8.1'],
'@editorjs/list'               => ['version' => '1.10.0'],
'@editorjs/image'              => ['version' => '2.9.0'],
'@editorjs/quote'              => ['version' => '2.6.0'],
'@ckeditor/ckeditor5-build-classic' => ['version' => '41.4.2'],
'grapesjs'                     => ['version' => '0.21.7'],
```

- [ ] **Step 4: Verify importmap resolves**

```bash
php bin/console importmap:audit
```

Expected: zero advisories (or accept upstream advisories with explicit note).

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/composer.json ux.symfony.com/composer.lock ux.symfony.com/importmap.php
git commit -m "feat(demo): install ux-editor + 3 bridges + upstream libs"
```

---

## Task 2 — Demo entities + migration

**Files:**
- Create: `ux.symfony.com/src/Entity/Editor/{DemoArticle,DemoNote,DemoLanding}.php`
- Create: `ux.symfony.com/migrations/Version<TIMESTAMP>_AddEditorDemoTables.php`

- [ ] **Step 1: Write entities**

`DemoArticle.php`:
```php
<?php
namespace App\Entity\Editor;

use Doctrine\ORM\Mapping as ORM;
use Symfony\UX\Editor\Content\HtmlContent;

#[ORM\Entity]
#[ORM\Table(name: 'demo_article')]
class DemoArticle
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 200)]
    public string $title = '';

    #[ORM\Column(type: 'editor_html', nullable: true)]
    public ?HtmlContent $body = null;

    public function getId(): ?int { return $this->id; }
}
```

`DemoNote.php`:
```php
<?php
namespace App\Entity\Editor;

use Doctrine\ORM\Mapping as ORM;
use Symfony\UX\Editor\Content\BlockContent;

#[ORM\Entity]
#[ORM\Table(name: 'demo_note')]
class DemoNote
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 200)]
    public string $title = '';

    #[ORM\Column(type: 'editor_blocks', nullable: true)]
    public ?BlockContent $body = null;

    public function getId(): ?int { return $this->id; }
}
```

`DemoLanding.php`:
```php
<?php
namespace App\Entity\Editor;

use Doctrine\ORM\Mapping as ORM;
use Symfony\UX\Editor\Content\PageContent;

#[ORM\Entity]
#[ORM\Table(name: 'demo_landing')]
class DemoLanding
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 200)]
    public string $title = '';

    #[ORM\Column(type: 'editor_page', nullable: true)]
    public ?PageContent $homepage = null;

    public function getId(): ?int { return $this->id; }
}
```

- [ ] **Step 2: Generate + run migration**

```bash
cd ux.symfony.com
php bin/console make:migration --no-interaction
php bin/console doctrine:migrations:migrate --no-interaction
```

Expected: 3 new tables created.

- [ ] **Step 3: Schema validation**

```bash
php bin/console doctrine:schema:validate
```

Expected: schema in sync.

- [ ] **Step 4: N/A**

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/src/Entity/Editor/ ux.symfony.com/migrations/
git commit -m "feat(demo): add DemoArticle / DemoNote / DemoLanding entities + migration"
```

---

## Task 3 — Form types

**Files:**
- Create: `ux.symfony.com/src/Form/Editor/{DemoArticleType,DemoNoteType,DemoLandingType}.php`

- [ ] **Step 1: Write `DemoArticleType.php`**

```php
<?php
namespace App\Form\Editor;

use App\Entity\Editor\DemoArticle;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Form\EditorType;

final class DemoArticleType extends AbstractType
{
    public function buildForm(FormBuilderInterface $b, array $options): void
    {
        $b
            ->add('title', TextType::class, ['required' => true])
            ->add('body', EditorType::class, [
                'config' => new CKEditorConfig(
                    common: new CommonOptions(
                        toolbar: ['heading', 'bold', 'italic', 'link', 'bulletedList'],
                        placeholder: 'Write your article…',
                    ),
                    licenseKey: 'GPL',
                ),
            ])
            ->add('save', SubmitType::class, ['label' => 'Save', 'attr' => ['data-test-id' => 'submit-button']]);
    }

    public function configureOptions(OptionsResolver $r): void
    {
        $r->setDefaults(['data_class' => DemoArticle::class]);
    }
}
```

- [ ] **Step 2: Write `DemoNoteType.php`**

```php
<?php
namespace App\Form\Editor;

use App\Entity\Editor\DemoNote;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\UX\Editor\Form\EditorType;

final class DemoNoteType extends AbstractType
{
    public function buildForm(FormBuilderInterface $b, array $options): void
    {
        $b
            ->add('title', TextType::class, ['required' => true])
            ->add('body', EditorType::class, ['preset' => 'blog.standard'])
            ->add('save', SubmitType::class, ['label' => 'Save', 'attr' => ['data-test-id' => 'submit-button']]);
    }

    public function configureOptions(OptionsResolver $r): void
    {
        $r->setDefaults(['data_class' => DemoNote::class]);
    }
}
```

- [ ] **Step 3: Write `DemoLandingType.php`**

```php
<?php
namespace App\Form\Editor;

use App\Entity\Editor\DemoLanding;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\UX\Editor\Form\EditorType;

final class DemoLandingType extends AbstractType
{
    public function buildForm(FormBuilderInterface $b, array $options): void
    {
        $b
            ->add('title', TextType::class, ['required' => true])
            ->add('homepage', EditorType::class, ['preset' => 'page_builder.landing'])
            ->add('save', SubmitType::class, ['label' => 'Save', 'attr' => ['data-test-id' => 'submit-button']]);
    }

    public function configureOptions(OptionsResolver $r): void
    {
        $r->setDefaults(['data_class' => DemoLanding::class]);
    }
}
```

- [ ] **Step 4: Lint**

```bash
cd ux.symfony.com && php bin/console lint:container
```

Expected: no errors related to the form types.

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/src/Form/Editor/
git commit -m "feat(demo): add editor demo form types"
```

---

## Task 4 — Demo controllers + routes

**Files:**
- Create: `ux.symfony.com/src/Controller/Editor/{Wysiwyg,Block,Page,Live}DemoController.php`
- Create: `ux.symfony.com/config/routes/editor.yaml`

- [ ] **Step 1: Write `WysiwygDemoController.php`**

```php
<?php
namespace App\Controller\Editor;

use App\Entity\Editor\DemoArticle;
use App\Form\Editor\DemoArticleType;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class WysiwygDemoController extends AbstractController
{
    #[Route('/editor/wysiwyg', name: 'demo_editor_wysiwyg', methods: ['GET', 'POST'])]
    public function __invoke(Request $request): Response
    {
        $article = new DemoArticle();
        $form = $this->createForm(DemoArticleType::class, $article);
        $form->handleRequest($request);

        $entityDump = null;
        if ($form->isSubmitted() && $form->isValid()) {
            $entityDump = json_encode([
                'title' => $article->title,
                'body'  => $article->body?->html,
                'meta'  => $article->body?->getMetadata(),
            ], \JSON_PRETTY_PRINT | \JSON_THROW_ON_ERROR);
        }

        return $this->render('editor/wysiwyg.html.twig', ['form' => $form, 'entityDump' => $entityDump]);
    }
}
```

- [ ] **Step 2: Write `BlockDemoController.php`**

```php
<?php
namespace App\Controller\Editor;

use App\Entity\Editor\DemoNote;
use App\Form\Editor\DemoNoteType;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class BlockDemoController extends AbstractController
{
    #[Route('/editor/block', name: 'demo_editor_block', methods: ['GET', 'POST'])]
    public function __invoke(Request $request): Response
    {
        $note = new DemoNote();
        $form = $this->createForm(DemoNoteType::class, $note);
        $form->handleRequest($request);

        $entityDump = null;
        if ($form->isSubmitted() && $form->isValid()) {
            $entityDump = json_encode([
                'title'  => $note->title,
                'blocks' => $note->body?->blocks,
                'meta'   => $note->body?->getMetadata(),
            ], \JSON_PRETTY_PRINT | \JSON_THROW_ON_ERROR);
        }

        return $this->render('editor/block.html.twig', ['form' => $form, 'entityDump' => $entityDump]);
    }
}
```

- [ ] **Step 3: Write `PageDemoController.php`**

```php
<?php
namespace App\Controller\Editor;

use App\Entity\Editor\DemoLanding;
use App\Form\Editor\DemoLandingType;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class PageDemoController extends AbstractController
{
    #[Route('/editor/page', name: 'demo_editor_page', methods: ['GET', 'POST'])]
    public function __invoke(Request $request): Response
    {
        $landing = new DemoLanding();
        $form = $this->createForm(DemoLandingType::class, $landing);
        $form->handleRequest($request);

        $entityDump = null;
        if ($form->isSubmitted() && $form->isValid()) {
            $entityDump = json_encode([
                'title' => $landing->title,
                'page'  => $landing->homepage ? [
                    'html'       => $landing->homepage->html,
                    'css'        => $landing->homepage->css,
                    'assets'     => $landing->homepage->assets,
                    'components' => $landing->homepage->components,
                ] : null,
                'meta' => $landing->homepage?->getMetadata(),
            ], \JSON_PRETTY_PRINT | \JSON_THROW_ON_ERROR);
        }

        return $this->render('editor/page.html.twig', ['form' => $form, 'entityDump' => $entityDump]);
    }
}
```

- [ ] **Step 4: Write `LiveDemoController.php`**

```php
<?php
namespace App\Controller\Editor;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class LiveDemoController extends AbstractController
{
    #[Route('/editor/live', name: 'demo_editor_live', methods: ['GET'])]
    public function __invoke(): Response
    {
        return $this->render('editor/live.html.twig');
    }
}
```

- [ ] **Step 5: Write `config/routes/editor.yaml`**

```yaml
demo_editor:
    resource:
        path: ../src/Controller/Editor/
        namespace: App\Controller\Editor
    type: attribute
```

- [ ] **Step 6: Verify routes**

```bash
cd ux.symfony.com && php bin/console debug:router | grep demo_editor
```

Expected: 4 routes (`demo_editor_wysiwyg`, `demo_editor_block`, `demo_editor_page`, `demo_editor_live`).

- [ ] **Step 7: Commit**

```bash
git add ux.symfony.com/src/Controller/Editor/ ux.symfony.com/config/routes/editor.yaml
git commit -m "feat(demo): add 4 editor demo controllers + routes"
```

---

## Task 5 — Templates

**Files:**
- Create: `ux.symfony.com/templates/editor/{_layout,wysiwyg,block,page,live}.html.twig`

- [ ] **Step 1: Write `_layout.html.twig`**

```twig
{% extends 'base.html.twig' %}

{% block title %}{{ pageTitle ?? 'UX Editor demo' }}{% endblock %}

{% block body %}
    <main class="container">
        <h1>{{ pageTitle ?? 'UX Editor demo' }}</h1>
        {% block editor_body %}{% endblock %}

        {% if entityDump is defined and entityDump is not null %}
            <h2>Submitted entity</h2>
            <pre data-test-id="entity-dump">{{ entityDump }}</pre>
        {% endif %}
    </main>
{% endblock %}
```

- [ ] **Step 2: Write `wysiwyg.html.twig`**

```twig
{% extends 'editor/_layout.html.twig' %}

{% set pageTitle = 'WYSIWYG demo — CKEditor' %}

{% block editor_body %}
    {{ form_start(form) }}
        {{ form_row(form.title) }}
        {{ form_row(form.body) }}
        {{ form_row(form.save) }}
    {{ form_end(form) }}

    <p>
        <a href="{{ path('demo_editor_block') }}">Block demo</a> ·
        <a href="{{ path('demo_editor_page') }}">Page demo</a> ·
        <a href="{{ path('demo_editor_live') }}">Live demo</a>
    </p>

    <script type="module">
        document.addEventListener('ux:editor:connect', (e) => {
            if (e.detail?.bridgeId === 'ckeditor') {
                window.__demoCkEditor = e.detail.instance;
            }
        });
    </script>
{% endblock %}
```

- [ ] **Step 3: Write `block.html.twig`**

```twig
{% extends 'editor/_layout.html.twig' %}

{% set pageTitle = 'Block demo — EditorJS' %}

{% block editor_body %}
    {{ form_start(form) }}
        {{ form_row(form.title) }}
        <div data-test-id="editorjs-canvas">
            {{ form_row(form.body) }}
        </div>
        {{ form_row(form.save) }}
    {{ form_end(form) }}

    <p>
        <a href="{{ path('demo_editor_wysiwyg') }}">WYSIWYG demo</a> ·
        <a href="{{ path('demo_editor_page') }}">Page demo</a> ·
        <a href="{{ path('demo_editor_live') }}">Live demo</a>
    </p>

    <script type="module">
        document.addEventListener('ux:editor:connect', (e) => {
            if (e.detail?.bridgeId === 'editorjs') {
                const el = document.createElement('span');
                el.dataset.testId = 'editor-state';
                el.textContent = 'connected';
                document.body.append(el);
            }
        });
    </script>
{% endblock %}
```

- [ ] **Step 4: Write `page.html.twig`**

```twig
{% extends 'editor/_layout.html.twig' %}

{% set pageTitle = 'Page-builder demo — GrapesJS' %}

{% block editor_body %}
    {{ form_start(form) }}
        {{ form_row(form.title) }}
        {{ form_row(form.homepage) }}
        {{ form_row(form.save) }}
    {{ form_end(form) }}

    <p>
        <a href="{{ path('demo_editor_wysiwyg') }}">WYSIWYG demo</a> ·
        <a href="{{ path('demo_editor_block') }}">Block demo</a> ·
        <a href="{{ path('demo_editor_live') }}">Live demo</a>
    </p>

    <script type="module">
        document.addEventListener('ux:editor:connect', (e) => {
            if (e.detail?.bridgeId === 'grapesjs') {
                window.__demoGrapes = e.detail.instance;
            }
        });
    </script>
{% endblock %}
```

- [ ] **Step 5: Write `live.html.twig`**

```twig
{% extends 'editor/_layout.html.twig' %}

{% set pageTitle = 'LiveComponent demo — hot config + autosave' %}

{% block editor_body %}
    <twig:Editor:LiveEditorDemo />

    <p>
        <a href="{{ path('demo_editor_wysiwyg') }}">WYSIWYG demo</a> ·
        <a href="{{ path('demo_editor_block') }}">Block demo</a> ·
        <a href="{{ path('demo_editor_page') }}">Page demo</a>
    </p>
{% endblock %}
```

- [ ] **Step 6: Smoke-test all four routes**

```bash
cd ux.symfony.com && symfony server:start --daemon
curl -sf http://127.0.0.1:8000/editor/wysiwyg > /dev/null && echo wysiwyg OK
curl -sf http://127.0.0.1:8000/editor/block   > /dev/null && echo block OK
curl -sf http://127.0.0.1:8000/editor/page    > /dev/null && echo page OK
curl -sf http://127.0.0.1:8000/editor/live    > /dev/null && echo live OK
symfony server:stop
```

Expected: 4× `OK`.

- [ ] **Step 7: Commit**

```bash
git add ux.symfony.com/templates/editor/
git commit -m "feat(demo): add 5 editor demo templates"
```

---

## Task 6 — LiveEditorDemo component

**Files:**
- Create: `ux.symfony.com/src/Twig/Components/Editor/LiveEditorDemo.php`
- Create: `ux.symfony.com/templates/components/Editor/LiveEditorDemo.html.twig`

- [ ] **Step 1: Write component class**

```php
<?php
namespace App\Twig\Components\Editor;

use Symfony\UX\Editor\Bridge\CKEditor\Config\CKEditorConfig;
use Symfony\UX\Editor\Config\CommonOptions;
use Symfony\UX\Editor\Config\EditorConfigInterface;
use Symfony\UX\Editor\Live\LiveEditor;
use Symfony\UX\LiveComponent\Attribute\AsLiveComponent;
use Symfony\UX\LiveComponent\Attribute\LiveAction;
use Symfony\UX\LiveComponent\Attribute\LiveProp;
use Symfony\UX\LiveComponent\DefaultActionTrait;

#[AsLiveComponent('Editor:LiveEditorDemo')]
final class LiveEditorDemo
{
    use DefaultActionTrait;
    use LiveEditor;

    #[LiveProp(writable: true)]
    public bool $readOnly = false;

    #[LiveProp(writable: true)]
    public string $bodyDraft = '<p>Edit me…</p>';

    public function __construct()
    {
        // In-memory repo for the demo (host's saveDraft trait expects ->upsert).
        $this->draftRepo = new class { public array $store = []; public function upsert(string $id, string $field, mixed $content): void { $this->store[$id][$field] = $content; } };
    }

    public function getEntityId(): string { return 'demo-1'; }

    public function getConfig(): EditorConfigInterface
    {
        return new CKEditorConfig(
            common: new CommonOptions(
                toolbar: ['bold', 'italic', 'link'],
                readOnly: $this->readOnly,
                placeholder: 'Edit me…',
            ),
            licenseKey: 'GPL',
        );
    }

    #[LiveAction]
    public function toggleReadOnly(): void
    {
        $this->readOnly = !$this->readOnly;
    }
}
```

- [ ] **Step 2: Write template**

```twig
<div {{ attributes }} class="live-editor-demo">
    <label>
        <input type="checkbox" data-test-id="toggle-readonly"
               {{ this.readOnly ? 'checked' : '' }}
               data-action="live#action" data-live-action-param="toggleReadOnly">
        Read-only
    </label>

    <form data-live-ignore-form>
        {# Render an EditorType field driven by this.getConfig(). The form_factory global is
           available when symfony/twig-bundle's FormExtension is enabled (it ships with framework-bundle). #}
        {{ form_widget(form_factory.createNamed('demo_live', \Symfony\UX\Editor\Form\EditorType::class, this.bodyDraft, {
            'config': this.getConfig(),
            'sanitize': false,
        })) }}
    </form>

    <p data-test-id="dirty-state">
        {% if this.isDirty('bodyDraft') %}dirty{% else %}clean{% endif %}
        {% if this.getLastSavedAt('bodyDraft') %}
            · saved at {{ this.getLastSavedAt('bodyDraft')|date('H:i:s') }}
        {% endif %}
    </p>
</div>
```

> Note: `form_factory` is the Twig global exposed by `symfony/twig-bundle`'s `FormExtension`. If your demo app uses a different render path (e.g. passing the rendered widget via a component prop), adapt this template — the assertions only require the `data-test-id` attributes and a mounted CKEditor.

- [ ] **Step 3: Smoke-test live route**

```bash
cd ux.symfony.com && symfony server:start --daemon
curl -sf http://127.0.0.1:8000/editor/live | grep -q 'data-test-id="dirty-state"' && echo OK
symfony server:stop
```

Expected: `OK`.

- [ ] **Step 4: N/A**

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/src/Twig/Components/Editor/ ux.symfony.com/templates/components/Editor/
git commit -m "feat(demo): add LiveEditorDemo component (hot readOnly + dirty/lastSavedAt)"
```

---

## Task 7 — Asset entrypoint (EditorJS tool registry)

**Files:**
- Modify: `ux.symfony.com/assets/app.js`

- [ ] **Step 1: Append to `assets/app.js`**

```js
// Register EditorJS tool classes for the symfony/ux-editor-editorjs bridge controller.
// Bridge resolves names declared in EditorJSConfig::tools against window.UXEditorJSTools.
import Header from '@editorjs/header';
import List   from '@editorjs/list';
import Image  from '@editorjs/image';
import Quote  from '@editorjs/quote';

window.UXEditorJSTools = { Header, List, Image, Quote };
```

- [ ] **Step 2: Refresh importmap (AssetMapper) or rebuild (Encore)**

AssetMapper:
```bash
cd ux.symfony.com && php bin/console importmap:install
```

Encore: `npm run build` (or `npm run dev` while testing).

- [ ] **Step 3: Smoke-test**

```bash
symfony server:start --daemon
curl -sf http://127.0.0.1:8000/editor/block | grep -q 'editorjs' && echo OK
symfony server:stop
```

Expected: `OK`.

- [ ] **Step 4: N/A**

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/assets/app.js ux.symfony.com/importmap.php
git commit -m "feat(demo): register EditorJS tool classes on window.UXEditorJSTools"
```

---

## Task 8 — Confirm controllers.json enables bridge controllers

**Files:**
- Modify (only if needed): `ux.symfony.com/assets/controllers.json`

- [ ] **Step 1: Inspect existing `controllers.json`**

```bash
cat ux.symfony.com/assets/controllers.json
```

- [ ] **Step 2: If the app overrides per-controller flags, set them to `enabled: true`**

```json
{
    "controllers": {
        "@symfony/ux-editor-editorjs": { "editorjs": { "enabled": true } },
        "@symfony/ux-editor-ckeditor": { "ckeditor": { "enabled": true } },
        "@symfony/ux-editor-grapesjs": { "grapesjs": { "enabled": true } }
    }
}
```

If the app's `controllers.json` only references other packages, no edit needed — bridge packages ship `enabled: true` in their own `controllers.json` (Plans 2/3/4).

- [ ] **Step 3: Validate JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('ux.symfony.com/assets/controllers.json', 'utf8'))" && echo OK
```

- [ ] **Step 4: Smoke-list registered controllers**

```bash
cd ux.symfony.com && php bin/console debug:stimulus 2>/dev/null | grep -E 'ux-editor--(ckeditor|editorjs|grapesjs)' || echo "(debug:stimulus not available — skip)"
```

- [ ] **Step 5: Commit (only if file changed)**

```bash
git add ux.symfony.com/assets/controllers.json
git commit -m "feat(demo): ensure ux-editor bridge controllers enabled"
```

---

## Task 9 — Root Playwright config aggregating bridge projects

**Files:**
- Inspect: `playwright.config.base.ts` (root)
- Create: `playwright.config.ts` (root)
- Create: `src/Editor/tests/e2e/live/{hot-reload,autosave}.spec.ts`
- Modify: `package.json` (root) — `test:e2e` script

- [ ] **Step 1: Inspect base config**

```bash
cat playwright.config.base.ts
```

- [ ] **Step 2: Write root `playwright.config.ts`**

```ts
import { defineConfig } from '@playwright/test';
import base from './playwright.config.base';

export default defineConfig({
  ...base,
  webServer: {
    command: 'cd ux.symfony.com && symfony server:start --no-tls --port=8000',
    url: 'http://127.0.0.1:8000',
    reuseExistingServer: !process.env.CI,
    timeout: 60_000,
  },
  use: { ...base.use, baseURL: 'http://127.0.0.1:8000' },
  projects: [
    { name: 'editorjs', testDir: './src/Editor/src/Bridge/EditorJS/tests/e2e' },
    { name: 'ckeditor', testDir: './src/Editor/src/Bridge/CKEditor/tests/e2e' },
    { name: 'grapesjs', testDir: './src/Editor/src/Bridge/GrapesJS/tests/e2e' },
    { name: 'live',     testDir: './src/Editor/tests/e2e/live' },
  ],
});
```

- [ ] **Step 3: Create Live cross-cutting specs**

```bash
mkdir -p src/Editor/tests/e2e/live
```

`src/Editor/tests/e2e/live/hot-reload.spec.ts`:
```ts
import { test, expect } from '@playwright/test';

test('toggling readOnly applies in-place (no remount)', async ({ page }) => {
  await page.goto('/editor/live');
  await expect(page.locator('.ck-editor__editable[contenteditable="true"]')).toBeVisible();
  await page.locator('[data-test-id="toggle-readonly"]').check();
  await expect(page.locator('.ck-editor__editable[contenteditable="false"]')).toBeVisible();
});
```

`src/Editor/tests/e2e/live/autosave.spec.ts`:
```ts
import { test, expect } from '@playwright/test';

test('typing then waiting 800ms produces a saved-at indicator', async ({ page }) => {
  await page.goto('/editor/live');
  await page.locator('.ck-editor__editable').click();
  await page.keyboard.type('autosave probe');
  await page.waitForTimeout(900);
  await expect(page.locator('[data-test-id="dirty-state"]')).toContainText('saved at');
});
```

- [ ] **Step 4: Add `test:e2e` script to root `package.json`**

Append:
```json
"scripts": {
    "test:e2e": "playwright test"
}
```

- [ ] **Step 5: Smoke-list specs**

```bash
npx playwright test --list | head -30
```

Expected: lists specs from all 4 projects (editorjs, ckeditor, grapesjs, live).

- [ ] **Step 6: Commit**

```bash
git add playwright.config.ts package.json src/Editor/tests/e2e/live/
git commit -m "feat(demo): aggregate Playwright projects + add Live hot-reload/autosave specs"
```

---

## Task 10 — Run E2E suites against the demo

- [ ] **Step 1: Install Playwright browsers**

```bash
npx playwright install chromium
```

- [ ] **Step 2: Run editorjs project**

```bash
npx playwright test --project=editorjs
```

Expected: all EditorJS specs (Plan 2 Task 16) pass.

- [ ] **Step 3: Run ckeditor project**

```bash
npx playwright test --project=ckeditor
```

Expected: all CKEditor specs (Plan 3 Task 11) pass.

- [ ] **Step 4: Run grapesjs project**

```bash
npx playwright test --project=grapesjs
```

Expected: all GrapesJS specs (Plan 4 Task 10) pass.

- [ ] **Step 5: Run live project**

```bash
npx playwright test --project=live
```

Expected: hot-reload + autosave specs pass.

- [ ] **Step 6: If a spec fails — fix demo-side wiring**

Bridge code is locked. Failures are demo-side (template selectors, missing data-test-id, importmap version drift). Commit each fix per bridge:

```bash
git add ux.symfony.com/templates/editor/<file>.html.twig
git commit -m "fix(demo): align <bridge> demo selectors with Playwright spec"
```

- [ ] **Step 7: N/A**

---

## Task 11 — List ux-editor on demo home page

**Files:**
- Modify: existing demo home/controller that lists UX packages (inspect first)

- [ ] **Step 1: Locate existing listing**

```bash
grep -rln 'ux-map\|ux-chartjs\|UxPackages' ux.symfony.com/src/Controller ux.symfony.com/templates | head -10
```

- [ ] **Step 2: Add ux-editor entry matching existing shape**

Sample addition (adapt to observed shape):
```php
[
    'name'        => 'ux-editor',
    'title'       => 'UX Editor',
    'description' => 'One Symfony Form field over WYSIWYG, block, and page-builder editors.',
    'route'       => 'demo_editor_wysiwyg',
    'sub_routes'  => [
        ['label' => 'WYSIWYG (CKEditor)', 'route' => 'demo_editor_wysiwyg'],
        ['label' => 'Block (EditorJS)',   'route' => 'demo_editor_block'],
        ['label' => 'Page (GrapesJS)',    'route' => 'demo_editor_page'],
        ['label' => 'Live (autosave)',    'route' => 'demo_editor_live'],
    ],
],
```

- [ ] **Step 3: Smoke-test home page**

```bash
symfony server:start --daemon
curl -sf http://127.0.0.1:8000/ | grep -q 'UX Editor' && echo OK
symfony server:stop
```

Expected: `OK`.

- [ ] **Step 4: N/A**

- [ ] **Step 5: Commit**

```bash
git add ux.symfony.com/src/Controller/ ux.symfony.com/templates/
git commit -m "feat(demo): list ux-editor on demo home page"
```

---

## Task 12 — Final checkpoint + tag

- [ ] **Step 1: Full PHPUnit across all packages**

```bash
cd src/Editor && vendor/bin/phpunit
cd src/Editor/src/Bridge/EditorJS && vendor/bin/phpunit
cd src/Editor/src/Bridge/CKEditor && vendor/bin/phpunit
cd src/Editor/src/Bridge/GrapesJS && vendor/bin/phpunit
```

Expected: all green.

- [ ] **Step 2: Full Vitest**

```bash
cd src/Editor && npx vitest run
cd src/Editor/src/Bridge/EditorJS && npx vitest run
cd src/Editor/src/Bridge/CKEditor && npx vitest run
cd src/Editor/src/Bridge/GrapesJS && npx vitest run
```

Expected: all green.

- [ ] **Step 3: Full Playwright**

```bash
npx playwright test
```

Expected: all specs across editorjs / ckeditor / grapesjs / live projects pass.

- [ ] **Step 4: Tag demo wiring (local only, no push)**

```bash
git tag -a editor-demo/v0.1.0 -m "ux-editor demo wiring on ux.symfony.com 0.1.0"
```

- [ ] **Step 5: Verify tag**

```bash
git tag -l 'editor-demo/v0.1.0'
```

Expected: `editor-demo/v0.1.0`.

---

## Self-review for Plan 5

**1. Spec coverage:**
- §13 Demo app — three demo routes per format + one live showcase → Tasks 4, 5.
- §8 LiveComponent (`LiveEditor` trait + hot config + autosave UI) → Tasks 6, 9.
- §12 E2E — sharded Playwright projects per bridge + Live cross-cutting → Tasks 9, 10.
- §6 Asset stacks — AssetMapper importmap + Encore both supported → Tasks 1, 7, 8.
- §10 Phasing closes v1.0 → Task 12 tag.

**2. Placeholder scan:**
- Task 6 `form_factory` note: explicit fallback documented.
- Task 8 conditional edit: bounded (only if app overrides).
- Task 9 step 1 inspect base config: bounded adapt-to-existing.
- Task 10 step 6 fix iteration: bounded (locked bridges; failures must be demo-side).
- Task 11 inspect existing listing shape: bounded adapt-to-existing.
- No `TBD`/`FIXME`/handwave.

**3. Type consistency:**
- Entities use Doctrine types from Plan 1 Tasks 23/24/25 (`editor_html` / `editor_blocks` / `editor_page`).
- Form types use `EditorType` (Plan 1 Task 20) with bridge configs from Plans 2/3/4.
- Entity dumps match `HtmlContent`/`BlockContent`/`PageContent` fields from Plan 1 Tasks 4/5/6.
- `LiveEditorDemo` uses `LiveEditor` trait (Plan 1 Task 31): `dirty[]`, `lastSavedAt[]`, `draftRepo->upsert`, `getEntityId`.
- Playwright `data-test-id` attributes in templates exactly match selectors used by deferred specs in Plans 2/3/4 (`submit-button`, `entity-dump`, `editor-state`, `editorjs-canvas`, `dirty-state`, `toggle-readonly`).
- `window.__demoCkEditor` exposed in `wysiwyg.html.twig` matches Plan 3 Task 11 sanitize-on-submit spec.
- `window.__demoGrapes` exposed in `page.html.twig` matches Plan 4 Task 10 serialize-roundtrip + asset-extraction specs.
- `window.UXEditorJSTools` populated in `app.js` matches Plan 2 Task 13 tool-registry contract.
- Service-registered preset names (`blog.standard`, `wysiwyg.full`, `page_builder.landing`) match preset class registrations in Plans 2/3/4 Task 11/5/6/5.

No inline fixes needed.

---

## Plan 5 — complete

All 5 plans now written:
- `2026-05-19-ux-editor-core.md` (+ `-tasks-7-30`, `-tasks-31-50`, `-tasks-51-78`) — 78 tasks
- `2026-05-19-ux-editor-bridge-editorjs.md` — 22 tasks
- `2026-05-19-ux-editor-bridge-ckeditor.md` — 17 tasks
- `2026-05-19-ux-editor-bridge-grapesjs.md` — 16 tasks
- `2026-05-19-ux-editor-demo.md` — 12 tasks (this file)

**Total: 145 tasks across 5 plans for v1.0 ship of `symfony/ux-editor` ecosystem.**

---

## Execution handoff

Plan 5 saved to `docs/superpowers/plans/2026-05-19-ux-editor-demo.md`.

Execution options:

1. **Subagent-Driven (recommended)** — `superpowers:subagent-driven-development` dispatches one subagent per task.
2. **Inline** — `superpowers:executing-plans` walks all 12 tasks with checkpoints after Tasks 4, 6, 9, 12.

**Full v1.0 execution order across the 5 plans:**
1. Plan 1 (Tasks 1-78) — core + Tier 1 abstracts. Must complete first.
2. Plans 2, 3, 4 — three Tier 2 bridges. Parallelizable after Plan 1.
3. Plan 5 (this file) — demo + Playwright orchestration. Last.

Estimated 5-10 days for a single experienced engineer, or 2-3 days with parallel Plans 2-4 across multiple subagents.

Which approach for Plan 5, or proceed to execute Plan 1 first?
