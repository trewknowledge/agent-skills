---
name: wordpress-vip
description: Develop and deploy code for WordPress VIP environments following platform requirements, coding standards, and best practices. Use when working with WordPress VIP projects, VIP-CLI, VIP-hosted sites, or when the user mentions WordPress VIP, WPVIP, VIP Go, or enterprise WordPress.
---

# WordPress VIP Development

This skill guides development for WordPress VIP (WPVIP) platform, covering coding standards, deployment, performance, security, and platform-specific requirements.

## Quick Reference

WordPress VIP is an enterprise WordPress hosting platform with strict requirements:

- **Prohibited**: file system writes, uncached external requests, certain plugins/functions
- **Required**: code review, automated scanning, performance optimization
- **Deployment**: via GitHub integration or VIP-CLI
- **Local Dev**: VIP Local Dev Environment or Docker
- **Search**: Enterprise Search powered by Elasticsearch via ElasticPress

## Coding Standards

### File System Restrictions

WordPress VIP uses a read-only file system in production.

**Prohibited operations:**
- Direct file writes (`file_put_contents()`, `fwrite()`)
- Image uploads to local filesystem
- Cache writes to files
- Log writes to files

**VIP-approved alternatives:**
- Use `wpcom_vip_download_image()` for remote images
- Store uploads in VIP's object storage
- Use persistent object cache (Memcached)
- Use `error_log()` for logging (auto-streamed to logs)

### Restricted Functions

Functions that are blocked or discouraged:

```php
// Prohibited - direct database queries
$wpdb->query("DELETE FROM...");

// Use instead - WordPress functions
wp_delete_post($post_id);

// Prohibited - uncached remote requests
file_get_contents('https://api.example.com');
wp_remote_get('https://api.example.com'); // without caching

// Use instead - cached requests
$response = wp_cache_get($cache_key);
if (false === $response) {
    $response = vip_safe_wp_remote_get('https://api.example.com');
    wp_cache_set($cache_key, $response, 'api', HOUR_IN_SECONDS);
}
```

### Required Prefixes

All custom functions, classes, and constants must be prefixed:

```php
// Good - prefixed
function clientname_custom_function() {}
class ClientName_Custom_Class {}
define('CLIENTNAME_CONSTANT', 'value');

// Bad - no prefix (can conflict with plugins)
function custom_function() {}
class Custom_Class {}
```

### Database Queries

Follow WordPress VIP query patterns:

```php
// Bad - direct query without prepare
$results = $wpdb->get_results("SELECT * FROM $wpdb->posts WHERE post_status = 'publish'");

// Good - use WP_Query
$query = new WP_Query([
    'post_status' => 'publish',
    'posts_per_page' => 100,
]);

// If direct query needed - use prepare
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM $wpdb->posts WHERE post_status = %s",
        'publish'
    )
);
```

## Plugin Management

### Prohibited Plugins

Never allowed: caching plugins (VIP provides caching), security plugins that modify .htaccess, backup plugins (VIP handles backups), or plugins that write to filesystem.

Before using any plugin, verify: no file system writes, no uncached external requests, no restricted functions, performance tested, security reviewed.

## Performance Requirements

### Query Optimization

**Batch Processing:**
```php
// Bad - queries in loop
foreach ($post_ids as $post_id) {
    $post = get_post($post_id); // N queries
}

// Good - batch query
$posts = get_posts([
    'post__in' => $post_ids,
    'posts_per_page' => -1,
]);
```

**Object Caching:**
```php
function clientname_get_expensive_data($id) {
    $cache_key = 'expensive_data_' . $id;
    $data = wp_cache_get($cache_key, 'clientname');
    
    if (false === $data) {
        // Expensive operation
        $data = perform_expensive_operation($id);
        wp_cache_set($cache_key, $data, 'clientname', HOUR_IN_SECONDS);
    }
    
    return $data;
}
```

### Page Cache Compatibility

Ensure code works with full-page caching:

```php
// Bad - will show same content to all users
echo 'Welcome, ' . wp_get_current_user()->display_name;

// Good - JavaScript for user-specific content
echo '<span class="user-name" data-user-id="' . get_current_user_id() . '"></span>';
// Populate via AJAX or REST API
```

### External Requests

Always cache external API calls:

```php
function clientname_fetch_api_data($endpoint) {
    $cache_key = md5('api_' . $endpoint);
    $data = wp_cache_get($cache_key, 'api');
    
    if (false === $data) {
        $response = vip_safe_wp_remote_get(
            $endpoint,
            '',
            3,
            3,
            20
        );
        
        if (is_wp_error($response)) {
            return false;
        }
        
        $data = json_decode(wp_remote_retrieve_body($response), true);
        wp_cache_set($cache_key, $data, 'api', 15 * MINUTE_IN_SECONDS);
    }
    
    return $data;
}
```

## Security Requirements

### Input Validation

Always validate and sanitize:

```php
// User input
$user_input = sanitize_text_field($_POST['field_name']);
$email = sanitize_email($_POST['email']);
$url = esc_url_raw($_POST['url']);

// Output escaping
echo esc_html($user_input);
echo esc_url($url);
echo esc_attr($attribute);
```

### Nonce Verification

Protect forms and AJAX:

```php
// Form with nonce
wp_nonce_field('clientname_action', 'clientname_nonce');

// Verify nonce
if (!isset($_POST['clientname_nonce']) || 
    !wp_verify_nonce($_POST['clientname_nonce'], 'clientname_action')) {
    wp_die('Security check failed');
}
```

### SQL Injection Prevention

Always use `$wpdb->prepare()`:

```php
// Correct
$results = $wpdb->get_results($wpdb->prepare(
    "SELECT * FROM $wpdb->posts WHERE post_title LIKE %s",
    '%' . $wpdb->esc_like($search) . '%'
));
```

## Local Development

Use VIP Local Dev Environment

**VIP Local Dev Environment:**
```bash
npm install -g @automattic/vip
vip dev-env create --slug=mysite
vip dev-env start
```

**Sync from production:**
```bash
vip @mysite.production media pull
vip @mysite.production db pull
```

See [VIP Local Development documentation](https://docs.wpvip.com/local-development/) for complete setup instructions.

## Deployment Workflow

### Using GitHub Integration

VIP deploys automatically from GitHub branches:

1. **Development**: `develop` branch → deployed to `develop` environment
2. **Staging**: `master`/`main` branch → deployed to `production` environment
3. **Production**: Manual promotion via VIP Dashboard

**Commit and push:**
```bash
git add .
git commit -m "feat: add custom post type"
git push origin develop
```

Monitor deployment in VIP Dashboard.

### Using VIP-CLI

**Deploy code:**
```bash
# Deploy to environment
vip @mysite.develop deploy

# Check deployment status
vip @mysite.develop deploy list
```

**Run WP-CLI commands:**
```bash
# Clear cache
vip @mysite.production wp cache flush

# Run custom command
vip @mysite.production wp post list --post_type=page
```

### Pre-Deployment Checklist

Before deploying, verify:

- [ ] Code passes `phpcs` with WordPress VIP ruleset
- [ ] No restricted functions used
- [ ] All external requests are cached
- [ ] Database queries are optimized
- [ ] User inputs are sanitized
- [ ] Outputs are escaped
- [ ] No file system writes
- [ ] Tested in local VIP environment
- [ ] No `var_dump()` or debugging code
- [ ] Error logging uses `error_log()` only

## Code Review Process

VIP runs automated scans (`vip-go-ci`) on every commit, checking for restricted functions, direct database queries, uncached external requests, and security issues. All code requires review before production. Submit PRs with clear descriptions, address automated findings, and merge after approval.

## Code Scanning

Use PHPCS with WordPress VIP Coding Standards:

```bash
composer require --dev automattic/vipwpcs
phpcs --standard=WordPress-VIP-Go .
```

## Common Patterns

### Custom Post Types

```php
function clientname_register_cpt() {
    register_post_type('clientname_resource', [
        'labels' => [
            'name' => 'Resources',
            'singular_name' => 'Resource',
        ],
        'public' => true,
        'has_archive' => true,
        'supports' => ['title', 'editor', 'thumbnail'],
        'show_in_rest' => true,
    ]);
}
add_action('init', 'clientname_register_cpt');
```

### REST API Endpoints

```php
function clientname_register_api_routes() {
    register_rest_route('clientname/v1', '/data/(?P<id>\d+)', [
        'methods' => 'GET',
        'callback' => 'clientname_get_data',
        'permission_callback' => '__return_true',
        'args' => [
            'id' => [
                'validate_callback' => function($param) {
                    return is_numeric($param);
                }
            ],
        ],
    ]);
}
add_action('rest_api_init', 'clientname_register_api_routes');

function clientname_get_data($request) {
    $id = $request['id'];
    // Return data
    return rest_ensure_response(['data' => $data]);
}
```

### Cron Jobs

```php
// Register cron event
function clientname_schedule_cron() {
    if (!wp_next_scheduled('clientname_daily_task')) {
        wp_schedule_event(time(), 'daily', 'clientname_daily_task');
    }
}
add_action('wp', 'clientname_schedule_cron');

// Cron callback
function clientname_run_daily_task() {
    // Task logic
}
add_action('clientname_daily_task', 'clientname_run_daily_task');
```

## Enterprise Search

WordPress VIP includes Enterprise Search powered by Elasticsearch via the ElasticPress plugin. Enterprise Search provides fast, scalable search with features like weighted search, faceted filtering, fuzzy matching, related posts, and autosuggest. Once enabled in the VIP Dashboard, ElasticPress automatically handles WordPress search queries. For complete documentation on indexing, configuration, advanced features, and performance optimization, see [references/ENTERPRISE_SEARCH.md](references/ENTERPRISE_SEARCH.md)

## Troubleshooting

### Common Issues

**White screen/500 error:**
- Check PHP error logs: `vip @mysite.env logs php`
- Verify no fatal errors in recent code
- Check for memory limit issues

**Performance degradation:**
- Review New Relic in VIP Dashboard
- Check for N+1 queries
- Verify object caching is working
- Look for uncached external requests

**Cache not clearing:**
```bash
# Clear all caches
vip @mysite.production wp cache flush

# Purge page cache for specific URL
vip @mysite.production wp vip-go purge-url "https://example.com/page"
```

**Plugin conflicts:**
- Deactivate recent plugins
- Test in local environment
- Check plugin compatibility with VIP

## Additional Resources

- For complete WordPress VIP function reference, see [references/VIP_FUNCTIONS.md](references/VIP_FUNCTIONS.md)
- For deployment examples and workflows, see [references/DEPLOYMENT_EXAMPLES.md](references/DEPLOYMENT_EXAMPLES.md)
- For performance optimization patterns, see [references/PERFORMANCE.md](references/PERFORMANCE.md)
- For Enterprise Search details and advanced patterns, see [references/ENTERPRISE_SEARCH.md](references/ENTERPRISE_SEARCH.md)

## Key Reminders

1. **No file system writes** - use object cache or VIP's storage
2. **Cache external requests** - always use transients or object cache
3. **Prefix everything** - functions, classes, constants
4. **Prepare queries** - always use `$wpdb->prepare()`
5. **Test locally** - use VIP Local Dev Environment
6. **Monitor scans** - address automated findings before review
7. **Follow standards** - WordPress VIP Coding Standards (PHPCS)
