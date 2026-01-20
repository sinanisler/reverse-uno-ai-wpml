# WPML REST API Reverse Engineering: Alternative Auto-Translation System

A theoretical reverse engineering guide for WPML (WordPress Multilingual Plugin) REST API to create alternative auto-translation systems for WordPress pages, posts, and strings.

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Reverse Engineering WPML](#reverse-engineering-wpml)
- [Implementation Guide](#implementation-guide)
- [Auto-Translation Strategies](#auto-translation-strategies)
- [API Endpoints](#api-endpoints)
- [Usage Examples](#usage-examples)
- [Advanced Patterns](#advanced-patterns)
- [Security Considerations](#security-considerations)

---

## Overview

This document provides a theoretical framework for reverse engineering WPML's REST API to enable programmatic access to translation data and create alternative auto-translation workflows. This approach is particularly useful when:

- You need to manage translations for multiple countries with different content requirements
- Standard WPML workflows don't fit your programmatic needs
- You want to integrate external translation services or AI-based translation
- You need to access translation relationships via REST API or MCP tools

## The Problem

**WordPress REST API and WPML don't natively expose translation relationships.** 

When you query the WordPress REST API:
- You can't see which posts are translations of each other
- There's no way to identify the translation group (trid)
- Language information is not exposed in API responses
- You can't programmatically manage translations via REST

## The Solution

**Hook into WordPress REST API to expose WPML translation data as custom fields.**

By extending the REST API response with WPML data, we can:
- Access translation relationships programmatically
- Enable REST API and MCP tools to work with multilingual content
- Build alternative auto-translation workflows
- Create custom translation management interfaces

### Fields to Expose

1. **`wpml_current_locale`** - Language code of the current post (e.g., `en`, `es`, `fr`)
2. **`wpml_translations`** - Object containing all translations with:
   - ID
   - Title
   - Permalink
   - Status
3. **`wpml_trid`** - Translation group ID (links all translations together)

---

## Reverse Engineering WPML

### Understanding WPML's Database Structure

WPML stores translation data in custom database tables:

```
wp_icl_translations
├── translation_id (primary key)
├── element_type (post_post, post_page, etc.)
├── element_id (WordPress post ID)
├── trid (translation group ID)
├── language_code (en, es, fr, etc.)
└── source_language_code (null for original, parent lang for translations)

wp_icl_translate (for string translations)
├── tid (translation ID)
├── job_id
├── content_id
├── timestamp
└── field_data (serialized translation data)
```

### Key WPML Functions to Hook Into

```php
// Get current language
apply_filters('wpml_current_language', NULL);

// Get element translations
apply_filters('wpml_element_translations', NULL, $trid, $element_type);

// Get element language details
apply_filters('wpml_element_language_details', NULL, array('element_id' => $post_id, 'element_type' => 'post_' . $post_type));

// Get post language
apply_filters('wpml_post_language_details', NULL, $post_id);

// Get translation ID
apply_filters('wpml_element_trid', NULL, $post_id, 'post_' . $post_type);
```

---

## Implementation Guide

### Step 1: Basic REST API Extension

Create a PHP snippet (via WPCodeBox or functions.php) to expose WPML data:

```php
<?php
/**
 * WPML REST API Extension
 * Exposes translation data in WordPress REST API responses
 */

add_action('rest_api_init', function() {
    // Get all public post types
    $post_types = get_post_types(['public' => true], 'names');
    
    foreach ($post_types as $post_type) {
        // Register wpml_current_locale field
        register_rest_field($post_type, 'wpml_current_locale', [
            'get_callback' => 'wpml_rest_get_current_locale',
            'schema' => [
                'description' => 'Current post language code',
                'type' => 'string',
                'context' => ['view', 'edit']
            ]
        ]);
        
        // Register wpml_translations field
        register_rest_field($post_type, 'wpml_translations', [
            'get_callback' => 'wpml_rest_get_translations',
            'schema' => [
                'description' => 'All translations of this post',
                'type' => 'object',
                'context' => ['view', 'edit']
            ]
        ]);
        
        // Register wpml_trid field
        register_rest_field($post_type, 'wpml_trid', [
            'get_callback' => 'wpml_rest_get_trid',
            'schema' => [
                'description' => 'Translation group ID',
                'type' => 'integer',
                'context' => ['view', 'edit']
            ]
        ]);
    }
});

/**
 * Get current locale for post
 */
function wpml_rest_get_current_locale($object) {
    if (!function_exists('wpml_get_language_information')) {
        return null;
    }
    
    $lang_info = apply_filters('wpml_post_language_details', null, $object['id']);
    return $lang_info ? $lang_info['language_code'] : null;
}

/**
 * Get all translations for post
 */
function wpml_rest_get_translations($object) {
    if (!function_exists('wpml_get_language_information')) {
        return null;
    }
    
    $trid = apply_filters('wpml_element_trid', null, $object['id'], 'post_' . $object['type']);
    
    if (!$trid) {
        return null;
    }
    
    $translations = apply_filters('wpml_get_element_translations', null, $trid, 'post_' . $object['type']);
    
    if (!$translations) {
        return null;
    }
    
    $result = [];
    foreach ($translations as $lang_code => $translation) {
        if ($translation->element_id == $object['id']) {
            continue; // Skip current post
        }
        
        $result[$lang_code] = [
            'id' => $translation->element_id,
            'locale' => $lang_code,
            'title' => get_the_title($translation->element_id),
            'permalink' => get_permalink($translation->element_id),
            'status' => get_post_status($translation->element_id)
        ];
    }
    
    return $result;
}

/**
 * Get translation group ID
 */
function wpml_rest_get_trid($object) {
    if (!function_exists('wpml_get_language_information')) {
        return null;
    }
    
    return apply_filters('wpml_element_trid', null, $object['id'], 'post_' . $object['type']);
}
```

### Step 2: String Translation Support

Extend the system to handle WPML string translations:

```php
<?php
/**
 * WPML String Translation REST API
 */

add_action('rest_api_init', function() {
    register_rest_route('wpml/v1', '/strings', [
        'methods' => 'GET',
        'callback' => 'wpml_rest_get_strings',
        'permission_callback' => function() {
            return current_user_can('edit_posts');
        }
    ]);
    
    register_rest_route('wpml/v1', '/strings/(?P<id>\d+)', [
        'methods' => 'GET',
        'callback' => 'wpml_rest_get_string',
        'permission_callback' => function() {
            return current_user_can('edit_posts');
        }
    ]);
});

function wpml_rest_get_strings($request) {
    global $wpdb;
    
    $context = $request->get_param('context') ?? '';
    $language = $request->get_param('language') ?? '';
    
    $query = "SELECT * FROM {$wpdb->prefix}icl_strings WHERE 1=1";
    
    if ($context) {
        $query .= $wpdb->prepare(" AND context = %s", $context);
    }
    
    if ($language) {
        $query .= $wpdb->prepare(" AND language = %s", $language);
    }
    
    $strings = $wpdb->get_results($query);
    
    return rest_ensure_response($strings);
}

function wpml_rest_get_string($request) {
    global $wpdb;
    
    $id = $request->get_param('id');
    
    $string = $wpdb->get_row($wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}icl_strings WHERE id = %d",
        $id
    ));
    
    if (!$string) {
        return new WP_Error('not_found', 'String not found', ['status' => 404]);
    }
    
    // Get translations
    $translations = $wpdb->get_results($wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}icl_string_translations WHERE string_id = %d",
        $id
    ));
    
    $string->translations = $translations;
    
    return rest_ensure_response($string);
}
```

---

## Auto-Translation Strategies

### Strategy 1: REST API + External Translation Service

```php
<?php
/**
 * Auto-translate posts using external API
 */

add_action('rest_api_init', function() {
    register_rest_route('wpml/v1', '/auto-translate/(?P<id>\d+)', [
        'methods' => 'POST',
        'callback' => 'wpml_auto_translate_post',
        'permission_callback' => function() {
            return current_user_can('edit_posts');
        },
        'args' => [
            'target_languages' => [
                'required' => true,
                'type' => 'array',
                'items' => ['type' => 'string']
            ],
            'translation_service' => [
                'required' => false,
                'type' => 'string',
                'default' => 'google'
            ]
        ]
    ]);
});

function wpml_auto_translate_post($request) {
    $post_id = $request->get_param('id');
    $target_languages = $request->get_param('target_languages');
    $service = $request->get_param('translation_service');
    
    $post = get_post($post_id);
    if (!$post) {
        return new WP_Error('not_found', 'Post not found', ['status' => 404]);
    }
    
    $results = [];
    
    foreach ($target_languages as $target_lang) {
        // Translate title and content
        $translated_title = translate_text($post->post_title, $target_lang, $service);
        $translated_content = translate_text($post->post_content, $target_lang, $service);
        
        // Create translation post
        $translation_id = wp_insert_post([
            'post_title' => $translated_title,
            'post_content' => $translated_content,
            'post_type' => $post->post_type,
            'post_status' => 'draft' // Start as draft for review
        ]);
        
        // Link to WPML
        $trid = apply_filters('wpml_element_trid', null, $post_id, 'post_' . $post->post_type);
        
        if (!$trid) {
            // Create new translation group
            $trid = wpml_create_trid($post_id, $post->post_type);
        }
        
        // Add translation to group
        wpml_add_translation($translation_id, $trid, $target_lang, $post->post_type);
        
        $results[$target_lang] = [
            'id' => $translation_id,
            'status' => 'success',
            'permalink' => get_permalink($translation_id)
        ];
    }
    
    return rest_ensure_response([
        'original_id' => $post_id,
        'translations' => $results
    ]);
}

function translate_text($text, $target_lang, $service = 'google') {
    // Integration with translation services
    switch ($service) {
        case 'google':
            return google_translate($text, $target_lang);
        case 'deepl':
            return deepl_translate($text, $target_lang);
        case 'openai':
            return openai_translate($text, $target_lang);
        default:
            return $text;
    }
}

// Helper functions for WPML
function wpml_create_trid($post_id, $post_type) {
    global $wpdb;
    
    $source_lang = apply_filters('wpml_current_language', null);
    
    // Insert into translations table
    $wpdb->insert(
        $wpdb->prefix . 'icl_translations',
        [
            'element_type' => 'post_' . $post_type,
            'element_id' => $post_id,
            'language_code' => $source_lang,
            'source_language_code' => null
        ]
    );
    
    return $wpdb->get_var($wpdb->prepare(
        "SELECT trid FROM {$wpdb->prefix}icl_translations WHERE element_id = %d AND element_type = %s",
        $post_id,
        'post_' . $post_type
    ));
}

function wpml_add_translation($translation_id, $trid, $target_lang, $post_type) {
    global $wpdb;
    
    $source_lang = apply_filters('wpml_current_language', null);
    
    $wpdb->insert(
        $wpdb->prefix . 'icl_translations',
        [
            'element_type' => 'post_' . $post_type,
            'element_id' => $translation_id,
            'trid' => $trid,
            'language_code' => $target_lang,
            'source_language_code' => $source_lang
        ]
    );
}
```

### Strategy 2: Batch Translation with AI

```php
<?php
/**
 * Batch auto-translate multiple posts
 */

add_action('rest_api_init', function() {
    register_rest_route('wpml/v1', '/batch-translate', [
        'methods' => 'POST',
        'callback' => 'wpml_batch_translate',
        'permission_callback' => function() {
            return current_user_can('manage_options');
        }
    ]);
});

function wpml_batch_translate($request) {
    $post_ids = $request->get_param('post_ids');
    $target_languages = $request->get_param('target_languages');
    $use_ai = $request->get_param('use_ai') ?? true;
    
    // Process in batches to avoid timeouts
    $batch_size = 10;
    $results = [];
    
    foreach (array_chunk($post_ids, $batch_size) as $batch) {
        foreach ($batch as $post_id) {
            $results[$post_id] = wpml_auto_translate_post_internal(
                $post_id,
                $target_languages,
                $use_ai
            );
        }
    }
    
    return rest_ensure_response([
        'success' => true,
        'results' => $results
    ]);
}
```

### Strategy 3: MCP Tool Integration

```javascript
// Example MCP tool for translation management
{
  "name": "wpml_translate",
  "description": "Translate WordPress content using WPML",
  "inputSchema": {
    "type": "object",
    "properties": {
      "post_id": {
        "type": "number",
        "description": "Post ID to translate"
      },
      "target_languages": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Target language codes (en, es, fr, etc.)"
      },
      "auto_translate": {
        "type": "boolean",
        "description": "Use AI auto-translation"
      }
    }
  }
}
```

---

## API Endpoints

### Get Post with Translation Data

```bash
GET /wp-json/wp/v2/posts/{id}
```

**Response:**
```json
{
  "id": 123,
  "title": { "rendered": "Hello World" },
  "content": { "rendered": "<p>Content...</p>" },
  "wpml_current_locale": "en",
  "wpml_trid": 456,
  "wpml_translations": {
    "es": {
      "id": 124,
      "locale": "es",
      "title": "Hola Mundo",
      "permalink": "https://example.com/es/hola-mundo",
      "status": "publish"
    },
    "fr": {
      "id": 125,
      "locale": "fr",
      "title": "Bonjour le monde",
      "permalink": "https://example.com/fr/bonjour-le-monde",
      "status": "publish"
    }
  }
}
```

### Auto-Translate Post

```bash
POST /wp-json/wpml/v1/auto-translate/{id}
Content-Type: application/json

{
  "target_languages": ["es", "fr", "de"],
  "translation_service": "openai"
}
```

**Response:**
```json
{
  "original_id": 123,
  "translations": {
    "es": {
      "id": 124,
      "status": "success",
      "permalink": "https://example.com/es/post-slug"
    },
    "fr": {
      "id": 125,
      "status": "success",
      "permalink": "https://example.com/fr/post-slug"
    }
  }
}
```

### Get String Translations

```bash
GET /wp-json/wpml/v1/strings?context=theme&language=en
```

### Batch Translate

```bash
POST /wp-json/wpml/v1/batch-translate
Content-Type: application/json

{
  "post_ids": [123, 456, 789],
  "target_languages": ["es", "fr"],
  "use_ai": true
}
```

---

## Usage Examples

### Example 1: Translate a Post via cURL

```bash
curl -X POST https://yoursite.com/wp-json/wpml/v1/auto-translate/123 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "target_languages": ["es", "fr"],
    "translation_service": "openai"
  }'
```

### Example 2: Get All Translations of a Post

```bash
curl https://yoursite.com/wp-json/wp/v2/posts/123 | jq '.wpml_translations'
```

### Example 3: JavaScript Client

```javascript
async function translatePost(postId, targetLanguages) {
  const response = await fetch(`/wp-json/wpml/v1/auto-translate/${postId}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-WP-Nonce': wpApiSettings.nonce
    },
    body: JSON.stringify({
      target_languages: targetLanguages,
      translation_service: 'openai'
    })
  });
  
  return await response.json();
}

// Usage
translatePost(123, ['es', 'fr', 'de'])
  .then(result => console.log('Translations created:', result))
  .catch(error => console.error('Translation failed:', error));
```

### Example 4: Python Script for Batch Translation

```python
import requests
import json

def batch_translate_posts(post_ids, target_languages, site_url, token):
    endpoint = f"{site_url}/wp-json/wpml/v1/batch-translate"
    
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {token}'
    }
    
    data = {
        'post_ids': post_ids,
        'target_languages': target_languages,
        'use_ai': True
    }
    
    response = requests.post(endpoint, headers=headers, json=data)
    return response.json()

# Usage
result = batch_translate_posts(
    post_ids=[123, 456, 789],
    target_languages=['es', 'fr', 'de'],
    site_url='https://yoursite.com',
    token='YOUR_JWT_TOKEN'
)

print(json.dumps(result, indent=2))
```

---

## Advanced Patterns

### Pattern 1: Custom Field Translation

```php
<?php
/**
 * Translate ACF fields automatically
 */

add_filter('rest_prepare_post', 'add_acf_translations', 10, 3);

function add_acf_translations($response, $post, $request) {
    if (!function_exists('get_field_objects')) {
        return $response;
    }
    
    $trid = apply_filters('wpml_element_trid', null, $post->ID, 'post_' . $post->post_type);
    
    if (!$trid) {
        return $response;
    }
    
    $translations = apply_filters('wpml_get_element_translations', null, $trid, 'post_' . $post->post_type);
    
    $acf_translations = [];
    
    foreach ($translations as $lang_code => $translation) {
        $fields = get_field_objects($translation->element_id);
        $acf_translations[$lang_code] = $fields;
    }
    
    $response->data['acf_translations'] = $acf_translations;
    
    return $response;
}
```

### Pattern 2: Translation Status Tracking

```php
<?php
/**
 * Track translation progress and quality
 */

register_rest_field('post', 'translation_status', [
    'get_callback' => function($object) {
        $trid = apply_filters('wpml_element_trid', null, $object['id'], 'post_' . $object['type']);
        
        if (!$trid) {
            return null;
        }
        
        $translations = apply_filters('wpml_get_element_translations', null, $trid, 'post_' . $object['type']);
        
        $status = [
            'total_languages' => count($translations),
            'translated' => 0,
            'pending' => 0,
            'needs_review' => 0
        ];
        
        foreach ($translations as $translation) {
            $post_status = get_post_status($translation->element_id);
            
            if ($post_status === 'publish') {
                $status['translated']++;
            } elseif ($post_status === 'draft') {
                $status['pending']++;
            } elseif (get_post_meta($translation->element_id, '_needs_review', true)) {
                $status['needs_review']++;
            }
        }
        
        return $status;
    }
]);
```

### Pattern 3: Webhook Integration

```php
<?php
/**
 * Trigger webhooks on translation events
 */

add_action('wpml_after_save_post', 'trigger_translation_webhook', 10, 3);

function trigger_translation_webhook($post_id, $trid, $language) {
    $webhook_url = get_option('wpml_translation_webhook_url');
    
    if (!$webhook_url) {
        return;
    }
    
    $post = get_post($post_id);
    $translations = apply_filters('wpml_get_element_translations', null, $trid, 'post_' . $post->post_type);
    
    $payload = [
        'event' => 'translation_saved',
        'post_id' => $post_id,
        'trid' => $trid,
        'language' => $language,
        'post_type' => $post->post_type,
        'translations' => array_map(function($t) {
            return [
                'id' => $t->element_id,
                'language' => $t->language_code,
                'status' => get_post_status($t->element_id)
            ];
        }, $translations)
    ];
    
    wp_remote_post($webhook_url, [
        'headers' => ['Content-Type' => 'application/json'],
        'body' => json_encode($payload)
    ]);
}
```

---

## Security Considerations

### 1. Authentication & Authorization

```php
// Require authentication for translation endpoints
register_rest_route('wpml/v1', '/auto-translate/(?P<id>\d+)', [
    'permission_callback' => function($request) {
        // Check user capabilities
        if (!current_user_can('edit_posts')) {
            return new WP_Error(
                'rest_forbidden',
                'You do not have permission to translate posts',
                ['status' => 403]
            );
        }
        
        // Check if user can edit the specific post
        $post_id = $request->get_param('id');
        if (!current_user_can('edit_post', $post_id)) {
            return new WP_Error(
                'rest_forbidden',
                'You do not have permission to translate this post',
                ['status' => 403]
            );
        }
        
        return true;
    }
]);
```

### 2. Rate Limiting

```php
function wpml_check_rate_limit($user_id) {
    $transient_key = 'wpml_rate_limit_' . $user_id;
    $requests = get_transient($transient_key) ?: 0;
    
    if ($requests > 100) { // 100 requests per hour
        return new WP_Error(
            'rate_limit_exceeded',
            'Translation rate limit exceeded. Please try again later.',
            ['status' => 429]
        );
    }
    
    set_transient($transient_key, $requests + 1, HOUR_IN_SECONDS);
    return true;
}
```

### 3. Input Validation

```php
function wpml_validate_language_code($lang_code) {
    $active_languages = apply_filters('wpml_active_languages', null);
    
    if (!isset($active_languages[$lang_code])) {
        return new WP_Error(
            'invalid_language',
            'Invalid language code provided',
            ['status' => 400]
        );
    }
    
    return true;
}
```

### 4. Content Sanitization

```php
function wpml_sanitize_translation($translated_content) {
    // Remove potentially harmful content
    $translated_content = wp_kses_post($translated_content);
    
    // Additional sanitization for specific cases
    $translated_content = sanitize_text_field($translated_content);
    
    return $translated_content;
}
```

---

## Conclusion

This document provides a theoretical framework for reverse engineering WPML's REST API and creating alternative auto-translation workflows. The key insights are:

1. **REST API Extension**: Expose WPML data through custom REST fields
2. **Translation Groups**: Use `trid` to link translations together
3. **Programmatic Access**: Enable REST API and MCP tools to manage translations
4. **Auto-Translation**: Integrate external services (Google Translate, DeepL, OpenAI)
5. **Batch Processing**: Handle multiple posts efficiently
6. **Security**: Implement proper authentication, rate limiting, and validation

### Benefits

- ✅ Programmatic access to translation data
- ✅ Integration with external translation services
- ✅ Batch translation capabilities
- ✅ Support for complex content structures (ACF, custom fields)
- ✅ REST API and MCP tool compatibility
- ✅ Flexible auto-translation workflows

### Use Cases

- Managing translations for multiple countries with different content
- Automating translation workflows with AI/ML services
- Building custom translation management interfaces
- Integrating with external content management systems
- Supporting complex repeater fields and custom content

---

## Resources

- [WPML Documentation](https://wpml.org/documentation/)
- [WordPress REST API Handbook](https://developer.wordpress.org/rest-api/)
- [WPML Filters Reference](https://wpml.org/wpml-hook/)
- [Translation API Integrations](https://cloud.google.com/translate)

## License

This is a theoretical guide for educational purposes. Implementation should follow WPML's terms of service and WordPress coding standards.

## Contributing

This document is open to contributions and improvements. Feel free to suggest enhancements or additional patterns.

---

**Note**: This is a theoretical reverse engineering guide. Always ensure compliance with WPML's license and terms of service when implementing these solutions in production environments.

