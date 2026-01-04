# How to Add Images to Your Hugo Blog Posts

## Method 1: Using the `figure` Shortcode (Recommended)

The `figure` shortcode is the best option for images with captions. It provides responsive images and proper styling.

### Basic Usage:
```markdown
{{< figure src="/images/your-image.png" alt="Description" caption="Your caption text" >}}
```

### With Alignment:
```markdown
{{< figure src="/images/your-image.png" alt="Description" caption="Centered image" align="center" >}}
```

### With Link:
```markdown
{{< figure src="/images/your-image.png" alt="Description" caption="Clickable image" link="https://example.com" >}}
```

### With Custom Width/Height:
```markdown
{{< figure src="/images/your-image.png" alt="Description" width="800" height="600" caption="Custom sized image" >}}
```

## Method 2: Using Standard Markdown Syntax

Simple inline images using standard Markdown:

```markdown
![Alt text](/images/your-image.png)
```

With a link:
```markdown
[![Alt text](/images/your-image.png)](https://example.com)
```

## Method 3: Using `inTextImg` Shortcode (For Small Inline Images)

For small inline images within text:

```markdown
{{< inTextImg url="/images/icon.png" alt="Icon" height="20" >}}
```

## Method 4: Cover Image (Post Header Image)

Add a cover image in the frontmatter:

```yaml
+++
title = 'Your Post Title'
cover.image = '/images/cover-image.jpg'
cover.alt = 'Cover image description'
cover.caption = 'Cover image caption'
+++
```

## Where to Place Images

1. **Static Directory**: Place images in `/static/images/` folder
   - Access them as `/images/filename.png` in your markdown

2. **Post-Specific Images**: You can also organize by post:
   - `/static/images/posts/post-name/image.png`
   - Access as `/images/posts/post-name/image.png`

## Image Best Practices

- **Format**: Use WebP, PNG, or JPG
- **Size**: Optimize images before uploading (use tools like TinyPNG)
- **Naming**: Use descriptive, lowercase filenames with hyphens (e.g., `postgres-index-comparison.png`)
- **Alt Text**: Always include alt text for accessibility
- **Responsive**: The `figure` shortcode automatically makes images responsive

## Example in Your Post

Here's how you might add a benchmark results image:

```markdown
{{< figure src="/images/benchmark-results.png" alt="Benchmark comparison between 2 indexes vs 10 indexes" caption="Performance comparison showing latency differences" align="center" >}}
```

