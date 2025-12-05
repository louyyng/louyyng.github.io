---
title: "Example Blog Post"
date: 2025-12-05
categories:
  - blog
tags:
  - example
author_profile: true
read_time: true
comments: false
share: true
related: true
header:
  teaser: /images/profile.png
---

This is an example blog post to help you get started with writing content for your Jekyll site. This post demonstrates various markdown features and Jekyll-specific functionality.

## Headings

You can use different heading levels to structure your content:

### Level 3 Heading

#### Level 4 Heading

##### Level 5 Heading

###### Level 6 Heading

## Text Formatting

You can use **bold text**, *italic text*, and ***bold italic text***. You can also use ~~strikethrough~~ text.

## Lists

### Unordered Lists

- First item
- Second item
  - Nested item
  - Another nested item
- Third item

### Ordered Lists

1. First numbered item
2. Second numbered item
   1. Nested numbered item
   2. Another nested item
3. Third numbered item

## Links and Images

Here's a [link to GitHub](https://github.com) and an example image:

![Example Image]({{ site.baseurl }}/images/profile.png)

## Code Blocks

You can include code blocks with syntax highlighting:

```python
def hello_world():
    print("Hello, World!")
    return True
```

```javascript
function greet(name) {
    console.log(`Hello, ${name}!`);
    return `Greetings, ${name}`;
}
```

You can also use inline code like `console.log()` or `print()`.

## Blockquotes

> This is a blockquote. You can use it to highlight important information or quotes from other sources.
>
> You can have multiple paragraphs in a blockquote.

## Tables

| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Row 1    | Data     | More data|
| Row 2    | Info     | More info|
| Row 3    | Content  | More content|

## Horizontal Rules

You can use horizontal rules to separate sections:

---

## Jekyll Features

### Liquid Tags

You can use Jekyll's Liquid templating:

- Site title: {{ site.title }}
- Site description: {{ site.description }}
- Current date: {{ "now" | date: "%Y-%m-%d" }}

### MathJax (if enabled)

You can include mathematical expressions:

Inline math: $E = mc^2$

Display math:

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

## Notices

You can add notices or callouts:

**Note:** This is a note notice.
{: .notice}

**Warning:** This is a warning notice.
{: .notice--warning}

**Info:** This is an info notice.
{: .notice--info}

## Conclusion

This example post demonstrates many of the markdown features available in Jekyll. Feel free to use this as a template for your own blog posts!

---

**Tags:** {% for tag in page.tags %}{{ tag }}{% unless forloop.last %}, {% endunless %}{% endfor %}

