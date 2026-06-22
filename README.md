# WordPress Block Editor কিভাবে কাজ করে?

WordPress Classic theme এর মত WordPress Block Theme ও পোস্টের কনটেন্ট মূলত WordPress database-এই save হয়। Block editor যখন আপনি code editor এ ওপেন করবেন, তখন HTML comment এর মত কিছু কমেন্ট দেখতে পাবেন। এগুলোকে block markup বলা হয়। 

---

## How Gutenberg Stores Content

আমরা ব্লক এডিটরে যে কন্টেন্ট এড করি, এগুলো ব্লক মার্কআপ হিসাবে থাকে। এই কন্টেন্টগুলো ডাটাবেজের `wp_posts.post_content` field as serialized block markup হিসাবে সেভ হয়। 

### Example

```html
<!-- wp:paragraph -->
<p>Hello World</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":31,"sizeSlug":"large","linkDestination":"none"} -->
<figure class="wp-block-image size-large">
    <img src="uploads/image.jpg" alt="" class="wp-image-31" />
</figure>
<!-- /wp:image -->
```

---

## Database Structure

### Post Content

| Field          | Description                       |
| -------------- | --------------------------------- |
| `ID`           | Unique post ID                    |
| `post_type`    | post, page, attachment, etc.      |
| `post_status`  | publish, draft, inherit           |
| `post_content` | Serialized Gutenberg block markup |

### Example

| ID | post_type  | post_status |
| -- | ---------- | ----------- |
| 41 | post       | publish     |
| 31 | attachment | inherit     |

---

### ডাটাবেজ থেকে কন্টেন্ট কিভাবে প্রদর্শিত হয়?

### Paragraph Block

```html
<!-- wp:paragraph -->
<p>Hello World</p>
<!-- /wp:paragraph -->
```

উপরের ব্লক মার্কআপটি ডাটাবেজে plain text হিসাবে ডাটাবেজে সংরক্ষিত হয়। 

কিন্তু WordPress internally প্রায় এরকম structure তৈরি করে:

```html
{
  "blockName": "core/paragraph",
  "attrs": {},
  "content": "<p>Hello World</p>"
}
```

```html
Array(
    [0] => Array(
        [blockName] => "core/paragraph",
        [attrs] => Array(),
        [innerBlocks] => Array(),
        [innerHTML] => "<p>Hello World</p>",
        [innerContent] => Array(
            [0] => "<p>Hello World</p>"
        )
    )
)
```

### Image Block

```html
<!-- wp:image {"id":31,"sizeSlug":"large","linkDestination":"none"} -->
<figure class="wp-block-image size-large">
    <img src="uploads/image.jpg" alt="" class="wp-image-31" />
</figure>
<!-- /wp:image -->
```

```html
{
  "id": 31,
  "sizeSlug": "large",
  "linkDestination": "none"
}
```

```html
[
    [
        'blockName'    => 'core/image',
        'attrs'        => [
            'id' => 31,
            'sizeSlug' => 'large',
            'linkDestination' => 'none',
        ],
        'innerBlocks'  => [],
        'innerHTML'    => '<figure class="wp-block-image size-large">
            <img src="uploads/image.jpg" alt="" class="wp-image-31" />
        </figure>',
        'innerContent' => [
            '<figure class="wp-block-image size-large">
                <img src="uploads/image.jpg" alt="" class="wp-image-31" />
            </figure>'
        ],
    ]
]
```

### Paragraph with Attributes
```html
<!-- wp:paragraph {"className":"simple-text","style":{"css":"color:red;"},"anchor":"Hello-World"} -->
<p class="simple-text has-custom-css" id="Hello-World">Hello World!</p>
<!-- /wp:paragraph -->
```

```html
{
  "blockName": "core/paragraph",
  "attrs": {
    "className": "simple-text",
    "style": {
      "css": "color:red;"
    },
    "anchor": "Hello-World"
  },
  "innerBlocks": [],
  "innerHTML": "<p class=\"simple-text has-custom-css\" id=\"Hello-World\">Hello World!</p>",
  "innerContent": [
    "<p class=\"simple-text has-custom-css\" id=\"Hello-World\">Hello World!</p>"
  ]
}
```

```html
[
    [
        'blockName' => 'core/paragraph',

        'attrs' => [
            'className' => 'simple-text',

            'style' => [
                'css' => 'color:red;'
            ],

            'anchor' => 'Hello-World',
        ],

        'innerBlocks' => [],

        'innerHTML' =>
            '<p class="simple-text has-custom-css" id="Hello-World">Hello World!</p>',

        'innerContent' => [
            '<p class="simple-text has-custom-css" id="Hello-World">Hello World!</p>'
        ],
    ]
]
```

## Request Lifecycle

```mermaid
flowchart LR
    editor["Block Editor"]
    database["wp_posts.post_content"]
    parser["parse_blocks"]
    renderer["render_block"]
    frontend["Frontend HTML"]

    editor --> database
    database --> parser
    parser --> renderer
    renderer --> frontend
```

---

## Block Theme-এর Template এবং Template Part কোথায় save হয়?

যদি Site Editor থেকে আপনি template edit করেন:

Template changes → wp_template
Template Part changes → wp_template_part

এগুলো database-এ custom post type হিসেবে save হয়, এবং wp_posts টেবিলেই থাকে।

## Block Parsing

WordPress converts block markup into structured block objects.

```php
$blocks = parse_blocks( $post->post_content );
```

### Example Output

```php
Array(
    [0] => Array(
        [blockName] => "core/paragraph"
        [attrs] => Array()
    )
)
```

---

## Rendering Blocks

Each block is rendered individually.

```php
echo render_block( $block );
```

### Output

```html
<p>Hello World</p>
```

---

## Image Block Relationship

```mermaid
flowchart TD
    A[Image Block]
    A --> B[Attachment ID 31]
    B --> C[wp_posts]
    B --> D[wp_postmeta]
    D --> E[_wp_attachment_metadata]
    E --> F[Generated Image Sizes]
```

---

## Block Theme Rendering Flow

```mermaid
flowchart LR
    A[Block Editor]
    B[wp_posts.post_content]
    C[parse_blocks()]
    D[render_block()]
    E[Theme Template]
    F[Frontend HTML]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
```

---

## Template Resolution

When a visitor opens a post, WordPress selects the appropriate Block Theme template.

Typical hierarchy:

```text
single-post.html
single.html
page.html
index.html
```

---

## Site Editor Templates

Templates modified through the Site Editor are stored in the database as custom post types.

```text
wp_template
wp_template_part
```

These records are also stored inside the `wp_posts` table.

---

## Dynamic vs Static Blocks

### Static Blocks

Rendered directly from saved content.

Examples:

* Paragraph
* Heading
* Image
* List

### Dynamic Blocks

Rendered at runtime.

Examples:

* Latest Posts
* Query Loop
* Site Title
* Navigation

---

## Complete Architecture

```mermaid
flowchart TD

    A[User Creates Content]
    --> B[Block Editor]

    B --> C[wp_posts.post_content]

    C --> D[Visitor Requests URL]

    D --> E[WP_Query]

    E --> F[Load Post Data]

    F --> G[parse_blocks()]

    G --> H[render_block()]

    H --> I[Block Theme Templates]

    I --> J[Generate HTML]

    J --> K[Browser]
```

---

## Key Takeaways

* Gutenberg stores content inside `wp_posts.post_content`
* Blocks are represented using HTML comment markers
* `parse_blocks()` converts serialized markup into block objects
* `render_block()` generates frontend HTML
* Block Themes control presentation, not content storage
* Templates edited through the Site Editor are stored as `wp_template` and `wp_template_part`
