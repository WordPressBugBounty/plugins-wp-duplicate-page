# Scout Report — Core Plugin (includes/, root, views/, assets/, i18n/)

## Purpose
WP Duplicate Page (namespace `NjtDuplicate`, plugin slug `wp-duplicate-page`, current version 1.8.5) is a WordPress admin utility plugin that lets authorized users duplicate posts, pages, custom post types, media attachments, and (with WooCommerce active) shop orders — including both classic and HPOS order storage. It adds a "Duplicate" row action/bulk action on list-table screens and an optional "Duplicate" button inside the block editor sidebar. Duplication is a deep clone: post meta, child posts, comments, taxonomy terms, and (for orders) line items/shipping/coupons/billing data are copied to a new draft. Access is gated by a custom capability (`njt_duplicate_page`) assignable per-role via a Settings admin page, which also controls which post types are duplicable, the link's button text, and whether the editor-sidebar button is shown. The plugin bundles an optional "recommended modules" ad/upsell system (`recommended-modules/`) wired into the bootstrap and settings page (see companion report).

## Found Files
- `wp-duplicate-page.php` - Plugin bootstrap. Defines constants (`NJT_DUPLICATE_VERSION`, `NJT_DUPLICATE_DOMAIN`, `NJT_DUPLICATE_PLUGIN_DIR/NAME/URL/PATH`), registers a PSR-4-like `spl_autoload_register` mapping `NjtDuplicate\*` to `includes/`, conditionally requires `recommended-modules/loader.php`, registers as ads-toggle consumer, defines `init()` (loads i18n + instantiates `Page\Settings`) on `plugins_loaded`, registers `Plugin::activate`/`deactivate`.
- `uninstall.php` - Guards on `WP_UNINSTALL_PLUGIN`; globals declared but no cleanup logic implemented.
- `index.php` (root, `includes/`, `includes/Page/`, `views/pages/`) - Standard "Silence is golden." guard.
- `includes/Plugin.php` - Singleton. `activate()` grants `njt_duplicate_page` capability to editor/administrator roles. `deactivate()` no-op.
- `includes/I18n.php` - `loadPluginTextdomain()` loads `wp-duplicate-page` textdomain from `i18n/languages/{locale}.mo`.
- `includes/Classes/ButtonDuplicate.php` - Singleton from `Page\Settings`. Hooks `admin_init` → registers `post_row_actions`/`page_row_actions`/`bulk_actions-edit-{type}`/`handle_bulk_actions-edit-{type}` filters (HPOS-aware for `shop_order`). Registers `admin_action_njt_duplicate_page_save_as_new_post` handler validating capability+nonce, calling `CreateDuplicate::createDuplicate()`, then redirecting.
- `includes/Classes/CreateDuplicate.php` - Core cloning logic: `createDuplicate($post, $parentId='')` → `wp_insert_post` draft + unique slug → `duplicateDetails()` fans to `copyPostMeta`, `copyChildrens` (recursive, skips attachments), `copyComments`, `copyTaxonomies`, `copyOrderDetails` (shop_order). `createDuplicateOrderHPOS()` parallel path for WooCommerce HPOS orders.
- `includes/Classes/EditorDuplicate.php` - Singleton, conditional on `njt_duplicate_in_editor` option. Hooks `post_submitbox_start` (classic editor link) and `enqueue_block_editor_assets` (enqueues `editor-duplicate.js`, localizes nonce'd link + button text).
- `includes/Helper/Utils.php` - Static helpers: `isCurrentUserAllowedToCopy()`, `checkPostTypeDuplicate($postType)` (option `njt_duplicate_post_types`, default `['post','page']`), `excludeMetaKey($key)` (filter `wp_duplicate_page_exclude_meta_key`), `getDuplicateLink($postId, $inEditor)`.
- `includes/Page/Settings.php` - Singleton on `plugins_loaded`. Hooks `admin_menu`, `admin_enqueue_scripts` (priority 20), `plugin_action_links_{name}`, `wp_ajax_njt_duplicate_page_settings` → `saveSettings()` (persists options, syncs capability across roles), `wp_ajax_njt_duplicate_page_track_review` → `trackReview()`. Instantiates `ButtonDuplicate` (always) and `EditorDuplicate` (conditional) — the actual wiring point for feature classes.
- `views/pages/html-settings.php` - Settings markup: role checkboxes, post-type checkboxes, link-text input, editor-button toggle, ads-toggle card, review-nudge footer.
- `assets/css/admin-setting.css` (722 lines), `assets/js/admin-setting.js` (98 lines, jQuery AJAX form submit + toast + tooltip), `assets/js/editor-duplicate.js` (37 lines, Gutenberg `PluginPostStatusInfo` button via `wp.plugins`/`wp.editPost`).
- `i18n/languages/` - `wp-duplicate-page.pot`, `en_US.po`/`.mo`.

## Architecture Notes
- **Bootstrap flow**: constants → autoloader → recommended-modules loader + ad-consumer registration → `plugins_loaded` → `init()` (i18n + `Page\Settings`) → activation grants capability.
- **Autoloading**: custom `spl_autoload_register`, `NjtDuplicate\Foo\Bar` → `includes/Foo/Bar.php`. No Composer for plugin's own classes.
- **Singleton pattern everywhere**: `Plugin`, `Page\Settings`, `ButtonDuplicate`, `CreateDuplicate`, `EditorDuplicate` — hook registration as a side effect of instantiation.
- **End-to-end duplicate flow**: `Settings` constructs `ButtonDuplicate` (always) + `EditorDuplicate` (conditional) → row/bulk action links built via `Utils::getDuplicateLink` → `admin_action_njt_duplicate_page_save_as_new_post` handler checks cap+nonce → `CreateDuplicate::createDuplicate()` deep-clones → redirect to edit screen (editor-triggered) or list table.
- **Settings page architecture**: `Page\Settings` renders `views/pages/html-settings.php` (plain PHP view). AJAX-only submit (`admin-ajax.php`), nonce-protected, `saveSettings()` persists options + reconciles capability across roles.

## Patterns
- Namespace `NjtDuplicate` mirroring directory structure (`Classes\`, `Helper\`, `Page\`).
- camelCase methods/properties; snake_case for WP hook names/option/meta keys.
- Prefixes: `njt_`/`njt_duplicate_` (options/caps), `njt-duplicate-` (CSS), `njt`/`njtDuplicateEditor` (JS globals).
- Singleton + hook-registration-in-constructor consistently.
- Capability/post-type checks centralized in `Helper\Utils`.
- Security: nonce + capability checks throughout; output escaped in views.
- One extensibility filter: `wp_duplicate_page_exclude_meta_key`.

## Unresolved Questions
- `uninstall.php` has no actual cleanup code — unclear if intentional.
- `modules.json`, `bin/`, `release/` (build/release tooling) outside scope of this pass — see companion recommended-modules report.
