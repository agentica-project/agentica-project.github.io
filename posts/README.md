# Blog Posts

This directory contains markdown files for blog posts that are rendered using the `post.html` template.

## How to Create a New Blog Post

1. **Create a markdown file** in this `posts/` directory (e.g., `my-new-post.md`)

2. **Add frontmatter** at the top of your markdown file:
   ```markdown
   ---
   title: "Your Blog Post Title"
   author: "Author Name"
   author_line: "Author Name in collaboration with others at Organization"
   date: "2025-01-27"
   ---
   ```

3. **Write your content** using standard markdown syntax. The system supports:
   - Headers (automatically added to table of contents)
   - Code blocks with syntax highlighting
   - Math expressions using LaTeX syntax: `\(inline\)` and `\[display\]`
   - Tables, lists, blockquotes, images
   - Links and emphasis

4. **Add the post to blog.html** by adding a new article entry:
   ```html
   <article class="blog-post">
     <div class="date">
       <h2>Month<sup>day</sup></h2>
     </div>
     <div class="post-content">
       <h3>
         <a href="post.html?post=my-new-post.md">
           Your Blog Post Title
         </a>
       </h3>
       <p class="post-meta">
         Jan 27, 2025 | X min read | Author Name
       </p>
     </div>
   </article>
   ```

## Features

The blog post system includes:

- **Automatic table of contents** generation from headers
- **Syntax highlighting** for code blocks
- **Math rendering** using MathJax
- **Reading time estimation**
- **Automatic citation generation**
- **Dark/light theme support**
- **Responsive design**
- **Smooth scrolling** to sections

## Accessing Posts

Posts are accessed via URL parameters:
- `post.html?post=sample-post.md` - loads the sample post
- `post.html?post=your-filename.md` - loads your post

## Math Support

Use LaTeX syntax for mathematical expressions:
- Inline: `\(x = y + z\)`
- Display: `\[E = mc^2\]`

## Code Highlighting

Specify the language for syntax highlighting:
```python
def hello_world():
    print("Hello, world!")
```

## Example

See `sample-post.md` for a complete example that demonstrates all features. 