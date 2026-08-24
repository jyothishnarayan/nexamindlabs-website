# NexaMind Labs — Website Runbook

> Complete technical reference for the NexaMind Labs website.
> Keep this document updated whenever a service, credential, or process changes.

**Last updated:** 2026-08-24  
**Maintained by:** Jyothish Narayan  
**Contact:** info@nexamindlabs.com

---

## Table of Contents

1. [Domain](#1-domain)
2. [DNS & Hosting](#2-dns--hosting)
3. [Codebase & GitHub](#3-codebase--github)
4. [Database — Supabase](#4-database--supabase)
5. [Email](#5-email)
6. [Contact Form](#6-contact-form)
7. [Analytics](#7-analytics)
8. [Services & Credentials Summary](#8-services--credentials-summary)
9. [How to update the website](#9-how-to-update-the-website)
10. [How to add a new blog post](#10-how-to-add-a-new-blog-post)
11. [How to add a new page or section](#11-how-to-add-a-new-page-or-section)
12. [How to add a new email address](#12-how-to-add-a-new-email-address)
13. [Where enquiries go](#13-where-enquiries-go)
14. [Pending items](#14-pending-items)

---

## 1. Domain

| Property | Value |
|---|---|
| Domain name | `nexamindlabs.com` |
| Current registrar | Hostinger |
| Hostinger login | https://hpanel.hostinger.com |
| Hostinger account email | `getjyothishn@gmail.com` |
| Domain expiry | 31 December 2026 |
| Renewal price at Hostinger | ₹3,468/year |
| Planned transfer to | Cloudflare Registrar |
| Cloudflare renewal price | ~$10.46/year (~₹870) |
| Transfer status | ⏸ Pending — domain is Ready for Transfer in Cloudflare |

### Domain transfer — when ready
1. Go to **https://dash.cloudflare.com → Domain Registration → Transfer Domains**
2. Check the box next to `nexamindlabs.com` → Continue
3. Get EPP/Auth code from Hostinger → **https://hpanel.hostinger.com/domain/nexamindlabs.com/settings**
4. Paste the EPP code in Cloudflare → pay $10.46
5. Approve the transfer confirmation email (speeds up from 7 days to 24–48 hours)
6. After transfer completes — cancel Hostinger subscription (NOT before)

> ⚠️ The website stays live throughout the transfer. Zero downtime.

---

## 2. DNS & Hosting

### DNS
- **Provider:** Cloudflare (free)
- **Dashboard:** https://dash.cloudflare.com
- **Cloudflare account:** `getjyothishn@gmail.com`
- **Zone:** `nexamindlabs.com` — Status: Active

Nameservers at Hostinger point to Cloudflare. All DNS records (A, CNAME, MX, TXT) are managed in Cloudflare DNS dashboard.

### Website Hosting
- **Provider:** Cloudflare Pages (free tier)
- **Project name:** `nexamindlabs-website`
- **Live URL:** https://nexamindlabs.com
- **Pages URL:** https://nexamindlabs-website.pages.dev
- **Deploy method:** Auto-deploy on every push to `main` branch in GitHub
- **Deploy time:** ~30 seconds after commit

### Auth Worker (CMS login)
- **Worker name:** `cms-auth`
- **Worker URL:** https://cms-auth.getjyothishn.workers.dev
- **Purpose:** GitHub OAuth for Decap CMS admin panel
- **Workers subdomain:** `getjyothishn.workers.dev`

---

## 3. Codebase & GitHub

| Property | Value |
|---|---|
| GitHub account | `jyothishnarayan` |
| Repository | `jyothishnarayan/nexamindlabs-website` |
| Repo URL | https://github.com/jyothishnarayan/nexamindlabs-website |
| Default branch | `main` |
| GitHub login | `getjyothishn@gmail.com` |

### File structure
```
nexamindlabs-website/
├── index.html              → Homepage
├── about.html              → About page
├── training.html           → Training page
├── blog.html               → Blog listing
├── post.html               → Individual post reader
├── sitemap.xml             → Google sitemap
├── manifest.json           → PWA manifest
├── favicon.svg             → SVG favicon
├── favicon-16.png          → Favicon 16×16
├── favicon-32.png          → Favicon 32×32
├── apple-touch-icon.png    → iOS home screen icon
├── icon-192.png            → Android icon
├── icon-512.png            → PWA splash icon
├── og-image.png            → Social share card (1200×630)
├── admin/
│   ├── index.html          → Decap CMS editor
│   └── config.yml          → CMS configuration
├── posts/
│   ├── 2026-08-09-building-rag-pipeline-pgvector-supabase.md
│   ├── 2026-08-24-ai-powered-pm-tool.md
│   └── 2026-08-24-ai-in-qa-testing-automation.md
├── images/
│   ├── thumb-ai-qa.png
│   ├── thumb-ai-pm-tool.png
│   └── thumb-rag-pipeline.png
└── docs/
    ├── CONTENT_BIBLE.md    → Blog & content creation guide
    └── WEBSITE_RUNBOOK.md  → This document
```

---

## 4. Database — Supabase

| Property | Value |
|---|---|
| Provider | Supabase (free tier) |
| Dashboard | https://supabase.com |
| Login | `getjyothishn@gmail.com` |
| Project name | `nexamindlabs` |
| Project URL | `https://mdpsdsprdlxbfhmnkxaa.supabase.co` |
| Region | Southeast Asia (Singapore) |
| Anon/Public key | `sb_publishable_-NcEyQkOLTkzBqlQ-3YaBQ_ueVFK7rA` |
| Service role key | In Supabase → Settings → API (never expose publicly) |

### Tables

| Table | Purpose | Who writes | Who reads |
|---|---|---|---|
| `contact_enquiries` | Website contact form submissions | Public (anon) | Authenticated only |
| `enrolments` | Training enrolment form submissions | Public (anon) | Authenticated only |

### Viewing records
Supabase → Table Editor → select table → view rows

### Free tier limits
- 500 MB database storage
- 1 GB file storage
- 2 free projects
- Projects pause after 7 days of inactivity (add a keep-alive ping when going live seriously)

---

## 5. Email

| Address | Purpose | Provider |
|---|---|---|
| `info@nexamindlabs.com` | Primary — website contact form, general enquiries | Zoho Mail |
| `jyothish@nexamindlabs.com` | Personal/admin inbox | Zoho Mail |

### Zoho Mail access
- **Inbox:** https://mail.zoho.in
- **Admin console:** https://mailadmin.zoho.com
- **Zoho account login:** `getjyothishn@gmail.com` (Google SSO)
- **Domain:** `nexamindlabs.com` — verified via MX records in Cloudflare DNS

### DNS records for email (already set in Cloudflare)
- **MX records** — routes incoming email to Zoho
- **SPF record** — authorises Zoho to send from the domain
- **DKIM record** — cryptographic signature to prevent spam flagging

---

## 6. Contact Form

The homepage contact form (`/#contact`) works in two layers:

### Layer 1 — Formspree (email notification)
- **Provider:** Formspree (free tier — 50 submissions/month)
- **Dashboard:** https://formspree.io → login with `getjyothishn@gmail.com`
- **Form ID:** `xwlelpka`
- **Endpoint:** `https://formspree.io/f/xwlelpka`
- **Sends email to:** `info@nexamindlabs.com`
- **Email subject:** Dynamic — `New enquiry: [Topic] — NexaMind Labs`

### Layer 2 — Supabase (permanent record)
- **Table:** `contact_enquiries`
- **Fields saved:** full_name, email, enquiry_topic, message, created_at
- **View at:** Supabase → Table Editor → contact_enquiries

### Training enrolment form
- **Page:** `/training.html`
- **Saves to:** Supabase → `enrolments` table
- **Fields:** name, email, phone, track (pro/bright-future), role, experience, preferred_mode, student_name, grade, message

---

## 7. Analytics

| Property | Value |
|---|---|
| Provider | Cloudflare Web Analytics (free) |
| Dashboard | Cloudflare → nexamindlabs.com → Analytics → Web Analytics |
| Data available | Visits, page views, countries, browsers, devices, load time |
| Core Web Vitals | LCP, INP, CLS — all green |
| Page load time | ~398ms |
| Cookie-free | Yes — no GDPR banner needed |

### Google Search Console
- **Dashboard:** https://search.google.com/search-console
- **Login:** `getjyothishn@gmail.com`
- **Property:** `nexamindlabs.com`
- **Sitemap submitted:** `https://nexamindlabs.com/sitemap.xml`
- **Verified:** Yes

---

## 8. Services & Credentials Summary

| Service | URL | Login | Notes |
|---|---|---|---|
| Hostinger (domain registrar) | hpanel.hostinger.com | getjyothishn@gmail.com | Domain only until transfer |
| Cloudflare (DNS, hosting, analytics) | dash.cloudflare.com | getjyothishn@gmail.com | Primary control panel |
| GitHub (codebase) | github.com/jyothishnarayan | getjyothishn@gmail.com | Push here to deploy |
| Supabase (database) | supabase.com | getjyothishn@gmail.com | nexamindlabs project |
| Zoho Mail (email) | mail.zoho.in | Google SSO | info@nexamindlabs.com |
| Formspree (contact form) | formspree.io | getjyothishn@gmail.com | Form ID: xwlelpka |
| Google Search Console | search.google.com/search-console | getjyothishn@gmail.com | Sitemap submitted |

> 🔐 All passwords and secret keys (Supabase service role key, GitHub tokens, Formspree API key) should be stored in a password manager. Never commit secrets to GitHub.

---

## 9. How to update the website

### Method A — GitHub web editor (simplest)
1. Go to **https://github.com/jyothishnarayan/nexamindlabs-website**
2. Click the file you want to edit
3. Click the **pencil icon** (Edit)
4. Make your changes
5. Scroll down → **Commit changes** → Commit directly to `main`
6. Cloudflare Pages auto-deploys in ~30 seconds

### Method B — Clone locally (for bigger changes)
```bash
# One-time setup
git clone https://github.com/jyothishnarayan/nexamindlabs-website.git
cd nexamindlabs-website

# Every time you make changes
git add .
git commit -m "Describe what you changed"
git push origin main
```

### Which files to edit for what
| What you want to change | File to edit |
|---|---|
| Homepage content, sections | `index.html` |
| About page | `about.html` |
| Training page | `training.html` |
| Blog listing page | `blog.html` |
| Individual post reader | `post.html` |
| Add/edit a blog post | `posts/YYYY-MM-DD-title.md` |
| Social share card | `og-image.png` (regenerate from `nexamind-og-image.html`) |
| Site favicon | `favicon.svg` |
| Google sitemap | `sitemap.xml` |
| CMS configuration | `admin/config.yml` |

---

## 10. How to add a new blog post

**Full guide:** see `docs/CONTENT_BIBLE.md`

### Quick steps
1. Go to **https://github.com/jyothishnarayan/nexamindlabs-website/tree/main/posts**
2. Click **Add file → Create new file**
3. Name the file: `YYYY-MM-DD-post-title.md`
4. Paste the frontmatter template:

```markdown
---
title: Your Post Title
date: 2026-09-01T09:00:00.000Z
author: Jyothish Narayan
category: AI
summary: One or two sentence summary shown on the blog listing card.
image: /images/thumb-your-image.png
---

Post content in Markdown...
```

5. Write the post → Commit
6. Post appears on `nexamindlabs.com/blog.html` within 30 seconds
7. Homepage blog section auto-updates to show latest 3 posts

### Supported categories
`AI` · `Dev` · `Education` · `SaaS` · `General`

### Adding a thumbnail
- Screenshot the relevant `thumb-*.html` file in Chrome (full size screenshot)
- Upload the PNG to `images/` folder in GitHub
- Add `image: /images/your-thumb.png` to the post frontmatter

---

## 11. How to add a new page or section

### Adding a new standalone page (e.g. `/services.html`)
1. Create a new HTML file in the repo root
2. Copy the nav and footer from `blog.html` for consistency
3. Add it to `sitemap.xml` (see Section 9 — sitemap)
4. Add a link in the nav and footer of all existing pages
5. Update this runbook

### Adding a new section to the homepage
1. Edit `index.html`
2. Find the section comment (e.g. `<!-- BLOG -->`) — add your new section after it
3. Follow the existing CSS patterns (`.sec`, `.container`, `.sec-tag`, `.sec-title`)
4. Commit → deploys automatically

### Adding a new category to the blog
1. Edit `blog.html` — find the `THUMBS` object in the JavaScript
2. Add your new category:
```javascript
const THUMBS = {
  ...existing...,
  YourCategory: { bg: 'linear-gradient(135deg,#color1,#color2)', icon: '🎯' }
};
```
3. Also update `admin/config.yml` — add to the category `options` list
4. Update `index.html` — same `THUMBS` object in the dynamic blog loader script

---

## 12. How to add a new email address

1. Go to **https://mailadmin.zoho.com** (login with Google — `getjyothishn@gmail.com`)
2. Left sidebar → **User Management** → **Add User**
3. Fill in the email address (e.g. `hello@nexamindlabs.com`), name, password
4. Click Save
5. New inbox is immediately active — access via https://mail.zoho.in

**Current inboxes:**
- `info@nexamindlabs.com` — primary, used for website form and general contact
- `jyothish@nexamindlabs.com` — personal/admin

**Planned:**
- `hello@nexamindlabs.com` — training and friendly enquiries (not yet created)

---

## 13. Where enquiries go

### Contact form (`nexamindlabs.com/#contact`)
- **Email:** Arrives at `info@nexamindlabs.com` via Formspree
- **Database:** Saved to Supabase → `contact_enquiries` table
- **View in Supabase:** Table Editor → contact_enquiries → sorted by created_at

### Training enrolment form (`nexamindlabs.com/training.html`)
- **Database only:** Saved to Supabase → `enrolments` table
- **View in Supabase:** Table Editor → enrolments → sorted by created_at
- **Fields include:** track (pro/bright-future), name, email, phone, role/grade, message

> 💡 Tip: Bookmark the Supabase table editor URLs for quick access to new submissions.

---

## 14. Pending items

| Item | Priority | Notes |
|---|---|---|
| Domain transfer to Cloudflare | 🔴 High | Complete before Dec 2026. Domain ready, just need payment |
| Second email `hello@nexamindlabs.com` | 🟡 Medium | Add in Zoho Mail admin when needed |
| CMS login stability | 🟡 Medium | Deferred to Next.js migration phase |
| Replace placeholder testimonials | 🟡 Medium | Update `index.html` with real client/student quotes |
| Image uploads system | 🟢 Low | Currently manual GitHub upload |
| Dynamic site migration (Next.js) | 🟢 Low | When user accounts or dashboards are needed |

---

*Document maintained by NexaMind Labs. Update this file whenever a service, credential, or process changes.*  
*For content creation guidance see: `docs/CONTENT_BIBLE.md`*
