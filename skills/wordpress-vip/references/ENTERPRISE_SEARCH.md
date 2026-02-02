# WordPress VIP Enterprise Search

Comprehensive guide to using Enterprise Search (Elasticsearch via ElasticPress) on WordPress VIP.

## Overview

WordPress VIP Enterprise Search is powered by Elasticsearch and integrated through the ElasticPress plugin. It provides:

- Fast, scalable search across all content
- Advanced filtering and faceted search
- Fuzzy matching and typo tolerance
- Related content recommendations
- Real-time indexing
- Search analytics capabilities

## Activation

Enterprise Search is enabled per-environment through the VIP Dashboard.

### Verify Activation

```php
// Check if ElasticPress is active
if (function_exists('ep_is_activated') && ep_is_activated()) {
    // Enterprise Search is active
}

// Check if Elasticsearch is available
if (function_exists('ep_elasticsearch_alive') && ep_elasticsearch_alive()) {
    // Elasticsearch is responding
}
```

## Indexing Content

### Initial Setup

After enabling Enterprise Search, index your content:

```bash
# Full reindex with setup
vip @mysite.production wp elasticpress index --setup

# This will:
# - Delete existing index
# - Create new index with mappings
# - Index all indexable content
```

### Selective Indexing

```bash
# Index specific post type
vip @mysite.production wp elasticpress index --post-type=post --setup

# Index multiple post types
vip @mysite.production wp elasticpress index --post-type=post,page,product --setup

# Index single post
vip @mysite.production wp elasticpress index --post-ids=123

# Index multiple posts
vip @mysite.production wp elasticpress index --post-ids=123,456,789

# Index without bulk processing (for large sites)
vip @mysite.production wp elasticpress index --setup --nobulk
```

### Programmatic Indexing

```php
// Index a single post
if (function_exists('ep_index_post')) {
    ep_index_post($post_id);
}

// Delete post from index
if (function_exists('ep_delete_post')) {
    ep_delete_post($post_id);
}

// Trigger reindex after bulk operations
function clientname_reindex_content($post_ids) {
    if (!function_exists('ep_index_post')) {
        return;
    }
    
    foreach ($post_ids as $post_id) {
        ep_index_post($post_id);
        
        // Clean up memory for large batches
        if (0 === array_search($post_id, $post_ids) % 100) {
            vip_inmemory_cleanup();
        }
    }
}
```

## Search Configuration

### Indexable Post Types

Control which post types are indexed:

```php
// Add custom post type to index
add_filter('ep_indexable_post_types', function($post_types) {
    $post_types['resource'] = 'resource';
    $post_types['product'] = 'product';
    return $post_types;
});

// Exclude post type from index
add_filter('ep_indexable_post_types', function($post_types) {
    unset($post_types['attachment']);
    return $post_types;
});
```

### Indexable Post Statuses

```php
// Add custom post status to index
add_filter('ep_indexable_post_status', function($statuses) {
    $statuses[] = 'custom_status';
    return $statuses;
});
```

### Searchable Fields

Configure which fields are searchable:

```php
// Add custom fields to search
add_filter('ep_prepare_meta_allowed_protected_keys', function($keys) {
    $keys[] = '_custom_field';
    $keys[] = '_product_sku';
    $keys[] = '_event_date';
    return $keys;
});

// Define searchable fields in query
$query = new WP_Query([
    's' => 'search term',
    'search_fields' => [
        'post_title',
        'post_content',
        'post_excerpt',
        'author_name',
        'taxonomies' => ['category', 'post_tag'],
        'meta' => ['_custom_field', '_product_sku'],
    ],
]);
```

## Search Weighting

Boost relevance by adjusting field weights:

```php
add_filter('ep_weighting_configuration', function($config, $post_type) {
    // Title is most important
    $config[$post_type]['post_title']['weight'] = 5;
    $config[$post_type]['post_title']['enabled'] = true;
    
    // Excerpt is moderately important
    $config[$post_type]['post_excerpt']['weight'] = 3;
    $config[$post_type]['post_excerpt']['enabled'] = true;
    
    // Content is baseline
    $config[$post_type]['post_content']['weight'] = 1;
    $config[$post_type]['post_content']['enabled'] = true;
    
    // Custom fields
    $config[$post_type]['meta']['_custom_field']['weight'] = 2;
    $config[$post_type]['meta']['_custom_field']['enabled'] = true;
    
    // Taxonomy terms
    $config[$post_type]['terms']['category']['weight'] = 2;
    $config[$post_type]['terms']['post_tag']['weight'] = 1;
    
    return $config;
}, 10, 2);
```

## Advanced Query Features

### Fuzzy Matching

Enable typo tolerance:

```php
add_filter('ep_formatted_args', function($formatted_args, $args) {
    if (!empty($args['s'])) {
        // AUTO fuzziness (0-2 edits based on term length)
        $formatted_args['query']['bool']['should'][0]['multi_match']['fuzziness'] = 'AUTO';
        
        // Or set specific edit distance (0, 1, or 2)
        // $formatted_args['query']['bool']['should'][0]['multi_match']['fuzziness'] = 1;
    }
    
    return $formatted_args;
}, 10, 2);
```

### Boosting Recent Content

Give more weight to newer posts:

```php
add_filter('ep_formatted_args', function($formatted_args, $args) {
    // Boost posts from last 30 days
    $formatted_args['query']['function_score'] = [
        'query' => $formatted_args['query'],
        'functions' => [
            [
                'gauss' => [
                    'post_date' => [
                        'origin' => 'now',
                        'scale' => '30d',
                        'decay' => 0.5,
                    ],
                ],
            ],
        ],
        'boost_mode' => 'multiply',
    ];
    
    return $formatted_args;
}, 10, 2);
```

### Highlighting Search Terms

Show where search terms appear in results:

```php
add_filter('ep_formatted_args', function($formatted_args, $args) {
    if (!empty($args['s'])) {
        $formatted_args['highlight'] = [
            'fields' => [
                'post_title' => [
                    'number_of_fragments' => 0,
                ],
                'post_content' => [
                    'fragment_size' => 150,
                    'number_of_fragments' => 3,
                ],
            ],
            'pre_tags' => ['<mark>'],
            'post_tags' => ['</mark>'],
        ];
    }
    
    return $formatted_args;
}, 10, 2);

// Access highlights in template
$highlights = get_post_meta(get_the_ID(), 'elasticsearch_highlight', true);
if (!empty($highlights['post_content'])) {
    echo wp_kses_post($highlights['post_content'][0]);
}
```

## Faceted Search

Build advanced filtering with aggregations:

```php
// Add facets/aggregations to search
add_filter('ep_formatted_args', function($formatted_args, $args) {
    if (!empty($args['s']) || !empty($args['facets'])) {
        $formatted_args['aggs'] = [
            // Category facet
            'categories' => [
                'terms' => [
                    'field' => 'terms.category.slug',
                    'size' => 20,
                ],
            ],
            // Tag facet
            'tags' => [
                'terms' => [
                    'field' => 'terms.post_tag.slug',
                    'size' => 20,
                ],
            ],
            // Author facet
            'authors' => [
                'terms' => [
                    'field' => 'post_author.id',
                    'size' => 10,
                ],
            ],
            // Date range facet
            'date_ranges' => [
                'date_range' => [
                    'field' => 'post_date',
                    'ranges' => [
                        ['from' => 'now-7d/d', 'to' => 'now', 'key' => 'last_week'],
                        ['from' => 'now-30d/d', 'to' => 'now', 'key' => 'last_month'],
                        ['from' => 'now-1y/y', 'to' => 'now', 'key' => 'last_year'],
                    ],
                ],
            ],
            // Price range (for e-commerce)
            'price_ranges' => [
                'range' => [
                    'field' => 'meta._price.double',
                    'ranges' => [
                        ['to' => 50, 'key' => 'under_50'],
                        ['from' => 50, 'to' => 100, 'key' => '50_to_100'],
                        ['from' => 100, 'to' => 200, 'key' => '100_to_200'],
                        ['from' => 200, 'key' => 'over_200'],
                    ],
                ],
            ],
        ];
    }
    
    return $formatted_args;
}, 10, 2);

// Access aggregation results
function clientname_get_search_facets($query) {
    if (!$query->elasticsearch_success) {
        return [];
    }
    
    $response = $query->elasticsearch_response;
    $facets = [];
    
    if (!empty($response['aggregations'])) {
        foreach ($response['aggregations'] as $key => $agg) {
            if (!empty($agg['buckets'])) {
                $facets[$key] = $agg['buckets'];
            }
        }
    }
    
    return $facets;
}

// Display facets
add_action('pre_get_posts', function($query) {
    if ($query->is_search() && !is_admin()) {
        $query->set('facets', true);
        
        add_action('wp_footer', function() use ($query) {
            $facets = clientname_get_search_facets($query);
            
            if (!empty($facets['categories'])) {
                echo '<div class="search-facets">';
                echo '<h3>Filter by Category</h3>';
                foreach ($facets['categories'] as $bucket) {
                    printf(
                        '<a href="?s=%s&category=%s">%s (%d)</a>',
                        urlencode(get_search_query()),
                        esc_attr($bucket['key']),
                        esc_html($bucket['key']),
                        $bucket['doc_count']
                    );
                }
                echo '</div>';
            }
        });
    }
});
```

## Autosuggest

Implement search autocomplete:

```php
// Enable autosuggest
add_filter('ep_autosuggest_enabled', '__return_true');

// Configure autosuggest
add_filter('ep_autosuggest_options', function($options) {
    return [
        'post_type' => ['post', 'page', 'product'],
        'endpoint_url' => home_url('/wp-json/elasticpress/v1/autosuggest'),
        'selector' => 'input[type="search"]',
        'trigger_ga_event' => false,
        'addSearchTermHeader' => true,
    ];
});

// Customize autosuggest results
add_filter('ep_autosuggest_http_headers', function($headers) {
    $headers['X-Search-Term'] = isset($_GET['s']) ? sanitize_text_field($_GET['s']) : '';
    return $headers;
});
```

Frontend JavaScript:

```javascript
// Basic autosuggest implementation
jQuery(document).ready(function($) {
    var searchInput = $('input[type="search"]');
    var resultsContainer = $('<div class="autosuggest-results"></div>');
    searchInput.after(resultsContainer);
    
    var timeout;
    searchInput.on('input', function() {
        clearTimeout(timeout);
        var term = $(this).val();
        
        if (term.length < 3) {
            resultsContainer.hide();
            return;
        }
        
        timeout = setTimeout(function() {
            $.get('/wp-json/elasticpress/v1/autosuggest', {
                s: term,
                per_page: 5
            }, function(response) {
                resultsContainer.empty();
                
                if (response.length > 0) {
                    response.forEach(function(item) {
                        resultsContainer.append(
                            '<a href="' + item.url + '">' + 
                            item.title + '</a>'
                        );
                    });
                    resultsContainer.show();
                }
            });
        }, 300);
    });
});
```

## Related Content

### Related Posts

```php
function clientname_get_related_posts($post_id = null, $limit = 5) {
    if (!$post_id) {
        $post_id = get_the_ID();
    }
    
    if (!function_exists('ep_find_related')) {
        // Fallback to category-based related posts
        return clientname_fallback_related_posts($post_id, $limit);
    }
    
    $related = ep_find_related($post_id, $limit, [
        'post_type' => get_post_type($post_id),
    ]);
    
    return $related;
}

// Template usage
$related_posts = clientname_get_related_posts(get_the_ID(), 4);
foreach ($related_posts as $post) {
    echo '<article>';
    echo '<h3>' . esc_html($post->post_title) . '</h3>';
    echo '<a href="' . get_permalink($post->ID) . '">Read more</a>';
    echo '</article>';
}
```

### Custom Related Logic

```php
add_filter('ep_find_related_args', function($args, $post_id) {
    // Only find related posts from same author
    $post = get_post($post_id);
    $args['author'] = $post->post_author;
    
    // Exclude the current post
    $args['post__not_in'] = [$post_id];
    
    // Boost posts with shared categories
    $categories = wp_get_post_categories($post_id);
    if (!empty($categories)) {
        $args['tax_query'] = [
            [
                'taxonomy' => 'category',
                'field' => 'term_id',
                'terms' => $categories,
            ],
        ];
    }
    
    return $args;
}, 10, 2);
```

## Performance Optimization

### Caching Search Results

```php
function clientname_cached_search($search_term, $args = []) {
    $cache_key = md5('search_' . $search_term . serialize($args));
    $results = wp_cache_get($cache_key, 'clientname_search');
    
    if (false === $results) {
        $default_args = [
            's' => $search_term,
            'posts_per_page' => 20,
        ];
        
        $query_args = wp_parse_args($args, $default_args);
        $query = new WP_Query($query_args);
        
        $results = [
            'posts' => $query->posts,
            'total' => $query->found_posts,
        ];
        
        // Cache for 5 minutes
        wp_cache_set($cache_key, $results, 'clientname_search', 5 * MINUTE_IN_SECONDS);
    }
    
    return $results;
}
```

### Limiting Index Size

```php
// Exclude old content from index
add_filter('ep_indexable_post_status', function($statuses, $post) {
    // Only index posts from last 2 years
    $two_years_ago = strtotime('-2 years');
    $post_date = strtotime($post->post_date);
    
    if ($post_date < $two_years_ago) {
        return [];
    }
    
    return $statuses;
}, 10, 2);

// Exclude specific meta keys from indexing
add_filter('ep_prepare_meta_excluded_public_keys', function($keys) {
    $keys[] = '_edit_lock';
    $keys[] = '_edit_last';
    $keys[] = '_thumbnail_id';
    return $keys;
});
```

### Optimizing Queries

```php
// Disable unnecessary features for better performance
add_filter('ep_integrate', function($integrate, $query) {
    // Only use Elasticsearch for search queries
    if (!$query->is_search()) {
        return false;
    }
    
    return $integrate;
}, 10, 2);

// Limit aggregation size
add_filter('ep_formatted_args', function($formatted_args) {
    if (isset($formatted_args['aggs'])) {
        foreach ($formatted_args['aggs'] as $key => $agg) {
            if (isset($agg['terms'])) {
                // Limit to 10 buckets max
                $formatted_args['aggs'][$key]['terms']['size'] = 10;
            }
        }
    }
    return $formatted_args;
});
```

## Monitoring and Debugging

### Check Index Status

```bash
# Check if site is indexed
vip @mysite.production wp elasticpress status

# Get index stats
vip @mysite.production wp elasticpress stats
```

### Debug Search Queries

```php
// Log Elasticsearch queries
add_filter('ep_do_intercept_request', function($request, $query, $args, $failures) {
    error_log('ES Query: ' . print_r($args, true));
    return $request;
}, 10, 4);

// Log search results
add_action('pre_get_posts', function($query) {
    if ($query->is_search() && !is_admin()) {
        add_filter('the_posts', function($posts) use ($query) {
            error_log(sprintf(
                'Search: "%s" - %d results',
                get_search_query(),
                $query->found_posts
            ));
            return $posts;
        });
    }
});

// Check if query used Elasticsearch
add_action('pre_get_posts', function($query) {
    if ($query->is_search()) {
        add_action('wp_footer', function() use ($query) {
            if (is_admin()) return;
            
            if ($query->elasticsearch_success) {
                echo '<!-- Search powered by Elasticsearch -->';
            } else {
                echo '<!-- Search using MySQL fallback -->';
            }
        });
    }
});
```

### Fallback to MySQL

ElasticPress automatically falls back to MySQL if Elasticsearch is unavailable:

```php
// Detect fallback
add_action('ep_wp_query_search_response', function($response, $query) {
    if (is_wp_error($response)) {
        error_log('Elasticsearch error: ' . $response->get_error_message());
        // Alert monitoring service
    }
}, 10, 2);

// Force MySQL for specific queries
add_filter('ep_skip_query_integration', function($skip, $query) {
    // Don't use Elasticsearch for admin queries
    if (is_admin()) {
        return true;
    }
    
    return $skip;
}, 10, 2);
```

## Common Patterns

### E-commerce Product Search

```php
// Enhanced product search
add_filter('ep_formatted_args', function($formatted_args, $args) {
    if (isset($args['post_type']) && 'product' === $args['post_type']) {
        // Boost in-stock products
        $formatted_args['query']['function_score'] = [
            'query' => $formatted_args['query'],
            'functions' => [
                [
                    'filter' => [
                        'term' => ['meta._stock_status.value' => 'instock'],
                    ],
                    'weight' => 2,
                ],
            ],
        ];
    }
    
    return $formatted_args;
}, 10, 2);
```

### Multi-language Search

```php
// WPML/Polylang integration
add_filter('ep_formatted_args', function($formatted_args, $args) {
    $current_lang = apply_filters('wpml_current_language', null);
    
    if ($current_lang) {
        $formatted_args['post_filter'] = [
            'bool' => [
                'must' => [
                    ['term' => ['lang' => $current_lang]],
                ],
            ],
        ];
    }
    
    return $formatted_args;
}, 10, 2);
```

### Search Analytics

```php
// Track popular searches
function clientname_track_search($search_term, $result_count) {
    global $wpdb;
    
    $table = $wpdb->prefix . 'search_analytics';
    
    $wpdb->insert(
        $table,
        [
            'search_term' => $search_term,
            'result_count' => $result_count,
            'timestamp' => current_time('mysql'),
            'user_id' => get_current_user_id(),
        ],
        ['%s', '%d', '%s', '%d']
    );
}

// Popular searches report
function clientname_get_popular_searches($limit = 10, $days = 30) {
    global $wpdb;
    
    $table = $wpdb->prefix . 'search_analytics';
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    
    return $wpdb->get_results($wpdb->prepare(
        "SELECT search_term, COUNT(*) as search_count
        FROM {$table}
        WHERE timestamp > %s
        GROUP BY search_term
        ORDER BY search_count DESC
        LIMIT %d",
        $since,
        $limit
    ));
}
```

## Best Practices

### Index Management

1. **Reindex after major changes**:
   - Theme changes affecting content structure
   - Plugin updates changing indexed fields
   - Bulk content imports

2. **Use --nobulk for large sites**:
   ```bash
   vip @mysite.production wp elasticpress index --setup --nobulk
   ```

3. **Monitor index health** in VIP Dashboard

### Query Optimization

1. **Cache search results** when possible
2. **Limit result sets** with pagination
3. **Filter by post_type** to reduce scope
4. **Use fields parameter** to only fetch needed data

### Indexing Strategy

1. **Exclude unnecessary post types** (revisions, nav menus)
2. **Limit meta key indexing** to relevant fields only
3. **Consider date-based exclusions** for old content
4. **Index during off-peak hours** for large reindexes

### Search Experience

1. **Implement autosuggest** for better UX
2. **Use fuzzy matching** for typo tolerance
3. **Weight title higher** than content
4. **Provide filters/facets** for refinement
5. **Show search highlights** in results

## Troubleshooting

### Search Not Working

```bash
# Check ElasticPress status
vip @mysite.production wp elasticpress status

# Verify Elasticsearch health
vip @mysite.production wp elasticpress get-cluster-health

# Reindex content
vip @mysite.production wp elasticpress index --setup
```

### Empty Results

1. Check if content is indexed:
   ```bash
   vip @mysite.production wp elasticpress stats
   ```

2. Verify post type is indexable:
   ```php
   add_filter('ep_indexable_post_types', function($types) {
       error_log('Indexable types: ' . print_r($types, true));
       return $types;
   });
   ```

3. Check query integration:
   ```php
   add_filter('ep_integrate', function($integrate, $query) {
       error_log('EP integrate: ' . ($integrate ? 'yes' : 'no'));
       return $integrate;
   }, 10, 2);
   ```

### Slow Queries

1. Reduce aggregation complexity
2. Limit result size
3. Cache results
4. Optimize weighting configuration
5. Review VIP Dashboard for performance metrics

### Index Corruption

```bash
# Delete and recreate index
vip @mysite.production wp elasticpress delete-index
vip @mysite.production wp elasticpress index --setup
```

## Additional Resources

- [ElasticPress Documentation](https://www.elasticpress.io/documentation/)
- [WordPress VIP Search Documentation](https://docs.wpvip.com/technical-references/vip-search/)
- [Elasticsearch Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
