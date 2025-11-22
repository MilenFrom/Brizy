# Brizy Visual Builder - Laravel Integration Guide

Complete guide to integrate the Brizy visual page builder into your existing Laravel application.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Installation Steps](#installation-steps)
4. [Database Setup](#database-setup)
5. [Backend Implementation](#backend-implementation)
6. [Frontend Integration](#frontend-integration)
7. [Configuration](#configuration)
8. [Testing](#testing)
9. [Advanced Features](#advanced-features)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Laravel 9.x or higher
- PHP 8.1+
- MySQL/PostgreSQL database
- Composer
- Node.js & NPM (for asset compilation, optional)

---

## Quick Start

**Time to complete:** ~30 minutes

```bash
# 1. Copy Brizy assets to your Laravel public directory
cp -r /path/to/Brizy/public/editor-build/prod ./public/brizy/editor-build/

# 2. Create migrations
php artisan make:migration create_brizy_projects_table
php artisan make:migration create_brizy_pages_table

# 3. Run migrations
php artisan migrate

# 4. Create models and controller
php artisan make:model BrizyProject
php artisan make:model BrizyPage
php artisan make:controller BrizyController

# 5. Access editor at /brizy/editor/{page_id}
```

---

## Installation Steps

### Step 1: Copy Brizy Editor Assets

Copy the compiled Brizy editor files to your Laravel `public` directory:

```bash
# From the Brizy repository root
mkdir -p public/brizy
cp -r /path/to/Brizy/public/editor-build/prod ./public/brizy/editor-build/
```

**Directory structure after copy:**

```
your-laravel-app/
└── public/
    └── brizy/
        └── editor-build/
            └── prod/
                ├── editor/
                │   ├── css/
                │   │   └── editor.min.css
                │   ├── js/
                │   │   ├── editor.min.js
                │   │   ├── editor.vendor.min.js
                │   │   ├── react.js
                │   │   ├── react-dom.js
                │   │   ├── polyfill.min.js
                │   │   └── compiler.min.js
                │   └── views/
                └── ...
```

**Verify assets are accessible:**

```bash
# Test in browser
http://your-app.test/brizy/editor-build/prod/editor/js/editor.min.js
```

---

### Step 2: Create Database Migrations

#### Migration 1: Brizy Projects Table

```bash
php artisan make:migration create_brizy_projects_table
```

**File:** `database/migrations/xxxx_xx_xx_create_brizy_projects_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('brizy_projects', function (Blueprint $table) {
            $table->id();
            $table->string('uid')->unique()->index();
            $table->json('data')->nullable(); // Fonts, styles, global settings
            $table->json('meta')->nullable(); // Additional metadata
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('brizy_projects');
    }
};
```

#### Migration 2: Brizy Pages Table

```bash
php artisan make:migration create_brizy_pages_table
```

**File:** `database/migrations/xxxx_xx_xx_create_brizy_pages_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('brizy_pages', function (Blueprint $table) {
            $table->id();
            $table->foreignId('project_id')
                ->constrained('brizy_projects')
                ->onDelete('cascade');
            $table->string('uid')->unique()->index();
            $table->string('title');
            $table->string('slug')->unique()->index();
            $table->json('data')->nullable(); // Page blocks JSON
            $table->json('compiled')->nullable(); // Compiled HTML/CSS/JS
            $table->longText('compiled_html')->nullable(); // Final rendered HTML
            $table->enum('status', ['draft', 'published', 'archived'])
                ->default('draft')
                ->index();
            $table->foreignId('user_id')
                ->nullable()
                ->constrained('users')
                ->onDelete('set null');
            $table->timestamp('published_at')->nullable();
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('brizy_pages');
    }
};
```

#### Run Migrations

```bash
php artisan migrate
```

---

### Step 3: Create Eloquent Models

#### Model 1: BrizyProject

**File:** `app/Models/BrizyProject.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class BrizyProject extends Model
{
    use HasFactory;

    protected $fillable = [
        'uid',
        'data',
        'meta',
    ];

    protected $casts = [
        'data' => 'array',
        'meta' => 'array',
    ];

    /**
     * Boot model events
     */
    protected static function boot()
    {
        parent::boot();

        static::creating(function ($project) {
            if (!$project->uid) {
                $project->uid = (string) Str::uuid();
            }
        });
    }

    /**
     * Relationship: Pages
     */
    public function pages()
    {
        return $this->hasMany(BrizyPage::class, 'project_id');
    }

    /**
     * Get default project data structure
     */
    public function getDefaultData(): array
    {
        return [
            'id' => $this->id,
            'data' => [
                'font' => 'lato',
                'fonts' => [
                    'google' => [
                        'id' => 'google',
                        'data' => [],
                    ],
                    'upload' => [
                        'data' => [],
                    ],
                ],
                'styles' => [],
                'extraFontStyles' => [],
                'extraStyles' => [],
            ],
        ];
    }

    /**
     * Get or create default project
     */
    public static function getDefault(): self
    {
        return static::firstOrCreate(
            ['uid' => 'default-project'],
            [
                'data' => [
                    'font' => 'lato',
                    'fonts' => [
                        'google' => ['id' => 'google', 'data' => []],
                        'upload' => ['data' => []],
                    ],
                ],
            ]
        );
    }
}
```

#### Model 2: BrizyPage

**File:** `app/Models/BrizyPage.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Str;

class BrizyPage extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'project_id',
        'uid',
        'title',
        'slug',
        'data',
        'compiled',
        'compiled_html',
        'status',
        'user_id',
        'published_at',
    ];

    protected $casts = [
        'data' => 'array',
        'compiled' => 'array',
        'published_at' => 'datetime',
    ];

    /**
     * Boot model events
     */
    protected static function boot()
    {
        parent::boot();

        static::creating(function ($page) {
            if (!$page->uid) {
                $page->uid = (string) Str::uuid();
            }
            if (!$page->slug && $page->title) {
                $page->slug = Str::slug($page->title);
            }
        });

        static::updating(function ($page) {
            if ($page->isDirty('title') && !$page->isDirty('slug')) {
                $page->slug = Str::slug($page->title);
            }
        });
    }

    /**
     * Relationship: Project
     */
    public function project()
    {
        return $this->belongsTo(BrizyProject::class, 'project_id');
    }

    /**
     * Relationship: User (creator)
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get default page data structure
     */
    public function getDefaultData(): array
    {
        return [
            'items' => [],
            'triggers' => [],
        ];
    }

    /**
     * Scope: Published pages only
     */
    public function scopePublished($query)
    {
        return $query->where('status', 'published');
    }

    /**
     * Scope: Draft pages only
     */
    public function scopeDraft($query)
    {
        return $query->where('status', 'draft');
    }

    /**
     * Publish the page
     */
    public function publish(): bool
    {
        return $this->update([
            'status' => 'published',
            'published_at' => now(),
        ]);
    }

    /**
     * Unpublish the page
     */
    public function unpublish(): bool
    {
        return $this->update([
            'status' => 'draft',
        ]);
    }
}
```

---

### Step 4: Create Controller

```bash
php artisan make:controller BrizyController
```

**File:** `app/Http/Controllers/BrizyController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\BrizyPage;
use App\Models\BrizyProject;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class BrizyController extends Controller
{
    /**
     * List all pages (admin view)
     */
    public function index()
    {
        $pages = BrizyPage::with('project')
            ->latest()
            ->paginate(20);

        return view('brizy.index', compact('pages'));
    }

    /**
     * Show the Brizy editor for a page
     */
    public function editor(BrizyPage $page)
    {
        $project = $page->project ?? BrizyProject::getDefault();

        return view('brizy.editor', [
            'page' => $page,
            'project' => $project,
        ]);
    }

    /**
     * Create a new page
     */
    public function create(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
        ]);

        $project = BrizyProject::getDefault();

        $page = BrizyPage::create([
            'project_id' => $project->id,
            'title' => $validated['title'],
            'user_id' => auth()->id(),
            'data' => ['items' => []],
        ]);

        return redirect()->route('brizy.editor', $page);
    }

    /**
     * API: Get page data
     */
    public function getPage(BrizyPage $page)
    {
        return response()->json([
            'data' => [
                'id' => $page->id,
                'uid' => $page->uid,
                'title' => $page->title,
                'slug' => $page->slug,
                'status' => $page->status,
                'data' => $page->data ?? $page->getDefaultData(),
            ],
        ]);
    }

    /**
     * API: Save page data
     */
    public function savePage(Request $request, BrizyPage $page)
    {
        $validated = $request->validate([
            'title' => 'sometimes|string|max:255',
            'data' => 'required|array',
            'status' => 'sometimes|in:draft,published',
            'compiled' => 'sometimes|array',
            'compiled_html' => 'sometimes|string',
        ]);

        $updateData = [
            'data' => $validated['data'],
        ];

        if (isset($validated['title'])) {
            $updateData['title'] = $validated['title'];
        }

        if (isset($validated['status'])) {
            $updateData['status'] = $validated['status'];
            if ($validated['status'] === 'published') {
                $updateData['published_at'] = now();
            }
        }

        if (isset($validated['compiled'])) {
            $updateData['compiled'] = $validated['compiled'];
        }

        if (isset($validated['compiled_html'])) {
            $updateData['compiled_html'] = $validated['compiled_html'];
        }

        $page->update($updateData);

        return response()->json([
            'data' => 'success',
            'page' => [
                'id' => $page->id,
                'uid' => $page->uid,
                'updated_at' => $page->updated_at->toIso8601String(),
            ],
        ]);
    }

    /**
     * API: Get project data
     */
    public function getProject(Request $request)
    {
        $project = BrizyProject::getDefault();

        return response()->json([
            'data' => $project->data ?? $project->getDefaultData(),
        ]);
    }

    /**
     * API: Save project data
     */
    public function saveProject(Request $request)
    {
        $validated = $request->validate([
            'data' => 'required|array',
        ]);

        $project = BrizyProject::getDefault();
        $project->update(['data' => $validated['data']]);

        return response()->json([
            'data' => 'success',
        ]);
    }

    /**
     * API: Upload media
     */
    public function uploadMedia(Request $request)
    {
        $request->validate([
            'file' => 'required|file|mimes:jpeg,png,jpg,gif,svg,webp,mp4,webm,pdf|max:20480',
        ]);

        $file = $request->file('file');
        $filename = Str::random(40) . '.' . $file->getClientOriginalExtension();
        $path = $file->storeAs('brizy/media', $filename, 'public');
        $url = Storage::url($path);

        return response()->json([
            'uid' => (string) Str::uuid(),
            'url' => url($url),
            'filename' => $file->getClientOriginalName(),
            'size' => $file->getSize(),
            'mime' => $file->getMimeType(),
        ]);
    }

    /**
     * Display published page on frontend
     */
    public function show($slug)
    {
        $page = BrizyPage::where('slug', $slug)
            ->where('status', 'published')
            ->firstOrFail();

        return view('brizy.page', [
            'page' => $page,
            'html' => $page->compiled_html ?? $this->renderPage($page),
        ]);
    }

    /**
     * Delete page
     */
    public function destroy(BrizyPage $page)
    {
        $page->delete();

        return redirect()->route('brizy.index')
            ->with('success', 'Page deleted successfully');
    }

    /**
     * Render page HTML (fallback if no compiled HTML)
     */
    protected function renderPage(BrizyPage $page): string
    {
        // Basic fallback - in production you'd implement proper compilation
        $items = $page->data['items'] ?? [];

        if (empty($items)) {
            return '<div class="brz"><p>Empty page</p></div>';
        }

        // Return compiled HTML if available
        if ($page->compiled && isset($page->compiled['html'])) {
            return $page->compiled['html'];
        }

        return '<div class="brz">Page content</div>';
    }
}
```

---

### Step 5: Define Routes

**File:** `routes/web.php`

```php
<?php

use App\Http\Controllers\BrizyController;
use Illuminate\Support\Facades\Route;

// Admin routes (protected by auth middleware)
Route::middleware(['auth'])->prefix('admin/brizy')->name('brizy.')->group(function () {
    Route::get('/', [BrizyController::class, 'index'])->name('index');
    Route::post('/pages', [BrizyController::class, 'create'])->name('create');
    Route::get('/editor/{page}', [BrizyController::class, 'editor'])->name('editor');
    Route::delete('/pages/{page}', [BrizyController::class, 'destroy'])->name('destroy');
});

// API routes (protected by auth middleware)
Route::middleware(['auth'])->prefix('brizy/api')->name('brizy.api.')->group(function () {
    Route::get('/page/{page}', [BrizyController::class, 'getPage'])->name('page.get');
    Route::post('/page/{page}', [BrizyController::class, 'savePage'])->name('page.save');
    Route::get('/project', [BrizyController::class, 'getProject'])->name('project.get');
    Route::post('/project', [BrizyController::class, 'saveProject'])->name('project.save');
    Route::post('/media', [BrizyController::class, 'uploadMedia'])->name('media');
});

// Public frontend route
Route::get('/pages/{slug}', [BrizyController::class, 'show'])->name('brizy.page.show');
```

---

### Step 6: Create Blade Views

#### View 1: Editor View

**File:** `resources/views/brizy/editor.blade.php`

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>Brizy Editor - {{ $page->title }}</title>

    <!-- Brizy Editor CSS -->
    <link rel="stylesheet" href="{{ asset('brizy/editor-build/prod/editor/css/editor.min.css') }}">

    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { margin: 0; padding: 0; overflow: hidden; }
        #brz-ed-root { width: 100%; height: 100vh; }
    </style>
</head>
<body>
    <div id="brz-ed-root"></div>

    <script>
        // Brizy Editor Configuration
        window.__VISUAL_CONFIG__ = {
            mode: "page",

            // Page data
            pageData: @json($page->data ?? $page->getDefaultData()),

            // Project data (global styles, fonts, etc.)
            projectData: @json($project->data ?? $project->getDefaultData()),

            // API handlers
            api: {
                // Publish/Save handler
                ui: {
                    publish: {
                        handler: async (res, rej, data) => {
                            try {
                                const response = await fetch("{{ route('brizy.api.page.save', $page) }}", {
                                    method: 'POST',
                                    headers: {
                                        'Content-Type': 'application/json',
                                        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
                                        'Accept': 'application/json'
                                    },
                                    body: JSON.stringify({
                                        data: data.page,
                                        status: data.status || 'draft',
                                        compiled: data.compiled || null,
                                        compiled_html: data.html || null
                                    })
                                });

                                if (!response.ok) {
                                    throw new Error(`HTTP error! status: ${response.status}`);
                                }

                                const result = await response.json();
                                res({ data: result });
                            } catch (error) {
                                console.error('Save error:', error);
                                rej(error.message);
                            }
                        }
                    }
                },

                // Auto-save handler (called periodically)
                onAutoSave: async (data) => {
                    try {
                        await fetch("{{ route('brizy.api.page.save', $page) }}", {
                            method: 'POST',
                            headers: {
                                'Content-Type': 'application/json',
                                'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
                                'Accept': 'application/json'
                            },
                            body: JSON.stringify({
                                data: data.page,
                                status: 'draft'
                            })
                        });
                    } catch (error) {
                        console.error('Auto-save error:', error);
                    }
                },

                // Media upload handler
                media: {
                    addMedia: {
                        handler: async (res, rej, file) => {
                            const formData = new FormData();
                            formData.append('file', file);

                            try {
                                const response = await fetch("{{ route('brizy.api.media') }}", {
                                    method: 'POST',
                                    headers: {
                                        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
                                        'Accept': 'application/json'
                                    },
                                    body: formData
                                });

                                if (!response.ok) {
                                    throw new Error(`Upload failed! status: ${response.status}`);
                                }

                                const result = await response.json();
                                res(result);
                            } catch (error) {
                                console.error('Upload error:', error);
                                rej(error.message);
                            }
                        }
                    }
                },

                // Project save handler
                project: {
                    save: {
                        handler: async (res, rej, data) => {
                            try {
                                const response = await fetch("{{ route('brizy.api.project.save') }}", {
                                    method: 'POST',
                                    headers: {
                                        'Content-Type': 'application/json',
                                        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
                                        'Accept': 'application/json'
                                    },
                                    body: JSON.stringify({ data })
                                });

                                if (!response.ok) {
                                    throw new Error(`HTTP error! status: ${response.status}`);
                                }

                                const result = await response.json();
                                res(result);
                            } catch (error) {
                                console.error('Project save error:', error);
                                rej(error.message);
                            }
                        }
                    }
                },

                // Saved blocks (optional - can implement later)
                savedBlocks: {
                    get: { handler: (res) => res([]) },
                    create: { handler: (res) => res({}) },
                    update: { handler: (res) => res({}) },
                    delete: { handler: (res) => res({}) }
                },

                // Saved layouts (optional - can implement later)
                savedLayouts: {
                    get: { handler: (res) => res([]) },
                    create: { handler: (res) => res({}) },
                    update: { handler: (res) => res({}) },
                    delete: { handler: (res) => res({}) }
                }
            },

            // URLs configuration
            urls: {
                editorAssets: "{{ asset('brizy/editor-build/prod') }}/",
                pageUrl: "{{ route('brizy.page.show', $page->slug) }}",
                api: "{{ route('brizy.api.page.save', $page) }}",
                uploads: "{{ asset('storage/brizy/media') }}/",
                image: "{{ asset('storage/brizy/media') }}/",
            },

            // User configuration
            user: {
                role: "admin",
                allowScripts: true,
                allowExternalAssets: true
            },

            // UI configuration
            ui: {
                leftSidebar: {
                    topTabsOrder: [
                        { id: "addElements", type: "addElements" },
                        { id: "reorderBlock", type: "reorderBlock" },
                        { id: "globalStyle", type: "globalStyle" }
                    ]
                }
            },

            // Disable cloud-dependent features
            cloud: {
                enabled: false
            }
        };
    </script>

    <!-- Brizy Editor Scripts -->
    <script src="{{ asset('brizy/editor-build/prod/editor/js/polyfill.min.js') }}"></script>
    <script src="{{ asset('brizy/editor-build/prod/editor/js/react.js') }}"></script>
    <script src="{{ asset('brizy/editor-build/prod/editor/js/react-dom.js') }}"></script>
    <script src="{{ asset('brizy/editor-build/prod/editor/js/editor.vendor.min.js') }}"></script>
    <script src="{{ asset('brizy/editor-build/prod/editor/js/editor.min.js') }}"></script>
</body>
</html>
```

#### View 2: Frontend Page View

**File:** `resources/views/brizy/page.blade.php`

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="{{ $page->title }}">
    <title>{{ $page->title }}</title>

    <!-- Compiled CSS -->
    @if($page->compiled && isset($page->compiled['css']))
        <style>{!! $page->compiled['css'] !!}</style>
    @endif

    <style>
        body { margin: 0; padding: 0; }
        .brz { min-height: 100vh; }
    </style>
</head>
<body class="brz">

    <!-- Page Content -->
    {!! $html !!}

    <!-- Compiled JavaScript -->
    @if($page->compiled && isset($page->compiled['js']))
        <script>{!! $page->compiled['js'] !!}</script>
    @endif

</body>
</html>
```

#### View 3: Pages List (Admin)

**File:** `resources/views/brizy/index.blade.php`

```blade
@extends('layouts.app')

@section('content')
<div class="container mx-auto px-4 py-8">
    <div class="flex justify-between items-center mb-6">
        <h1 class="text-3xl font-bold">Brizy Pages</h1>

        <button
            onclick="document.getElementById('createModal').classList.remove('hidden')"
            class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded"
        >
            Create New Page
        </button>
    </div>

    <!-- Pages Table -->
    <div class="bg-white shadow rounded-lg overflow-hidden">
        <table class="min-w-full">
            <thead class="bg-gray-50">
                <tr>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Title</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Slug</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Status</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Updated</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Actions</th>
                </tr>
            </thead>
            <tbody class="divide-y divide-gray-200">
                @forelse($pages as $page)
                    <tr>
                        <td class="px-6 py-4">{{ $page->title }}</td>
                        <td class="px-6 py-4 text-sm text-gray-500">{{ $page->slug }}</td>
                        <td class="px-6 py-4">
                            <span class="px-2 py-1 text-xs rounded-full
                                {{ $page->status === 'published' ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800' }}">
                                {{ ucfirst($page->status) }}
                            </span>
                        </td>
                        <td class="px-6 py-4 text-sm text-gray-500">
                            {{ $page->updated_at->diffForHumans() }}
                        </td>
                        <td class="px-6 py-4 text-sm space-x-2">
                            <a href="{{ route('brizy.editor', $page) }}"
                               class="text-blue-600 hover:text-blue-800">
                                Edit
                            </a>
                            @if($page->status === 'published')
                                <a href="{{ route('brizy.page.show', $page->slug) }}"
                                   target="_blank"
                                   class="text-green-600 hover:text-green-800">
                                    View
                                </a>
                            @endif
                            <form action="{{ route('brizy.destroy', $page) }}"
                                  method="POST"
                                  class="inline"
                                  onsubmit="return confirm('Are you sure?')">
                                @csrf
                                @method('DELETE')
                                <button type="submit" class="text-red-600 hover:text-red-800">
                                    Delete
                                </button>
                            </form>
                        </td>
                    </tr>
                @empty
                    <tr>
                        <td colspan="5" class="px-6 py-4 text-center text-gray-500">
                            No pages found. Create your first page!
                        </td>
                    </tr>
                @endforelse
            </tbody>
        </table>
    </div>

    <!-- Pagination -->
    <div class="mt-4">
        {{ $pages->links() }}
    </div>
</div>

<!-- Create Page Modal -->
<div id="createModal" class="hidden fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center">
    <div class="bg-white rounded-lg p-8 max-w-md w-full">
        <h2 class="text-2xl font-bold mb-4">Create New Page</h2>
        <form action="{{ route('brizy.create') }}" method="POST">
            @csrf
            <div class="mb-4">
                <label for="title" class="block text-sm font-medium text-gray-700 mb-2">
                    Page Title
                </label>
                <input
                    type="text"
                    name="title"
                    id="title"
                    required
                    class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                    placeholder="Enter page title"
                >
            </div>
            <div class="flex justify-end space-x-3">
                <button
                    type="button"
                    onclick="document.getElementById('createModal').classList.add('hidden')"
                    class="px-4 py-2 text-gray-700 bg-gray-200 rounded hover:bg-gray-300"
                >
                    Cancel
                </button>
                <button
                    type="submit"
                    class="px-4 py-2 text-white bg-blue-600 rounded hover:bg-blue-700"
                >
                    Create Page
                </button>
            </div>
        </form>
    </div>
</div>
@endsection
```

---

## Configuration

### Environment Variables

Add to your `.env` file:

```env
# Brizy Configuration
BRIZY_CLOUD_ENABLED=false
BRIZY_MEDIA_MAX_SIZE=20480
BRIZY_ALLOWED_MIME_TYPES=jpeg,png,jpg,gif,svg,webp,mp4,webm
```

### File Storage

Ensure storage is linked:

```bash
php artisan storage:link
```

This creates a symlink from `public/storage` to `storage/app/public`.

### Permissions

Set correct permissions for storage:

```bash
chmod -R 775 storage
chmod -R 775 bootstrap/cache
```

---

## Testing

### Step 1: Create Your First Page

```bash
php artisan tinker
```

```php
// Create a project
$project = \App\Models\BrizyProject::create([
    'uid' => 'default-project',
    'data' => []
]);

// Create a page
$page = \App\Models\BrizyPage::create([
    'project_id' => $project->id,
    'title' => 'Home Page',
    'slug' => 'home',
    'user_id' => 1, // Your user ID
    'data' => ['items' => []]
]);

echo "Page created! Edit at: /admin/brizy/editor/{$page->id}";
```

### Step 2: Access the Editor

Navigate to:

```
http://your-app.test/admin/brizy/editor/1
```

### Step 3: Test Basic Operations

1. **Add a section** - Drag a section from the left sidebar
2. **Add text** - Add a text element
3. **Save** - Click the publish button
4. **View frontend** - Visit `/pages/home`

---

## Advanced Features

### 1. Add Custom Template Library

Extend `__VISUAL_CONFIG__` in `editor.blade.php`:

```javascript
api: {
    // ... existing handlers

    // Custom templates
    defaultKits: {
        label: "My Templates",
        getKits: (res) => {
            fetch('/api/brizy/templates/kits')
                .then(r => r.json())
                .then(res)
                .catch(() => res([]));
        },
        getMeta: (res, rej, kitId) => {
            fetch(`/api/brizy/templates/kits/${kitId}`)
                .then(r => r.json())
                .then(res)
                .catch(rej);
        },
        getData: (res, rej, block) => {
            fetch(`/api/brizy/templates/blocks/${block.id}`)
                .then(r => r.json())
                .then(res)
                .catch(rej);
        }
    }
}
```

### 2. Add Custom Elements

Create a custom component:

```javascript
window.__VISUAL_CONFIG__ = {
    // ... existing config

    thirdPartyComponents: {
        "custom-product-loop": {
            id: "custom-product-loop",
            title: "Product Loop",
            icon: "nc-star",
            category: "custom",
            component: {
                editor: MyProductLoopEditor,
                view: MyProductLoopView
            },
            options: (props) => [
                {
                    id: "settings",
                    type: "popover",
                    options: [
                        {
                            id: "category",
                            type: "select",
                            label: "Category",
                            choices: [
                                { value: "all", title: "All Products" },
                                { value: "featured", title: "Featured" }
                            ]
                        }
                    ]
                }
            ]
        }
    }
};
```

### 3. Multi-Language Support

Add language field to pages table:

```php
Schema::table('brizy_pages', function (Blueprint $table) {
    $table->string('language', 5)->default('en')->index();
});
```

Update routes to support language:

```php
Route::get('/{lang}/pages/{slug}', [BrizyController::class, 'show'])
    ->where('lang', '[a-z]{2}');
```

### 4. Page Revisions

Create revisions table:

```php
Schema::create('brizy_page_revisions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('page_id')->constrained('brizy_pages')->onDelete('cascade');
    $table->json('data');
    $table->foreignId('user_id')->nullable()->constrained('users');
    $table->timestamps();
});
```

Save revision on each update:

```php
// In BrizyController@savePage
BrizyPageRevision::create([
    'page_id' => $page->id,
    'data' => $page->data,
    'user_id' => auth()->id()
]);
```

---

## Troubleshooting

### Issue: Editor Not Loading

**Solution:**

```bash
# Verify assets exist
ls -la public/brizy/editor-build/prod/editor/js/editor.min.js

# Check browser console for 404 errors
# Ensure asset URLs are correct
```

### Issue: CSRF Token Mismatch

**Solution:**

Ensure CSRF token is in the meta tag:

```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

And included in fetch requests:

```javascript
headers: {
    'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
}
```

### Issue: Media Upload Fails

**Solution:**

```bash
# Check storage permissions
chmod -R 775 storage/app/public

# Verify storage link exists
php artisan storage:link

# Check upload limits in php.ini
upload_max_filesize = 20M
post_max_size = 20M
```

### Issue: JSON Column Errors (SQLite)

**Solution:**

SQLite doesn't support JSON natively. Use MySQL or PostgreSQL, or change columns to `TEXT`:

```php
$table->text('data')->nullable();
$table->text('meta')->nullable();
```

And handle JSON encoding/decoding manually in the model.

### Issue: Assets Not Found in Production

**Solution:**

Run asset optimization:

```bash
php artisan optimize
php artisan config:cache
php artisan route:cache
```

Ensure `APP_URL` in `.env` is correct.

---

## Production Checklist

- [ ] Run migrations on production database
- [ ] Copy Brizy assets to production server
- [ ] Set correct file permissions (755 for directories, 644 for files)
- [ ] Configure storage symlink
- [ ] Set `APP_ENV=production` and `APP_DEBUG=false`
- [ ] Run `php artisan optimize`
- [ ] Configure HTTPS for editor (required for some features)
- [ ] Set up regular database backups
- [ ] Configure CDN for media assets (optional)
- [ ] Test save/load functionality
- [ ] Test media uploads
- [ ] Test published pages display correctly

---

## Next Steps

1. **Implement server-side compilation** - Process Brizy JSON to HTML on the server
2. **Add SEO fields** - Meta descriptions, OG tags, etc.
3. **Create custom blocks** - Build reusable components specific to your app
4. **Add permissions** - Control who can edit/publish pages
5. **Implement template library** - Provide pre-built page templates
6. **Add analytics** - Track page views and engagement
7. **Version control** - Implement page revision history
8. **Multi-site support** - Manage multiple sites from one installation

---

## Support & Resources

- **Brizy Repository**: Study the WordPress integration at `/home/user/Brizy`
- **Key Files to Reference**:
  - `public/main.php` - How WordPress loads the editor
  - `editor/api.php` - API endpoint examples
  - `public/editor-src/editor/js/utils/api/index.ts` - API interface definitions
  - `config.php` - Configuration constants

---

## License

This integration guide is provided as-is. Ensure you comply with Brizy's licensing terms when using the visual builder in your application.

---

**Last Updated:** 2025-01-22
