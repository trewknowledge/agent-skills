# WordPress VIP Functions Reference

This document provides detailed reference for WordPress VIP-specific functions and replacements for restricted functions.

## VIP Helper Functions

### Image Handling

#### `wpcom_vip_download_image()`

Download and attach remote images to posts.

```php
/**
 * @param string $image_url URL of the image to download
 * @param int $post_id Post ID to attach the image to
 * @param string $description Optional description
 * @return int|WP_Error Attachment ID or error
 */
$attachment_id = wpcom_vip_download_image(
    'https://example.com/image.jpg',
    $post_id,
    'Image description'
);
```

### Safe Remote Requests

#### `vip_safe_wp_remote_get()`

Wrapper for `wp_remote_get()` with VIP-optimized timeouts and error handling.

```php
/**
 * @param string $url URL to request
 * @param string $fallback_value Value to return on failure
 * @param int $threshold Timeout threshold in seconds (default: 3)
 * @param int $timeout Request timeout in seconds (default: 3)  
 * @param int $retry Number of retries (default: 20)
 * @return mixed Response or fallback value
 */
$response = vip_safe_wp_remote_get(
    'https://api.example.com/data',
    [], // fallback value
    3,  // threshold
    3,  // timeout
    20  // retry
);
```

#### `vip_safe_wp_remote_request()`

Safe wrapper for custom HTTP methods.

```php
$response = vip_safe_wp_remote_request(
    'https://api.example.com/endpoint',
    [],
    [
        'method' => 'POST',
        'body' => json_encode($data),
        'headers' => [
            'Content-Type' => 'application/json',
        ],
    ]
);
```

### File System Functions

#### `get_temp_dir()`

VIP-compatible temporary directory.

```php
$temp_file = get_temp_dir() . 'temp_file.txt';
file_put_contents($temp_file, $data); // OK for temp files
// Clean up after use
unlink($temp_file);
```

### URL Functions

#### `wpcom_vip_get_term_link()`

Cached version of `get_term_link()`.

```php
$term_link = wpcom_vip_get_term_link($term, $taxonomy);
```

#### `wpcom_vip_get_term_by()`

Cached version of `get_term_by()`.

```php
$term = wpcom_vip_get_term_by('slug', 'term-slug', 'category');
```

#### `wpcom_vip_get_category_by_name()`

Cached category lookup.

```php
$category = wpcom_vip_get_category_by_name('Category Name');
```

## Restricted Functions and Alternatives

### File System Operations

| Restricted | Alternative | Notes |
|------------|-------------|-------|
| `file_put_contents()` | Object cache or VIP Files API | For persistent data |
| `fwrite()` | Object cache | Writes not allowed in production |
| `touch()` | N/A | File system is read-only |
| `mkdir()` | N/A | Cannot create directories |
| `unlink()` | N/A | Cannot delete files (except temp) |

### Caching

| Restricted | Alternative | Notes |
|------------|-------------|-------|
| `WP_Cache::add()` | `wp_cache_add()` | Use WP function |
| `WP_Cache::set()` | `wp_cache_set()` | Use WP function |
| File-based caching | Object cache | Memcached available |

### Database Operations

| Restricted | Alternative | Notes |
|------------|-------------|-------|
| Direct `$wpdb->query()` | Use WordPress functions | When possible |
| Unprepared queries | `$wpdb->prepare()` | Always prepare |
| `TRUNCATE` | `DELETE` with `WHERE` | Safer operation |

### WordPress Functions

| Restricted | Alternative | Notes |
|------------|-------------|-------|
| `get_term_link()` | `wpcom_vip_get_term_link()` | Cached version |
| `get_term_by()` | `wpcom_vip_get_term_by()` | Cached version |
| `get_category_by_slug()` | `wpcom_vip_get_category_by_name()` | Cached version |
| `get_posts()` without limit | Add `posts_per_page` | Prevent large queries |
| `switch_to_blog()` | Multisite functions | Use carefully |

### Execution Control

| Restricted | Alternative | Notes |
|------------|-------------|-------|
| `exec()` | N/A | Not allowed |
| `shell_exec()` | N/A | Not allowed |
| `system()` | N/A | Not allowed |
| `passthru()` | N/A | Not allowed |
| `eval()` | N/A | Security risk |
| `extract()` | Manual variable assignment | Security risk |

## VIP-Specific Constants

### Environment Detection

```php
// Check if running on VIP
if (defined('WPCOM_IS_VIP_ENV') && WPCOM_IS_VIP_ENV) {
    // VIP-specific code
}

// Check environment type
if (defined('VIP_GO_ENV') && 'production' === VIP_GO_ENV) {
    // Production-only code
}
```

### File Paths

```php
// VIP client MU plugins directory
WPCOM_VIP_CLIENT_MU_PLUGIN_DIR

// VIP plugins directory  
WP_CONTENT_DIR . '/plugins'

// Uploads directory (use WordPress functions instead)
wp_upload_dir()
```

## Performance Functions

### Batching Operations

#### `vip_inmemory_cleanup()`

Reset in-memory caches during large operations.

```php
foreach ($large_dataset as $index => $item) {
    process_item($item);
    
    // Every 500 items, clean up
    if (0 === $index % 500) {
        vip_inmemory_cleanup();
    }
}
```

### Cache Groups

WordPress VIP supports cache groups for organization:

```php
// Set with group
wp_cache_set('key', $value, 'group_name', $expiration);

// Get with group
$value = wp_cache_get('key', 'group_name');

// Delete with group
wp_cache_delete('key', 'group_name');
```

**Common cache groups:**
- `posts` - Post data
- `terms` - Taxonomy terms
- `options` - Site options
- Custom groups for your data

## WP-CLI on VIP

### Cache Management

```bash
# Flush object cache
vip @mysite.env wp cache flush

# Flush specific cache group
vip @mysite.env wp cache flush-group group_name

# Purge page cache for URL
vip @mysite.env wp vip-go purge-url https://example.com/page

# Purge entire site cache
vip @mysite.env wp vip-go purge-all
```

### Database Operations

```bash
# Export database
vip @mysite.env wp db export - > backup.sql

# Search and replace
vip @mysite.env wp search-replace 'old-domain.com' 'new-domain.com'

# Optimize tables
vip @mysite.env wp db optimize
```

### User Management

```bash
# Create admin user
vip @mysite.env wp user create username user@example.com --role=administrator

# List users
vip @mysite.env wp user list

# Update user password
vip @mysite.env wp user update 1 --user_pass=newpassword
```

## Debugging Functions

### Error Logging

```php
// Log to error log (auto-streamed to VIP logs)
error_log('Debug message: ' . print_r($data, true));

// Conditional logging
if (defined('WP_DEBUG') && WP_DEBUG) {
    error_log('Debug info');
}
```

### VIP Development Mode

```php
// Check if in development
if (defined('VIP_GO_ENV') && 'development' === VIP_GO_ENV) {
    // Development-only code
    var_dump($data); // OK in dev, remove before production
}
```

## Query Optimization Functions

### Batch Processing

Use `WP_Query` with pagination for large datasets:

```php
function process_all_posts() {
    $paged = 1;
    $posts_per_page = 100;
    
    do {
        $query = new WP_Query([
            'post_type' => 'post',
            'posts_per_page' => $posts_per_page,
            'paged' => $paged,
            'fields' => 'ids', // Only get IDs if that's all you need
        ]);
        
        if (!$query->have_posts()) {
            break;
        }
        
        foreach ($query->posts as $post_id) {
            process_post($post_id);
        }
        
        // Clean up
        wp_reset_postdata();
        vip_inmemory_cleanup();
        
        $paged++;
        
    } while ($query->have_posts());
}
```

### Optimized Queries

```php
// Get only IDs
$query = new WP_Query([
    'fields' => 'ids',
    'posts_per_page' => -1,
]);

// Disable term counting for bulk operations
wp_defer_term_counting(true);
// ... bulk operations ...
wp_defer_term_counting(false);

// Disable comment counting
wp_defer_comment_counting(true);
// ... bulk operations ...
wp_defer_comment_counting(false);
```

## Security Functions

### Escaping

```php
// Text output
echo esc_html($text);

// Attributes
echo '<div class="' . esc_attr($class) . '">';

// URLs
echo '<a href="' . esc_url($url) . '">';

// JavaScript
echo '<script>var data = ' . wp_json_encode($data) . ';</script>';

// SQL (use with $wpdb->prepare)
$wpdb->esc_like($string);
```

### Sanitization

```php
// Text fields
$clean = sanitize_text_field($_POST['field']);

// Textarea
$clean = sanitize_textarea_field($_POST['textarea']);

// Email
$email = sanitize_email($_POST['email']);

// URLs
$url = sanitize_url($_POST['url']);

// File name
$filename = sanitize_file_name($_FILES['file']['name']);

// Key (alphanumeric, dashes, underscores)
$key = sanitize_key($_POST['key']);

// HTML classes
$class = sanitize_html_class($_POST['class']);
```

### Capability Checks

```php
// Check user capability
if (!current_user_can('manage_options')) {
    wp_die('Unauthorized');
}

// Check for specific post
if (!current_user_can('edit_post', $post_id)) {
    wp_die('Cannot edit this post');
}

// Check admin referer (nonce)
check_admin_referer('action_name', 'nonce_field_name');
```
