# WordPress VIP Deployment Examples

This document provides practical examples of common deployment scenarios and workflows.

## Example 1: Feature Branch Workflow

### Scenario
Developing a new custom post type feature with REST API endpoints.

### Workflow

```bash
# 1. Create feature branch
git checkout -b feature/custom-post-type

# 2. Develop locally using VIP dev environment
vip dev-env start
# Edit files, test locally at http://mysite.vipdev.lndo.site

# 3. Run code quality checks
composer run phpcs
./scripts/vip-scan.sh

# 4. Commit changes
git add plugins/clientname-cpt/
git commit -m "feat: add Resource custom post type with REST API

- Register custom post type for Resources
- Add custom fields using Fieldmanager
- Create REST API endpoints for filtering
- Add caching for API responses"

# 5. Push to GitHub
git push origin feature/custom-post-type

# 6. Create Pull Request on GitHub
# VIP automated scans run automatically

# 7. Merge to develop branch after review
# Deploys automatically to develop environment

# 8. Test in develop environment
vip @mysite.develop wp post list --post_type=clientname_resource

# 9. Merge to main for production deployment
# Promote to production via VIP Dashboard
```

## Example 2: Hotfix Deployment

### Scenario
Critical bug fix needs immediate production deployment.

### Workflow

```bash
# 1. Create hotfix branch from main
git checkout main
git pull origin main
git checkout -b hotfix/critical-cache-bug

# 2. Make minimal fix
# Edit the specific file

# 3. Test locally
vip dev-env start
# Verify fix works

# 4. Commit with clear description
git add plugins/clientname-core/includes/caching.php
git commit -m "fix: prevent cache stampede on homepage

Issue: Multiple simultaneous requests causing cache regeneration
Solution: Add lock mechanism using transients"

# 5. Push and create PR
git push origin hotfix/critical-cache-bug
# Create PR targeting main branch

# 6. Fast-track review
# Get review from team lead
# Address any automated scan findings

# 7. Merge to main
git checkout main
git merge hotfix/critical-cache-bug
git push origin main

# 8. Monitor deployment
# Check VIP Dashboard for deployment status
# Monitor error logs
vip @mysite.production logs php --tail

# 9. Verify fix in production
# Test the affected functionality
# Check New Relic for performance impact

# 10. Backport to develop
git checkout develop
git merge hotfix/critical-cache-bug
git push origin develop
```

## Example 3: Plugin Addition

### Scenario
Adding a new third-party plugin to the VIP environment.

### Workflow

#### Option A: VIP Shared Plugin

```php
// In client-mu-plugins/vip-config/vip-config.php

// Check if plugin is in VIP shared plugins
// List: https://docs.wpvip.com/plugins/

// Activate the plugin
wpcom_vip_load_plugin('advanced-custom-fields');
wpcom_vip_load_plugin('wordpress-seo');

// Configure plugin settings
if (function_exists('acf_update_setting')) {
    acf_update_setting('show_admin', true);
}
```

```bash
# Commit and deploy
git add client-mu-plugins/vip-config/vip-config.php
git commit -m "feat: add ACF and Yoast SEO plugins"
git push origin develop
```

#### Option B: Custom Plugin

```bash
# 1. Add plugin to plugins/ directory
mkdir -p plugins/custom-plugin
# Copy plugin files

# 2. Review plugin code for VIP compatibility
./scripts/vip-scan.sh plugins/custom-plugin

# 3. Fix any VIP violations
# Edit plugin files to use VIP-compatible functions

# 4. Test locally
vip dev-env start
# Verify plugin functionality

# 5. Commit plugin
git add plugins/custom-plugin/
git commit -m "feat: add custom analytics plugin

- Tracks user interactions
- Uses object cache for performance
- All external requests cached
- VIP compliance verified"

# 6. Deploy to develop environment
git push origin develop

# 7. Request VIP review (first deployment)
# Submit support ticket with plugin details
# Wait for approval before production
```

## Example 4: Database Migration

### Scenario
Migrating custom post type data and updating post meta.

### Approach

Create a WP-CLI command for safe execution:

```php
// In client-mu-plugins/commands/class-migration-command.php

if (defined('WP_CLI') && WP_CLI) {
    
    class ClientName_Migration_Command {
        
        /**
         * Migrate old post type to new structure
         *
         * @param array $args
         * @param array $assoc_args
         */
        public function migrate_post_type($args, $assoc_args) {
            $dry_run = isset($assoc_args['dry-run']);
            
            WP_CLI::log('Starting migration...');
            
            $paged = 1;
            $posts_per_page = 100;
            $total_migrated = 0;
            
            do {
                $query = new WP_Query([
                    'post_type' => 'old_post_type',
                    'posts_per_page' => $posts_per_page,
                    'paged' => $paged,
                    'fields' => 'ids',
                ]);
                
                if (!$query->have_posts()) {
                    break;
                }
                
                foreach ($query->posts as $post_id) {
                    if ($dry_run) {
                        WP_CLI::log("Would migrate post ID: $post_id");
                    } else {
                        $this->migrate_single_post($post_id);
                        $total_migrated++;
                    }
                }
                
                WP_CLI::log("Processed batch $paged");
                
                wp_reset_postdata();
                vip_inmemory_cleanup();
                
                $paged++;
                
            } while ($query->have_posts());
            
            WP_CLI::success("Migration complete. Migrated $total_migrated posts.");
        }
        
        private function migrate_single_post($post_id) {
            // Get old meta
            $old_meta = get_post_meta($post_id, 'old_meta_key', true);
            
            // Transform data
            $new_meta = $this->transform_meta($old_meta);
            
            // Update post type
            wp_update_post([
                'ID' => $post_id,
                'post_type' => 'new_post_type',
            ]);
            
            // Update meta
            update_post_meta($post_id, 'new_meta_key', $new_meta);
            
            // Delete old meta
            delete_post_meta($post_id, 'old_meta_key');
        }
        
        private function transform_meta($old_meta) {
            // Transform logic
            return $new_meta;
        }
    }
    
    WP_CLI::add_command('clientname migration', 'ClientName_Migration_Command');
}
```

### Deployment and Execution

```bash
# 1. Deploy migration command
git add client-mu-plugins/commands/
git commit -m "feat: add post type migration command"
git push origin develop

# 2. Test in develop environment (dry run)
vip @mysite.develop wp clientname migration migrate_post_type --dry-run

# 3. Run actual migration in develop
vip @mysite.develop wp clientname migration migrate_post_type

# 4. Verify results
vip @mysite.develop wp post list --post_type=new_post_type

# 5. Clear caches
vip @mysite.develop wp cache flush

# 6. Deploy to production
git checkout main
git merge develop
git push origin main

# 7. Run migration in production
vip @mysite.production wp clientname migration migrate_post_type

# 8. Verify and clear cache
vip @mysite.production wp post list --post_type=new_post_type --post_status=publish
vip @mysite.production wp cache flush
```

## Example 5: Performance Optimization

### Scenario
Site experiencing slow page loads due to inefficient queries.

### Investigation

```bash
# 1. Check New Relic in VIP Dashboard
# Identify slow transactions

# 2. Review logs for slow queries
vip @mysite.production logs php --filter="slow"

# 3. Enable Query Monitor locally
# Install Query Monitor plugin in local dev environment
vip dev-env start
```

### Optimization

```php
// Before: N+1 query problem
// In template file showing posts with authors

foreach ($posts as $post) {
    $author = get_user_by('id', $post->post_author); // Query per post!
    echo $author->display_name;
}

// After: Pre-fetch all authors
$author_ids = wp_list_pluck($posts, 'post_author');
$author_ids = array_unique($author_ids);

$authors = get_users([
    'include' => $author_ids,
]);

$authors_by_id = [];
foreach ($authors as $author) {
    $authors_by_id[$author->ID] = $author;
}

foreach ($posts as $post) {
    $author = $authors_by_id[$post->post_author] ?? null;
    if ($author) {
        echo $author->display_name;
    }
}
```

### Deployment

```bash
# 1. Make changes
git checkout -b perf/optimize-author-queries

# 2. Test locally with Query Monitor
# Verify query count reduction

# 3. Commit and deploy
git add themes/clientname/template-parts/
git commit -m "perf: eliminate N+1 query for post authors

Reduced queries from N to 2 by batch-fetching authors"
git push origin perf/optimize-author-queries

# 4. Deploy to develop and verify with New Relic
# Monitor response times

# 5. Deploy to production
# Monitor New Relic for performance improvements
```

## Example 6: Cache Invalidation Strategy

### Scenario
Content update requires targeted cache clearing.

### Implementation

```php
// Add cache invalidation hooks
function clientname_clear_related_caches($post_id) {
    $post = get_post($post_id);
    
    if ('resource' !== $post->post_type) {
        return;
    }
    
    // Clear post-specific caches
    wp_cache_delete("resource_data_{$post_id}", 'clientname');
    
    // Clear listing caches
    wp_cache_delete('resource_listing_all', 'clientname');
    
    // Clear taxonomy caches
    $terms = wp_get_post_terms($post_id, 'resource_category', ['fields' => 'ids']);
    foreach ($terms as $term_id) {
        wp_cache_delete("resource_listing_term_{$term_id}", 'clientname');
    }
    
    // Purge page cache for specific URLs
    if (function_exists('wpcom_vip_purge_edge_cache_for_url')) {
        wpcom_vip_purge_edge_cache_for_url(get_permalink($post_id));
        wpcom_vip_purge_edge_cache_for_url(home_url('/resources/'));
    }
}
add_action('save_post_resource', 'clientname_clear_related_caches');
add_action('delete_post', 'clientname_clear_related_caches');
```

### Manual Cache Clearing

```bash
# Clear specific URL
vip @mysite.production wp vip-go purge-url "https://example.com/resources/item/"

# Clear pattern
vip @mysite.production wp vip-go purge-url "https://example.com/resources/*"

# Clear object cache
vip @mysite.production wp cache flush

# Clear specific cache group
vip @mysite.production wp cache flush-group clientname
```

## Example 7: Scheduled Task Deployment

### Scenario
Add daily task to sync external data.

### Implementation

```php
// Register cron schedule
function clientname_add_cron_schedules($schedules) {
    $schedules['five_minutes'] = [
        'interval' => 300,
        'display' => __('Every 5 Minutes'),
    ];
    return $schedules;
}
add_filter('cron_schedules', 'clientname_add_cron_schedules');

// Schedule event
function clientname_schedule_sync() {
    if (!wp_next_scheduled('clientname_sync_data')) {
        wp_schedule_event(time(), 'daily', 'clientname_sync_data');
    }
}
add_action('wp', 'clientname_schedule_sync');

// Task callback
function clientname_run_sync() {
    $lock_key = 'clientname_sync_lock';
    
    // Prevent overlapping runs
    if (wp_cache_get($lock_key, 'clientname')) {
        error_log('Sync already running, skipping');
        return;
    }
    
    wp_cache_set($lock_key, true, 'clientname', 600); // 10 min lock
    
    try {
        // Sync logic
        $data = clientname_fetch_external_data();
        clientname_process_data($data);
        
        error_log('Sync completed successfully');
    } catch (Exception $e) {
        error_log('Sync failed: ' . $e->getMessage());
    } finally {
        wp_cache_delete($lock_key, 'clientname');
    }
}
add_action('clientname_sync_data', 'clientname_run_sync');
```

### Testing and Monitoring

```bash
# View scheduled cron events
vip @mysite.develop wp cron event list

# Run cron manually for testing
vip @mysite.develop wp cron event run clientname_sync_data

# Monitor cron execution in logs
vip @mysite.production logs php --filter="Sync completed"

# Check for errors
vip @mysite.production logs php --filter="Sync failed"
```

## Deployment Checklist Template

Use this checklist for all deployments:

```markdown
## Pre-Deployment
- [ ] Code reviewed by team member
- [ ] PHPCS passed with VIP ruleset
- [ ] VIP scan shows no violations
- [ ] Tested in local VIP dev environment
- [ ] No debugging code (var_dump, console.log, etc.)
- [ ] All external requests cached
- [ ] Database queries optimized
- [ ] No file system writes (except temp)

## Deployment
- [ ] Deployed to develop environment
- [ ] Smoke tested in develop
- [ ] Reviewed automated scan results
- [ ] No errors in develop logs
- [ ] Merged to main branch
- [ ] Production deployment started

## Post-Deployment
- [ ] Verified functionality in production
- [ ] Checked error logs for issues
- [ ] Reviewed New Relic for performance
- [ ] Cache cleared if needed
- [ ] Monitored for 15+ minutes
- [ ] Team notified of deployment
```
