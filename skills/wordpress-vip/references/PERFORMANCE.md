# WordPress VIP Performance Optimization

This document provides detailed patterns and strategies for optimizing performance on WordPress VIP.

## Query Optimization

### Identifying Query Problems

#### Using Query Monitor Locally

```php
// Install Query Monitor in local dev environment
// Activate plugin and review the admin bar

// Look for:
// - Duplicate queries
// - Queries in loops
// - Slow queries (>0.1s)
// - High query counts (>100 per page)
```

#### Common Query Anti-Patterns

**Anti-Pattern 1: Queries in Loops (N+1)**

```php
// BAD - N queries
$posts = get_posts(['posts_per_page' => 100]);
foreach ($posts as $post) {
    $meta = get_post_meta($post->ID, 'custom_field', true); // Query!
}

// GOOD - 2 queries
$posts = get_posts(['posts_per_page' => 100]);
$post_ids = wp_list_pluck($posts, 'ID');

// Prime meta cache
update_meta_cache('post', $post_ids);

foreach ($posts as $post) {
    $meta = get_post_meta($post->ID, 'custom_field', true); // From cache
}
```

**Anti-Pattern 2: Unbounded Queries**

```php
// BAD - Gets ALL posts
$posts = get_posts(['posts_per_page' => -1]);

// GOOD - Limit results
$posts = get_posts(['posts_per_page' => 100]);

// BETTER - Paginate for large datasets
function process_all_posts() {
    $page = 1;
    do {
        $posts = get_posts([
            'posts_per_page' => 100,
            'paged' => $page,
            'fields' => 'ids', // Only IDs if that's enough
        ]);
        
        foreach ($posts as $post_id) {
            process_post($post_id);
        }
        
        $page++;
    } while (count($posts) > 0);
}
```

**Anti-Pattern 3: Expensive Taxonomy Queries**

```php
// BAD - Multiple taxonomy queries
$categories = get_the_terms($post_id, 'category');
$tags = get_the_terms($post_id, 'post_tag');
$custom = get_the_terms($post_id, 'custom_taxonomy');

// GOOD - Single query for all taxonomies
$all_terms = wp_get_post_terms($post_id, ['category', 'post_tag', 'custom_taxonomy']);

// Group by taxonomy
$terms_by_tax = [];
foreach ($all_terms as $term) {
    $terms_by_tax[$term->taxonomy][] = $term;
}
```

### WP_Query Optimization

#### Use Efficient Parameters

```php
// Only get what you need
$query = new WP_Query([
    'post_type' => 'post',
    'posts_per_page' => 20,
    
    // Get only IDs if you don't need full post objects
    'fields' => 'ids',
    
    // Or get ID and parent only
    'fields' => 'id=>parent',
    
    // Disable counting if you don't need pagination info
    'no_found_rows' => true,
    
    // Disable term ordering if not needed
    'update_post_term_cache' => false,
    
    // Disable meta cache update if not needed
    'update_post_meta_cache' => false,
]);
```

#### Optimize Meta Queries

```php
// BAD - Multiple meta queries
$query = new WP_Query([
    'meta_query' => [
        ['key' => 'field1', 'value' => 'value1'],
        ['key' => 'field2', 'value' => 'value2'],
        ['key' => 'field3', 'value' => 'value3'],
    ],
]);

// BETTER - Use single meta key with serialized data
// Or store as custom table
// Or use taxonomy instead of meta for filterable data

// GOOD - For checkbox filters, use taxonomy
// Convert meta fields to custom taxonomy
register_taxonomy('features', 'product', [
    'labels' => ['name' => 'Features'],
    'public' => true,
]);

// Query by taxonomy (much faster)
$query = new WP_Query([
    'post_type' => 'product',
    'tax_query' => [
        [
            'taxonomy' => 'features',
            'field' => 'slug',
            'terms' => ['feature-1', 'feature-2'],
            'operator' => 'AND',
        ],
    ],
]);
```

## Object Cache Strategy

### Cache Layers

WordPress VIP provides multiple cache layers:

1. **Object Cache (Memcached)**: Fast, temporary
2. **Page Cache (Edge Cache)**: Full page caching
3. **CDN Cache**: Static assets

### Effective Object Caching

#### Cache Keys Best Practices

```php
// Use specific, versioned keys
$cache_key = 'clientname_v2_user_' . $user_id . '_data';

// Include all query variables in key
$cache_key = md5(serialize([
    'clientname_query',
    $post_type,
    $tax_query,
    $meta_query,
    $paged,
]));

// Use cache groups for organization
wp_cache_set($cache_key, $data, 'clientname_users', HOUR_IN_SECONDS);
```

#### Cache Invalidation Strategy

```php
// Strategy 1: Time-based expiration
wp_cache_set('data', $value, 'group', 15 * MINUTE_IN_SECONDS);

// Strategy 2: Event-based invalidation
function clientname_clear_user_cache($user_id) {
    wp_cache_delete("user_{$user_id}_data", 'clientname_users');
}
add_action('profile_update', 'clientname_clear_user_cache');

// Strategy 3: Version-based cache busting
$cache_version = get_option('clientname_cache_version', 1);
$cache_key = "data_v{$cache_version}_{$id}";

// Increment version to invalidate all
function clientname_clear_all_caches() {
    $version = get_option('clientname_cache_version', 1);
    update_option('clientname_cache_version', $version + 1);
}
```

#### Caching Complex Data

```php
function clientname_get_dashboard_data($user_id) {
    $cache_key = "dashboard_{$user_id}";
    $data = wp_cache_get($cache_key, 'clientname_dashboard');
    
    if (false === $data) {
        // Expensive operations
        $data = [
            'posts' => clientname_get_user_posts($user_id),
            'stats' => clientname_calculate_stats($user_id),
            'activity' => clientname_get_activity($user_id),
        ];
        
        // Cache for 5 minutes
        wp_cache_set($cache_key, $data, 'clientname_dashboard', 5 * MINUTE_IN_SECONDS);
    }
    
    return $data;
}
```

### Fragment Caching

Cache parts of templates:

```php
// In template file
function clientname_render_sidebar() {
    $cache_key = 'sidebar_html';
    $html = wp_cache_get($cache_key, 'clientname_fragments');
    
    if (false === $html) {
        ob_start();
        ?>
        <aside class="sidebar">
            <?php 
            // Expensive sidebar logic
            dynamic_sidebar('sidebar-1');
            clientname_render_popular_posts();
            ?>
        </aside>
        <?php
        $html = ob_get_clean();
        
        wp_cache_set($cache_key, $html, 'clientname_fragments', HOUR_IN_SECONDS);
    }
    
    echo $html;
}
```

## Page Cache Optimization

### Cache-Friendly Code Patterns

#### Separate Dynamic Content

```php
// BAD - Breaks page caching
<div class="header">
    Welcome, <?php echo wp_get_current_user()->display_name; ?>
</div>

// GOOD - Hydrate via JavaScript
<div class="header">
    Welcome, <span id="user-name" data-user-id="<?php echo get_current_user_id(); ?>"></span>
</div>

<script>
// Fetch and populate user-specific data
fetch('/wp-json/clientname/v1/user-info')
    .then(res => res.json())
    .then(data => {
        document.getElementById('user-name').textContent = data.name;
    });
</script>
```

#### Use REST API for Dynamic Data

```php
// Register REST endpoint for user-specific data
function clientname_register_user_endpoint() {
    register_rest_route('clientname/v1', '/user-info', [
        'methods' => 'GET',
        'callback' => 'clientname_get_user_info',
        'permission_callback' => 'is_user_logged_in',
    ]);
}
add_action('rest_api_init', 'clientname_register_user_endpoint');

function clientname_get_user_info() {
    $user = wp_get_current_user();
    
    // Cache per user
    $cache_key = 'user_info_' . $user->ID;
    $data = wp_cache_get($cache_key, 'clientname_api');
    
    if (false === $data) {
        $data = [
            'name' => $user->display_name,
            'email' => $user->user_email,
            'avatar' => get_avatar_url($user->ID),
        ];
        
        wp_cache_set($cache_key, $data, 'clientname_api', 5 * MINUTE_IN_SECONDS);
    }
    
    return rest_ensure_response($data);
}
```

### Vary Cache by Parameters

```php
// Set Vary header for different cached versions
function clientname_set_cache_headers() {
    // Cache varies by user role
    if (is_user_logged_in()) {
        header('Vary: Cookie');
    }
    
    // Cache varies by language
    if (function_exists('pll_current_language')) {
        header('Vary: Accept-Language');
    }
}
add_action('send_headers', 'clientname_set_cache_headers');
```

## External Request Optimization

### Caching External APIs

```php
function clientname_fetch_api($endpoint, $params = []) {
    $cache_key = md5('api_' . $endpoint . serialize($params));
    $data = wp_cache_get($cache_key, 'clientname_api');
    
    if (false === $data) {
        $url = 'https://api.example.com' . $endpoint;
        
        if (!empty($params)) {
            $url = add_query_arg($params, $url);
        }
        
        $response = vip_safe_wp_remote_get(
            $url,
            false, // fallback
            3,     // threshold
            3,     // timeout
            20     // retry
        );
        
        if (is_wp_error($response)) {
            error_log('API error: ' . $response->get_error_message());
            return false;
        }
        
        $data = json_decode(wp_remote_retrieve_body($response), true);
        
        // Cache for 15 minutes
        wp_cache_set($cache_key, $data, 'clientname_api', 15 * MINUTE_IN_SECONDS);
    }
    
    return $data;
}
```

### Fallback for Failed Requests

```php
function clientname_fetch_with_fallback($url) {
    $cache_key = md5('api_' . $url);
    $fresh_key = $cache_key . '_fresh';
    $stale_key = $cache_key . '_stale';
    
    // Try to get fresh data
    $data = wp_cache_get($fresh_key, 'clientname_api');
    
    if (false === $data) {
        // Fetch new data
        $response = vip_safe_wp_remote_get($url, false, 3, 3, 20);
        
        if (!is_wp_error($response)) {
            $data = json_decode(wp_remote_retrieve_body($response), true);
            
            // Store fresh copy (5 minutes)
            wp_cache_set($fresh_key, $data, 'clientname_api', 5 * MINUTE_IN_SECONDS);
            
            // Store stale copy (1 hour) as backup
            wp_cache_set($stale_key, $data, 'clientname_api', HOUR_IN_SECONDS);
        } else {
            // Use stale data if available
            $data = wp_cache_get($stale_key, 'clientname_api');
            
            if (false === $data) {
                error_log('API completely unavailable: ' . $url);
                return false;
            }
            
            error_log('Using stale API data for: ' . $url);
        }
    }
    
    return $data;
}
```

## Asset Optimization

### Conditional Asset Loading

```php
// Only load assets on pages that need them
function clientname_enqueue_assets() {
    // Load globally
    wp_enqueue_style('clientname-main', get_stylesheet_uri(), [], '1.0.0');
    
    // Load conditionally
    if (is_singular('resource')) {
        wp_enqueue_script(
            'clientname-resource',
            get_template_directory_uri() . '/js/resource.js',
            ['jquery'],
            '1.0.0',
            true
        );
    }
    
    if (is_page_template('template-contact.php')) {
        wp_enqueue_script(
            'clientname-contact-form',
            get_template_directory_uri() . '/js/contact-form.js',
            [],
            '1.0.0',
            true
        );
    }
}
add_action('wp_enqueue_scripts', 'clientname_enqueue_assets');
```

### Defer Non-Critical JavaScript

```php
// Add defer attribute to scripts
function clientname_defer_scripts($tag, $handle) {
    $defer_scripts = [
        'clientname-analytics',
        'clientname-social',
        'clientname-chat',
    ];
    
    if (in_array($handle, $defer_scripts, true)) {
        return str_replace(' src', ' defer src', $tag);
    }
    
    return $tag;
}
add_filter('script_loader_tag', 'clientname_defer_scripts', 10, 2);
```

### Inline Critical CSS

```php
// Inline critical CSS to eliminate render-blocking
function clientname_inline_critical_css() {
    ?>
    <style id="critical-css">
        /* Critical above-the-fold CSS */
        body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
        .header { background: #fff; padding: 20px; }
        /* ... more critical styles ... */
    </style>
    <?php
}
add_action('wp_head', 'clientname_inline_critical_css', 1);
```

## Database Optimization

### Efficient Term Queries

```php
// BAD - Multiple queries
foreach ($post_ids as $post_id) {
    $terms = wp_get_post_terms($post_id, 'category');
}

// GOOD - Prime term cache
update_object_term_cache($post_ids, 'post');

foreach ($post_ids as $post_id) {
    $terms = wp_get_post_terms($post_id, 'category'); // From cache
}
```

### Batch Operations

```php
function clientname_bulk_update_posts($post_ids, $meta_key, $meta_value) {
    // Disable counting during bulk operation
    wp_defer_term_counting(true);
    wp_defer_comment_counting(true);
    
    foreach ($post_ids as $post_id) {
        update_post_meta($post_id, $meta_key, $meta_value);
        
        // Clean up memory every 500 posts
        if (0 === array_search($post_id, $post_ids) % 500) {
            vip_inmemory_cleanup();
        }
    }
    
    // Re-enable counting
    wp_defer_term_counting(false);
    wp_defer_comment_counting(false);
}
```

## Monitoring and Profiling

### Custom Performance Tracking

```php
function clientname_track_performance($label, $callback) {
    $start_time = microtime(true);
    $start_memory = memory_get_usage();
    
    $result = $callback();
    
    $end_time = microtime(true);
    $end_memory = memory_get_usage();
    
    $duration = ($end_time - $start_time) * 1000; // ms
    $memory = ($end_memory - $start_memory) / 1024 / 1024; // MB
    
    if ($duration > 100 || $memory > 1) {
        error_log(sprintf(
            'Performance: %s - %.2fms, %.2fMB',
            $label,
            $duration,
            $memory
        ));
    }
    
    return $result;
}

// Usage
$data = clientname_track_performance('Expensive Query', function() {
    return get_posts(['posts_per_page' => 1000]);
});
```

### Slow Query Logging

```php
// Log slow queries
add_filter('log_query_custom_data', function($query_data, $query) {
    // Log queries taking >50ms
    if ($query_data['time'] > 0.05) {
        error_log(sprintf(
            'Slow query (%.3fs): %s',
            $query_data['time'],
            $query
        ));
    }
    return $query_data;
}, 10, 2);
```

## Performance Checklist

Use this checklist when optimizing:

```markdown
## Query Optimization
- [ ] No queries in loops (N+1)
- [ ] Queries have limits (posts_per_page)
- [ ] Using 'fields' parameter when only IDs needed
- [ ] Meta cache primed for bulk operations
- [ ] Term cache primed for bulk operations
- [ ] No unbounded queries (posts_per_page => -1)

## Caching
- [ ] Expensive operations cached
- [ ] External API calls cached
- [ ] Cache keys include all variables
- [ ] Cache expiration appropriate
- [ ] Cache invalidation on updates
- [ ] Fragment caching for expensive templates

## Page Cache
- [ ] User-specific content separated
- [ ] Dynamic content via AJAX/REST
- [ ] Appropriate Vary headers
- [ ] No logged-in-only content in cached pages

## Assets
- [ ] JavaScript deferred when possible
- [ ] Critical CSS inlined
- [ ] Assets loaded conditionally
- [ ] Images lazy loaded
- [ ] No blocking resources

## Database
- [ ] Batch operations deferred
- [ ] Memory cleanup in large operations
- [ ] Indexes on custom tables
- [ ] No slow queries (>100ms)
```
