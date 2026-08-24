# NexaMind Labs — Content Creation Bible

> **Living document.** Every time a new content feature is added to the website,
> this guide is updated to reflect it. If something isn't in here, it isn't
> officially supported yet.

**Last updated:** 2026-08-24  
**Site:** https://nexamindlabs.com  
**Repo:** https://github.com/jyothishnarayan/nexamindlabs-website  
**Contact:** info@nexamindlabs.com

---

## Table of contents

1. [Site structure](#1-site-structure)
2. [Blog posts](#2-blog-posts)
   - [File naming](#21-file-naming)
   - [Frontmatter reference](#22-frontmatter-reference)
   - [Markdown formatting guide](#23-markdown-formatting-guide)
   - [Categories](#24-categories)
   - [Publishing workflow — GitHub (primary)](#25-publishing-workflow--github-primary)
   - [Publishing workflow — CMS Admin (alternative)](#26-publishing-workflow--cms-admin-alternative)
   - [Editing or deleting a post](#27-editing-or-deleting-a-post)
3. [Homepage content](#3-homepage-content)
   - [Updating services](#31-updating-services)
   - [Updating testimonials](#32-updating-testimonials)
   - [Updating stats](#33-updating-stats)
4. [Images and media](#4-images-and-media)
5. [Adding new pages](#5-adding-new-pages)
6. [SEO checklist](#6-seo-checklist)
7. [Sitemap](#7-sitemap)
8. [Planned features](#8-planned-features)
9. [Changelog](#9-changelog)

---

## 1. Site structure

```
nexamindlabs.com/
├── /                    → Homepage (index.html)
├── /blog.html           → Blog listing page (auto-populates from posts/)
├── /post.html           → Individual post reader (?slug=filename)
├── /admin/              → CMS editor (Decap CMS — GitHub login)
└── /sitemap.xml         → Auto-indexed by Google
```

**How content flows:**

```
You write a .md file → push to GitHub posts/ folder
         ↓
Cloudflare Pages deploys (~30 seconds)
         ↓
blog.html fetches the file list from GitHub API
         ↓
Post appears on /blog.html automatically
         ↓
Clicking the post loads /post.html?slug=filename
         ↓
post.html fetches and renders the Markdown live
```

No build step. No CMS publish button (for the GitHub workflow). Commit = published.

---

## 2. Blog posts

### 2.1 File naming

All posts live in the `/posts/` folder of the GitHub repository.

**Format:**
```
YYYY-MM-DD-url-friendly-title.md
```

**Rules:**
- Date must match the `date:` field in frontmatter
- Use hyphens, never spaces or underscores
- Lowercase only
- Keep it short — this becomes the URL slug

**Good examples:**
```
2026-08-24-ai-powered-pm-tool.md
2026-09-01-cbse-maths-study-guide.md
2026-09-15-building-rag-pipeline-supabase.md
```

**Bad examples:**
```
AI PM Tool.md                  ← spaces, no date
ai_pm_tool.md                  ← underscores, no date
2026-08-24-This Is My Post.md  ← spaces, mixed case
```

---

### 2.2 Frontmatter reference

Every post **must** start with a frontmatter block between triple dashes.
Missing or malformed frontmatter will cause the post to not render correctly.

```markdown
---
title: Your Full Post Title Here
date: 2026-08-24T09:00:00.000Z
author: Jyothish Narayan
category: AI
summary: One or two sentence summary shown on the blog listing card. Keep it under 160 characters for best display.
---

Post content starts here...
```

| Field | Required | Format | Notes |
|---|---|---|---|
| `title` | ✅ Yes | Plain text | Shown as H1 on post page and card title on listing |
| `date` | ✅ Yes | `YYYY-MM-DDTHH:MM:SS.000Z` | Used for sorting — newest first |
| `author` | ✅ Yes | Plain text | Shown on post card and post header |
| `category` | ✅ Yes | See [Categories](#24-categories) | Must be exact match |
| `summary` | ✅ Yes | Plain text, <160 chars | Shown on blog listing card below title |

**Full example with all fields:**
```markdown
---
title: How to Build a CBSE Study Plan Using AI
date: 2026-09-01T08:00:00.000Z
author: Jyothish Narayan
category: Education
summary: A step-by-step guide to building a personalised CBSE study schedule using AI tools — tested with real students from the Bright Future programme.
---
```

---

### 2.3 Markdown formatting guide

Everything below the closing `---` of the frontmatter is your post body.
The site renders standard Markdown. Here's the full reference:

#### Headings

```markdown
## Section heading       → Large section break (use for main sections)
### Subsection heading   → Smaller subsection
#### Minor heading       → Use sparingly
```

> ⚠️ Don't use `# H1` inside the post body — the post title is already an H1.
> Start from `##`.

#### Paragraphs and emphasis

```markdown
Normal paragraph text. Just write. Line breaks in the source 
don't create new paragraphs — you need a blank line between them.

This is a new paragraph.

**Bold text** for important terms.
*Italic text* for emphasis or titles.
~~Strikethrough~~ for deprecated info.
`inline code` for file names, commands, variable names.
```

#### Lists

```markdown
Unordered list:
- First item
- Second item
  - Nested item (indent with 2 spaces)

Ordered list:
1. Step one
2. Step two
3. Step three
```

#### Code blocks

Always specify the language for syntax highlighting:

````markdown
```python
def hello():
    print("Hello, NexaMind!")
```

```javascript
const greet = () => console.log("Hello!");
```

```sql
select * from tasks order by priority desc;
```

```bash
npm install && npm run build
```

```yaml
backend:
  name: github
  repo: jyothishnarayan/nexamindlabs-website
```
````

Supported languages: `python`, `javascript`, `typescript`, `jsx`, `tsx`,
`sql`, `bash`, `yaml`, `json`, `html`, `css`, `markdown`, `text`

#### Tables

```markdown
| Column 1 | Column 2 | Column 3 |
|---|---|---|
| Row 1    | Data     | More data |
| Row 2    | Data     | More data |
```

#### Blockquotes

```markdown
> This is a blockquote. Use for important callouts, 
> quotes from clients, or key takeaways.
```

#### Links

```markdown
[Link text](https://example.com)
[Email us](mailto:info@nexamindlabs.com)
[Internal blog link](/blog.html)
```

#### Images

> 📌 **Current limitation:** The blog does not yet have an image upload system.
> Use one of these approaches in the meantime:

**Option A — External URL (recommended for now):**
```markdown
![Alt text describing the image](https://your-image-host.com/image.png)
```

**Option B — Unsplash (free, no attribution required for editorial use):**
```markdown
![Architecture diagram](https://images.unsplash.com/photo-XXXXXXX?w=800)
```

**Option C — ASCII diagrams (best for technical content):**
````markdown
```
┌─────────────┐     ┌─────────────┐
│   Frontend  │────▶│   Backend   │
└─────────────┘     └─────────────┘
```
````

> 📅 A proper image upload system will be added in a future update.
> This guide will be updated when that feature is ready.

#### Horizontal dividers

```markdown
---
```

Use sparingly — between major sections of very long posts.

---

### 2.4 Categories

The category field controls the colour and icon of the post card on the blog listing page.
Use **exactly** one of these values (case-sensitive):

| Category | Icon | Card colour | Use for |
|---|---|---|---|
| `AI` | 🤖 | Indigo/purple | Machine learning, LLMs, AI tools, RAG, embeddings |
| `Dev` | 🚀 | Teal/green | Development tutorials, code, architecture, tooling |
| `Education` | 📚 | Yellow/amber | CBSE guides, study tips, Bright Future content |
| `SaaS` | ⚡ | Green | Product building, SaaS strategies, business |
| `General` | 💡 | Light grey | Company news, announcements, opinion pieces |

**To add a new category** — requires a code change in `blog.html` (the `THUMBS` object). Document the change in [Section 9 — Changelog](#9-changelog).

---

### 2.5 Publishing workflow — GitHub (primary)

This is the main workflow. No CMS, no login issues, works every time.

**Step 1: Go to the posts folder**

> https://github.com/jyothishnarayan/nexamindlabs-website/tree/main/posts

**Step 2: Create a new file**

Click **Add file → Create new file**

**Step 3: Name the file**

In the filename box, type the filename following [Section 2.1](#21-file-naming):
```
2026-09-01-your-post-title.md
```

**Step 4: Write the post**

Paste the frontmatter template first, then write your content below it:

```markdown
---
title: 
date: 2026-09-01T09:00:00.000Z
author: Jyothish Narayan
category: AI
summary: 
---

Write your post here...
```

**Step 5: Preview (optional but recommended)**

Click the **Preview** tab at the top of the editor to see how the Markdown renders.
Note: frontmatter will show as raw text in GitHub preview — that's normal.

**Step 6: Commit**

Scroll down to "Commit new file":
- Leave the commit message as-is, or write a short note
- Select **"Commit directly to the `main` branch"**
- Click **Commit new file**

**Step 7: Wait ~30 seconds**

Cloudflare Pages detects the push and redeploys automatically.
Your post appears at `nexamindlabs.com/blog.html` within 30 seconds.

**Step 8: Verify**

Open https://nexamindlabs.com/blog.html — your post should appear at the top.
Click it to confirm the full post renders correctly.

---

### 2.6 Publishing workflow — CMS Admin (alternative)

> ⚠️ **Current status:** The CMS admin login is partially working.
> The GitHub OAuth flow completes but the session token handoff is inconsistent.
> Use the GitHub workflow (Section 2.5) as the primary method until this is resolved.

When working:

1. Go to **https://nexamindlabs.com/admin/**
2. Click **Login with GitHub** → authorize in the popup
3. Click **Blog Posts → New Blog Post**
4. Fill in the fields:
   - Title, Date, Author, Category, Summary — required fields
   - Body — the rich Markdown editor
5. Click **Publish** → this commits the `.md` file to the `posts/` folder automatically
6. Cloudflare deploys within 30 seconds

> 📅 CMS login stability fix is in the roadmap for the dynamic (Next.js) migration phase.

---

### 2.7 Editing or deleting a post

**To edit:**
1. Go to the file in GitHub → `posts/your-post-filename.md`
2. Click the **pencil (edit) icon**
3. Make your changes
4. Commit directly to main

Changes deploy in ~30 seconds.

**To delete:**
1. Go to the file in GitHub → `posts/your-post-filename.md`
2. Click the **three dots menu (···)** → **Delete file**
3. Commit the deletion

The post disappears from the blog listing within 30 seconds.

---

## 3. Homepage content

The homepage (`index.html`) is the one file that requires direct code editing.
Future updates will move these to a CMS-managed data source.

### 3.1 Updating services

In `index.html`, find the `<!-- SERVICES -->` section.
Each service card looks like:

```html
<div class="srv-card">
  <div class="srv-ico-wrap ico-i">
    <!-- SVG icon here -->
  </div>
  <h3>Service Title</h3>
  <p>Service description — 1-2 sentences.</p>
  <a href="#contact" class="srv-link">Let's talk →</a>
</div>
```

**Icon wrapper classes** (controls the background colour):
- `ico-i` — indigo (AI services)
- `ico-t` — teal (development)
- `ico-a` — amber (SaaS/products)
- `ico-n` — green (collaboration)

### 3.2 Updating testimonials

Find the `<!-- TESTIMONIALS -->` section in `index.html`.
Each testimonial card:

```html
<div class="testi-card">
  <div class="testi-stars">★★★★★</div>
  <p class="testi-text">"Testimonial text in quotes."</p>
  <div class="testi-author">
    <div class="testi-avatar" style="background:linear-gradient(...)">AB</div>
    <div>
      <div class="testi-name">Full Name</div>
      <div class="testi-role">Role, Company</div>
    </div>
  </div>
</div>
```

Replace the placeholder testimonials with real ones as they come in.

### 3.3 Updating stats

In the hero section, find:

```html
<div><div class="hs-num">50<em>+</em></div><div class="hs-lbl">Projects delivered</div></div>
<div><div class="hs-num">2<em> tracks</em></div><div class="hs-lbl">Pro & Bright Future</div></div>
<div><div class="hs-num">AI<em>-first</em></div><div class="hs-lbl">Every solution</div></div>
```

Update the numbers as the business grows.

---

## 4. Images and media

### Current state

No dedicated image upload system exists yet. Options:

| Method | How | Best for |
|---|---|---|
| External URL | Paste image URL in Markdown | Screenshots, diagrams |
| GitHub upload | Upload to `images/` folder in repo, reference as `/images/filename.png` | Logos, brand assets |
| Unsplash | Free editorial images, paste URL | Cover images, illustrations |

### Uploading to GitHub

1. Go to the repo root → **Add file → Upload files**
2. Upload your image to an `images/` folder (create it by typing `images/` before the filename in the Create new file field)
3. Reference it in your post:
   ```markdown
   ![Description](/images/your-image.png)
   ```

### Planned: CMS image uploads

When the CMS login is stable, Decap CMS is configured to store uploaded images in `/images/uploads/`. This guide will be updated.

---

## 5. Adding new pages

To add a new standalone page (e.g., `/training.html`, `/about.html`):

1. Create the HTML file in the repo root
2. Use the same nav and footer structure as `blog.html` or `post.html` for consistency
3. Add it to the sitemap (`sitemap.xml`)
4. Add a link in the footer nav in `index.html`
5. Update this guide's [Site structure](#1-site-structure) section

### Page template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Page Title — NexaMind Labs</title>
<!-- Copy the <link> and <meta> tags from index.html head -->
</head>
<body>
<!-- Copy nav from blog.html -->
<nav>...</nav>

<!-- Your page content here -->
<main>
  ...
</main>

<!-- Copy footer from blog.html -->
<footer>...</footer>
</body>
</html>
```

---

## 6. SEO checklist

Run through this before publishing any post or page:

- [ ] `title` frontmatter is descriptive and under 60 characters
- [ ] `summary` frontmatter is under 160 characters (used as meta description)
- [ ] First paragraph of the post clearly states what it's about
- [ ] All images have descriptive alt text
- [ ] Internal links use relative paths (`/blog.html`, not full URL)
- [ ] Code examples are accurate and tested
- [ ] No placeholder text left in the post
- [ ] File is committed to `main` branch (not a draft branch)
- [ ] Post appears on `nexamindlabs.com/blog.html` after deploy
- [ ] Post renders correctly on mobile (check on your phone)

---

## 7. Sitemap

The sitemap lives at `nexamindlabs.com/sitemap.xml`.

**Update the sitemap when:**
- Adding a new page (add a new `<url>` entry)
- Blog posts don't need to be added manually — they're crawled via the blog listing page

**Current sitemap** (`sitemap.xml` in repo root):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://nexamindlabs.com/</loc>
    <lastmod>2026-08-09</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <!-- Add new pages here -->
</urlset>
```

**Adding a new page to the sitemap:**
```xml
<url>
  <loc>https://nexamindlabs.com/training.html</loc>
  <lastmod>2026-09-01</lastmod>
  <changefreq>monthly</changefreq>
  <priority>0.8</priority>
</url>
```

After updating, resubmit the sitemap in Google Search Console → Sitemaps.

---

## 8. Planned features

These are coming. When implemented, a new section is added to this guide.

| Feature | Status | Section when ready |
|---|---|---|
| CMS admin login (stable) | 🔄 In progress | Updates Section 2.6 |
| Image upload via CMS | ⏳ Planned | New Section 4.1 |
| Training registration page | ⏳ Planned | New Section: Training Content |
| Course material publishing | ⏳ Planned | New Section: Courses |
| Blog comments | ⏳ Planned | New Section: Community |
| Newsletter signup | ⏳ Planned | New Section: Newsletter |
| Blog post scheduling | ⏳ Planned | Updates Section 2.5 |
| Dynamic site (Next.js migration) | ⏳ Q4 2026 | Major guide update |

---

## 9. Changelog

Every time the content system is updated, log it here.

| Date | Change | Affects |
|---|---|---|
| 2026-08-24 | Initial blog system live — `blog.html`, `post.html`, `posts/` folder | Section 2 |
| 2026-08-24 | Decap CMS admin panel deployed at `/admin/` — login partially working | Section 2.6 |
| 2026-08-24 | Auth worker deployed at `cms-auth.getjyothishn.workers.dev` | Section 2.6 |
| 2026-08-24 | First real blog post published: AI PM Tool guide | Section 2.5 |
| 2026-08-24 | This content bible created at `docs/CONTENT_BIBLE.md` | All |

---

*This document is maintained by the NexaMind Labs team.  
To suggest changes or report inaccuracies: [info@nexamindlabs.com](mailto:info@nexamindlabs.com)*
