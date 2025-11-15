# Brizy Visual Builder - WordPress Decoupling Analysis

## Executive Summary

**YES, the Brizy visual builder CAN be used outside WordPress**, but with significant caveats. The React-based editor is architecturally decoupled from WordPress at the UI layer, but it relies on well-defined API contracts that the backend must implement. The JavaScript/TypeScript code is largely platform-agnostic, with WordPress-specific logic isolated in configuration and API handlers.

---

## 1. Configuration Requirements (__VISUAL_CONFIG__)

### Configuration Structure

The builder requires two main configuration objects injected into the browser:

1. **`__VISUAL_CONFIG__`** (Editor-specific configuration)
   - Injected via PHP: `wp_add_inline_script('brizy-editor-vendor', "var __VISUAL_CONFIG__ = $editor_js_config;")`
   - Contains project data, page data, and API handlers

2. **`__BRZ_PLUGIN_ENV__`** (Plugin environment configuration)
   - Contains media URLs, action endpoints, and editor metadata
   - Injected via PHP: `wp_add_inline_script('brizy-client-editor', "var __BRZ_PLUGIN_ENV__ = $client_js_config;")`

### Key Configuration Interface

```typescript
// From: public/editor-src/editor/js/types/global.d.ts
interface VISUAL_CONFIG {
  // Events
  onStartLoad?: VoidFunction;
  onAutoSave?: (data: AutoSave) => void;
  onChange?: (data: OnChange) => void;
  
  // UI
  ui?: {
    publish?: {
      label?: string;
      handler: (res, rej, data) => void;  // CRITICAL: Save handler
    }
  };
  
  // API Handlers (see next section)
  api?: {...};
  
  // Dynamic content handlers
  dynamicContent?: {handler?: DCHandler};
  
  // Menu handlers
  menu?: {getMenus?: {handler: ...}};
}
```

### Initialization Flow (from /public/editor-src/editor/js/bootstraps/editor/index.tsx)

1. Editor reads `__VISUAL_CONFIG__` from window
2. Validates config exists and has projectData + pageData
3. Initializes fonts, styles, and project globals
4. Renders React `<Editor>` component with config
5. Calls `config.onLoad()` when ready

---

## 2. API Endpoints Required

The builder doesn't make HTTP requests directly in most cases. Instead, it expects **handler functions** in the config object. These handlers are responsible for making API calls.

### Critical API Handlers (Must Implement)

#### A. **Save/Publish Handler** (REQUIRED)
```typescript
ui.publish.handler = (res, rej, data) => {
  // data includes: {is_autosave, compiled HTML, pageData, projectData}
  // Must call res() on success or rej(error) on failure
}
```

#### B. **AutoSave Handler**
```typescript
onAutoSave = (data: AutoSave) => {
  // Called periodically with partial save data
  // AutoSave = {pageData?, projectData?, globalBlock?}
}
```

#### C. **onChange Handler**
```typescript
onChange = (data: OnChange) => {
  // Called when user modifies content
  // OnChange = {projectData?, pageData?, globalBlocks?}
}
```

### Supporting API Handlers (In config.api.*)

#### Media Upload
```typescript
api.media.addMedia = {
  handler: (res, rej, extra: AddImageExtra) => void;
}
api.media.mediaResizeUrl = "https://..."  // URL template
```

#### Save/Get Blocks, Layouts, Popups
```typescript
api.savedBlocks = {
  get: (res, rej, filter?) => Promise<SavedBlockAPIMeta[]>,
  create: (res, rej, data) => Promise<SavedBlock>,
  update: (res, rej, data) => Promise<SavedBlock>,
  delete: (res, rej, data) => Promise<void>,
  getByUid: (res, rej, {uid}) => Promise<SavedBlock>,
  import: (res, rej) => Promise<SavedBlockImport>,
  filter: {handler: (res, rej) => Promise<SavedBlockFilter>}
}
// Similar for: api.savedLayouts, api.savedPopups
```

#### Collection/Posts Handlers
```typescript
api.collectionItems = {
  searchCollectionItems: {handler: (res, rej, extra) => void},
  getCollectionItemsIds: {handler: (res, rej, {id}) => void}
}

api.collectionTypes = {
  loadCollectionTypes: {handler: (res, rej) => void}
}

api.posts = {
  getPosts: (res, rej, {search?, include?, postType?, excludePostType?, abortSignal?}) => void,
  getPostTaxonomies: (res, rej, {taxonomy, abortSignal?}) => void,
  getRulePostsGroupList: (res, rej, {postType}) => void
}

api.terms = {
  getTerms: (res, rej, taxonomy) => void,
  getTermsBy: (res, rej, {include?, search?, abortSignal?}) => void
}

api.authors = {
  getAuthors: (res, rej, {search?, include?, abortSignal?}) => void
}
```

#### Forms and Integrations
```typescript
integrations.form = {
  getForm, updateForm, getIntegration, updateIntegration,
  createIntegrationAccount, getIntegrationAccountApiKey,
  createIntegrationList, addRecaptcha, getSmtpIntegration,
  updateSmtpIntegration, deleteSmtpIntegration, addAccount,
  getAccounts, deleteAccount
}

integrations.fonts.upload = {
  get: (res, rej, {ids}) => void,
  upload: (res, rej, {files, name, id}) => void,
  delete: (res, rej, fontId) => void
}
```

#### Other Critical Handlers
- `api.sidebars.getSidebars` - Get available sidebars
- `api.menu.getMenus` - Get menus
- `api.shortcodeContent.handler` - Process shortcodes
- `api.heartBeat.sendHandler` - Project lock heartbeat
- `api.featuredImage.*` - Featured image operations
- `api.defaultKits.*` - Predefined block kits
- `api.defaultPopups.*` - Predefined popups
- `api.defaultLayouts.*` - Predefined layouts
- `api.screenshots.*` - Block screenshots

---

## 3. Data Structures

### Page Data Structure
```typescript
interface PageWP {
  id: string;
  title: string;
  slug: string;
  dataVersion: number;
  status: "draft" | "publish" | "future" | "private";
  dependencies: Array<string>;
  _kind: "wp";
  is_index: boolean;
  template: string;
  data: {
    items: Block[];        // Array of visual blocks
    editorVersion?: string;
    [k: string]: any;
  };
}
```

### Project Data Structure
```typescript
interface Project {
  id: string;
  dataVersion: number;
  data: {
    selectedKit: string;
    selectedStyle: string;
    styles: Style[];
    extraStyles: Style[];
    extraFontStyles: ExtraFontStyle[];
    font: string;
    fonts: Fonts;
    disabledElements: string[];
    pinnedElements: string[];
    editorVersion?: string;
  };
}
```

### Block Structure
Each block is a JSON object representing a visual element:
```typescript
interface Block {
  type: string;           // "row", "column", "text", "image", etc.
  value: {                // Visual properties
    _id: string;
    _component?: string;
    _styles?: string;
    [key: string]: any;
  };
  [key: string]: any;
}
```

---

## 4. WordPress-Specific Dependencies in Code

### Location: Minimal and Isolated

The TypeScript/JavaScript code has very few hard-coded WordPress dependencies:

#### A. Config Type Guard
```typescript
// public/editor-src/editor/js/global/Config/types/configs/WP.ts
export const isWp = (config: ConfigCommon): config is WP => TARGET === "WP";
```
- Uses compile-time `TARGET` constant to determine if running in WordPress mode
- Can be switched to different platforms (Cloud, Shopify, Ecwid)

#### B. WordPress-Specific Utils (Isolated)
```typescript
// public/editor-src/editor/js/utils/wp/index.ts
export const hasSidebars = (config: ConfigCommon): boolean => {
  if (isWp(config)) {
    return config.wp.hasSidebars;  // Only read if WP config
  }
  return false;
};

export const pluginActivated = (config: ConfigCommon, plugin: string): boolean => {
  if (isWp(config)) {
    const plugins = config.wp.plugins;
    return Boolean(plugins[plugin]);
  }
  return false;
};
```

#### C. WordPress Components
Files with `.wp.ts` or `.wp.tsx` suffix are WordPress-specific:
- `/editor/js/editorComponents/index.wp.ts` - WordPress element components
- `/editor/js/utils/elements/posts/index.wp.ts` - Post element handler

#### D. Global Block & Popup Handlers
Only WordPress uses certain features:
- Global Blocks for template systems
- Popup conditions mapped to WordPress rules

#### E. Dynamic Content System
```typescript
// Points to wp-specific providers
api.dynamicContent = {
  handler: (res, rej, {entityType, groupType}) => void
}
// Returns WordPress-specific placeholders like {{brizy_dc_permalink}}
```

### Key Finding: Platform Abstraction Works

The builder **already supports** multiple platforms:
- ✅ WordPress (WP)
- ✅ Cloud (Brizy cloud)
- ✅ Shopify
- ✅ Ecwid

Configuration is platform-agnostic. Only API handlers differ by platform.

---

## 5. Backend Functionality Requirements

### A. Core Operations (Absolutely Required)

1. **Save Page Data**
   - Store JSON blocks, metadata, compiled HTML
   - Update page status, dataVersion
   - Handle autosave vs. full save

2. **Load Page Data**
   - Return page configuration with all blocks
   - Return project globals
   - Return global blocks (templates)

3. **Compile HTML** (Client or Server)
   - Editor includes browser-based compiler
   - Can send raw JSON to server for server-side compilation
   - Result: compiled HTML + CSS + JS

4. **Media Upload & Management**
   - Upload image/file
   - Generate resized variants
   - Return URLs for insertion

### B. Semi-Critical Operations

1. **Saved Blocks/Layouts/Popups Management**
   - Get list, get by ID, create, update, delete
   - Can store as custom post types (WordPress does this)

2. **Dynamic Content Placeholders**
   - Provide list of available placeholder values
   - Resolve placeholders during compilation
   - Can be simple (return static list) or complex

3. **Global Blocks & Popup Conditions**
   - Store reusable components
   - Manage display rules
   - Handle version conflicts

### C. Optional Operations

- Featured image management
- Menu synchronization  
- User/author management
- Taxonomy/term management
- Sidebar registration
- Form integration with external services
- Font management (custom uploads)
- Adobe Fonts integration
- 3rd party integrations (Ecwid, Shopify, forms)

---

## 6. Builder Initialization & Minimum Setup

### Absolute Minimum Setup (Non-WordPress)

1. **Create a page/post record** with:
   ```
   id: string
   title: string
   status: "draft" | "publish"
   data: {items: []}  // Empty blocks array
   dataVersion: 1
   ```

2. **Create a project record** with:
   ```
   id: string
   data: {
     styles: [],
     fonts: {},
     selectedKit: string,
     selectedStyle: string
   }
   dataVersion: 1
   ```

3. **Create HTML page** that injects configuration:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
     <!-- Editor CSS -->
     <link rel="stylesheet" href="/brizy/editor/editor.min.css">
   </head>
   <body>
     <!-- Editor root -->
     <div id="brz-ed-root"></div>
     <div id="brz-popups"></div>
     
     <!-- Dependencies -->
     <script src="/brizy/editor/js/react.js"></script>
     <script src="/brizy/editor/js/react-dom.js"></script>
     <script src="/brizy/editor/js/editor.vendor.min.js"></script>
     
     <!-- Config injection -->
     <script>
       var __VISUAL_CONFIG__ = {
         mode: "page",
         projectData: {...},
         pageData: {...},
         globalBlocks: [],
         l10n: {},
         
         // Minimal handlers
         ui: {
           publish: {
             handler: (res, rej, data) => {
               // Send data.compiled to your backend
               console.log('Save:', data);
               res();
             }
           }
         },
         
         onAutoSave: (data) => {
           console.log('AutoSave:', data);
         },
         
         onLoad: () => {
           console.log('Editor ready');
         }
       };
     </script>
     
     <!-- Editor app -->
     <script src="/brizy/editor/js/editor.min.js"></script>
   </body>
   </html>
   ```

4. **Implement required API handlers** in config

### Typical Workflow

1. User loads editor page
2. `__VISUAL_CONFIG__` is parsed
3. Editor loads project + page data
4. User modifies blocks
5. Editor calls `onAutoSave` periodically
6. User clicks "Publish"
7. Editor calls `ui.publish.handler` with compiled HTML
8. Backend saves data, returns success
9. Page is accessible at public URL with compiled HTML

---

## 7. WordPress-Specific Bindings (Can Be Replaced)

### What's WordPress-Specific:

1. **Nonce Security** (`hash` parameter)
   - WordPress: `wp_create_nonce()`
   - Alternative: Your own token validation

2. **Admin AJAX** (`wp_ajax_` prefix)
   - WordPress: `add_action('wp_ajax_brizy_...')`
   - Alternative: REST API or custom endpoints

3. **Hooks System**
   - WordPress: `do_action()`, `apply_filters()`
   - Alternative: Event emitters or callbacks

4. **Post Meta Storage**
   - WordPress: `update_post_meta()`, `get_post_meta()`
   - Alternative: JSON columns, document stores, or any DB

5. **User Management**
   - WordPress: `current_user_can()`, `get_userdata()`
   - Alternative: Your permission system

6. **Post Types & Taxonomies**
   - WordPress: `register_post_type()`, `get_taxonomies()`
   - Alternative: Entity types defined in code

### What's Platform-Agnostic:

- ✅ React component architecture
- ✅ Block schema/structure
- ✅ Compilation system
- ✅ Configuration model
- ✅ API handler pattern
- ✅ Redux state management
- ✅ CSS/styling system

---

## 8. Comparison: Using Brizy Outside WordPress

### Option A: Headless CMS (Recommended for Non-WordPress)

Use Brizy as a visual builder connected to your own backend:

**Pros:**
- Full control over data storage
- No WordPress overhead
- Can use any backend (Node.js, Python, PHP, etc.)
- Can integrate with any CMS or custom system

**Cons:**
- Must implement all API handlers
- No built-in WordPress hooks/filters
- Custom integration work required

**Implementation Path:**
1. Extract editor files from Brizy plugin
2. Build backend API implementing required handlers
3. Create page/project management UI
4. Implement save/compile logic
5. Deploy compiled HTML to CDN or static hosting

### Option B: Embedded in Existing System

Integrate builder into existing non-WordPress PHP/Node.js/etc application:

**Requirements:**
- Database schema for pages, projects, blocks
- API endpoints matching handler signatures
- Authentication system
- Storage for compiled assets
- Media upload handling

---

## 9. Risk Assessment for Non-WordPress Use

### Challenges:

1. **No Official Non-WordPress Support**
   - Brizy is primarily a WordPress plugin
   - Going non-WordPress is "off the supported path"

2. **Complex Feature Set**
   - Many features assume WordPress (popups, templates, dynamic content)
   - Would need significant integration work

3. **Third-Party Integrations**
   - WooCommerce, form providers, etc. are WordPress-specific
   - Would need to reimplement or stub out

4. **Maintenance**
   - Plugin updates may introduce WordPress-specific code
   - Requires careful code review on updates

### Advantages:

1. **Core Editor is Decoupled**
   - React app is genuinely platform-agnostic
   - Config-driven architecture
   - Handler pattern is clean

2. **Well-Structured Code**
   - TypeScript types are comprehensive
   - Clear separation of concerns
   - Good API abstraction

3. **Reusable Compilation**
   - Browser-based compiler can generate static HTML
   - No need for server-side rendering

4. **Proven Architecture**
   - Brizy Cloud uses same editor (different config)
   - Shopify integration proves multi-platform support

---

## 10. Conclusion

### Can Brizy be used outside WordPress?

**Short Answer:** YES, but with caveats.

**Feasibility:** Medium to High (depending on feature requirements)

**Effort Required:** 
- Minimal setup: 1-2 weeks (basic save/load)
- Production setup: 4-8 weeks (full feature set)
- Enterprise setup: 8-16 weeks (custom integrations)

### Recommendation:

If you need a visual page builder and **don't require WordPress features**, consider:

1. **Use Brizy Cloud** - Already non-WordPress, hosted solution
2. **Build integration** - Extract editor, build custom backend
3. **License discussion** - Contact Brizy about licensing for non-WordPress use (important for legal/commercial use)

### Key Success Factors:

1. Implement all required API handlers correctly
2. Design proper data model for pages/projects/blocks
3. Build robust media upload system
4. Handle compilation (client-side or server-side)
5. Implement authentication/authorization
6. Plan for long-term maintenance

