# How to Set Up a Hugo Blog on Cloudflare Pages

A complete guide to creating a fast, free technical blog with automatic deployments from Git.

---

## Table of Contents

1. [Why Hugo + Cloudflare Pages?](#why-hugo--cloudflare-pages)
2. [Prerequisites](#prerequisites)
3. [Installation & Setup](#installation--setup)
4. [Configuration](#configuration)
5. [Creating Your First Post](#creating-your-first-post)
6. [Publishing Workflow](#publishing-workflow)
7. [Customization Tips](#customization-tips)
8. [Troubleshooting](#troubleshooting)

---

## Why Hugo + Cloudflare Pages?

### Why Hugo?

**Hugo** is a static site generator built in Go that's perfect for technical blogs:

**Advantages:**
- ⚡ **Blazing fast builds** - Compiles thousands of pages in seconds
- 📦 **Single binary** - No Node.js or complex dependencies
- 📝 **Content-focused** - Built specifically for blogs and documentation
- 🎨 **400+ themes** - Mature ecosystem with beautiful, tested themes
- 🔒 **Stable & reliable** - Battle-tested since 2013
- 💰 **Zero runtime** - Pure HTML/CSS, no server needed
- ✍️ **Markdown native** - Write in simple, portable Markdown format
- 🚀 **Easy migration** - When ready, import Markdown to Ghost or other platforms

**Perfect for:**
- Technical blogs with code snippets
- Documentation sites
- Portfolio sites
- Project journals (like documenting a K3s cluster build!)

### Why Cloudflare Pages?

**Cloudflare Pages** provides free, global hosting with incredible features:

**Benefits:**
- 💵 **Free hosting** - No cost for personal projects
- 🌍 **Global CDN** - Fast load times worldwide
- 🔐 **Automatic HTTPS** - Built-in SSL certificates
- 🔄 **Auto-deploy from Git** - Push to GitHub, instant deployment
- 🔀 **Preview deployments** - Every branch gets a preview URL
- 🎯 **Custom domains** - Use your own domain easily
- 📊 **Analytics** - Built-in Web Analytics (optional)
- ⚡ **Edge computing** - Cloudflare's global network

**Comparison to alternatives:**
- **GitHub Pages:** Hugo support, but Cloudflare has better CDN
- **Netlify:** Similar features, but Cloudflare has more generous free tier
- **Vercel:** Great for React apps, but Hugo + Cloudflare is simpler for static blogs
- **Self-hosting:** You'll get there! But start here while building your cluster

---

## Prerequisites

- **Windows, Mac, or Linux computer**
- **Git** installed ([download here](https://git-scm.com/))
- **GitHub account** ([sign up here](https://github.com/))
- **Cloudflare account** ([sign up here](https://dash.cloudflare.com/sign-up))
- **Text editor** (VS Code, Notepad++, or any code editor)
- **Optional:** Domain name already registered in Cloudflare

---

## Installation & Setup

### Step 1: Install Hugo

#### Windows (PowerShell)

**Option A: Using Chocolatey (Recommended)**
```powershell
# Install Chocolatey first (if not installed)
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Hugo Extended
choco install hugo-extended -y
```

**Option B: Direct Download**
1. Download from: https://github.com/gohugoio/hugo/releases/latest
2. Look for: `hugo_extended_X.XXX.X_windows-amd64.zip`
3. Extract to `C:\Hugo\bin`
4. Add `C:\Hugo\bin` to your PATH environment variable

**Option C: Using Winget (Windows 11)**
```powershell
winget install Hugo.Hugo.Extended
```

#### Mac
```bash
brew install hugo
```

#### Linux
```bash
# Snap
snap install hugo

# Or download from releases
wget https://github.com/gohugoio/hugo/releases/download/vX.XX.X/hugo_extended_X.XX.X_Linux-64bit.tar.gz
tar -xvf hugo_extended_X.XX.X_Linux-64bit.tar.gz
sudo mv hugo /usr/local/bin/
```

**Verify Installation:**
```bash
hugo version
# Should output: hugo v0.xxx.x-extended
```

---

### Step 2: Create Your Hugo Site

```bash
# Create new Hugo site
hugo new site my-blog
cd my-blog

# Initialize Git repository
git init

# Add PaperMod theme (excellent for technical blogs)
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**Why PaperMod?**
- Clean, minimalist design
- Perfect for technical content
- Excellent syntax highlighting
- Fast performance
- Built-in search functionality
- Responsive (mobile-friendly)
- Active development and support

---

### Step 3: Configure Your Site

Create or edit `hugo.toml` (or `config.toml`) in your root directory:

```toml
baseURL = 'https://blog.yourdomain.com/'
languageCode = 'en-us'
title = 'My Technical Blog'
theme = 'PaperMod'

[params]
  description = "My journey building and learning - documenting everything along the way"
  author = "Your Name"
  
  # Enable features
  showReadingTime = true
  showShareButtons = true
  showPostNavLinks = true
  showBreadCrumbs = true
  showCodeCopyButtons = true
  showRssButtonInSectionTermList = true
  
  # Home page info
  [params.homeInfoParams]
    Title = "Welcome to My Blog"
    Content = """
    Follow along as I document my learning journey. 
    I share technical insights, lessons learned, and detailed guides.
    """

  # Social links
  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/yourusername"
  
  [[params.socialIcons]]
    name = "linkedin"
    url = "https://linkedin.com/in/yourusername"
  
  [[params.socialIcons]]
    name = "email"
    url = "mailto:your@email.com"
  
  [[params.socialIcons]]
    name = "rss"
    url = "/index.xml"

# Navigation menu
[menu]
  [[menu.main]]
    name = "Archives"
    url = "/archives/"
    weight = 10
  
  [[menu.main]]
    name = "Tags"
    url = "/tags/"
    weight = 20
  
  [[menu.main]]
    name = "About"
    url = "/about/"
    weight = 30

# Enable search and RSS
[outputs]
  home = ["HTML", "RSS", "JSON"]

# Syntax highlighting for code blocks
[markup]
  [markup.highlight]
    style = "monokai"
    lineNos = true
    lineNumbersInTable = false
```

**Key Configuration Sections:**
- **baseURL:** Your final blog URL (update after Cloudflare setup)
- **params:** Theme-specific settings and features
- **menu:** Navigation links that appear in header
- **markup.highlight:** Code syntax highlighting style

---

### Step 4: Create Your First Post

```bash
hugo new posts/introduction.md
```

This creates `content/posts/introduction.md`. Edit it with your content:

```markdown
---
title: "Welcome to My Blog"
date: 2025-11-04
draft: false
tags: ["introduction", "meta"]
categories: ["General"]
author: "Your Name"
description: "Introduction to my blog and what I'll be writing about"
---

## Hello World!

Welcome to my blog where I'll be documenting my journey learning and building cool things.

### What to Expect

I'll be sharing:
- Technical tutorials and guides
- Lessons learned from real projects
- Deep dives into interesting topics
- Code snippets and examples

### Why I'm Blogging

Documenting my learning journey helps me:
1. Solidify my understanding
2. Share knowledge with the community
3. Create a reference for my future self
4. Connect with like-minded people

Stay tuned for more posts!
```

---

### Step 5: Test Locally

```bash
hugo server -D
```

**What this does:**
- Starts local development server
- `-D` flag includes draft posts
- Auto-reloads when you save changes
- Visit: http://localhost:1313

**You should see:**
- Your blog homepage
- Your first post
- Working navigation

Press `Ctrl+C` to stop the server.

---

### Step 6: Push to GitHub

```bash
# Create .gitignore file
echo "public/
resources/
.hugo_build.lock
.DS_Store" > .gitignore

# Add all files
git add .

# Commit
git commit -m "Initial Hugo blog setup"

# Create a new repository on GitHub
# Then connect and push:
git remote add origin https://github.com/yourusername/your-blog-repo.git
git branch -M main
git push -u origin main
```

**GitHub Repository Setup:**
1. Go to: https://github.com/new
2. Create repository (public or private)
3. Don't initialize with README (you already have one)
4. Copy the remote URL
5. Use commands above to push

---

### Step 7: Deploy to Cloudflare Pages

#### In Cloudflare Dashboard:

1. **Navigate to Pages**
   - Log into Cloudflare Dashboard
   - Click "Workers & Pages" in sidebar
   - Click "Create application"
   - Choose "Pages" tab
   - Click "Connect to Git"

2. **Connect GitHub**
   - Authorize Cloudflare to access GitHub
   - Select your blog repository
   - Click "Begin setup"

3. **Configure Build Settings**
   - **Project name:** your-blog (becomes subdomain)
   - **Production branch:** main
   - **Framework preset:** Hugo
   - **Build command:** `hugo --minify`
   - **Build output directory:** `public`
   
4. **Add Environment Variable**
   - Click "Add variable"
   - Variable name: `HUGO_VERSION`
   - Value: `0.120.0` (or your installed version from `hugo version`)
   
5. **Save and Deploy**
   - Click "Save and Deploy"
   - Wait 1-3 minutes for first build
   - You'll get a URL like: `your-blog.pages.dev`

#### Verify Deployment:
- Visit your `*.pages.dev` URL
- Your blog should be live!
- Check build logs if anything fails

---

### Step 8: Add Custom Domain (Optional)

If you have a domain registered in Cloudflare:

1. **In your Pages project:**
   - Go to "Custom domains" tab
   - Click "Set up a custom domain"
   - Enter your domain (e.g., `blog.yourdomain.com`)
   - Click "Continue"

2. **DNS Configuration:**
   - Cloudflare automatically creates DNS records
   - Wait 1-5 minutes for DNS propagation
   - SSL certificate is automatically provisioned

3. **Update hugo.toml:**
   ```toml
   baseURL = 'https://blog.yourdomain.com/'
   ```
   - Commit and push this change
   - Cloudflare auto-deploys

**Your blog is now live on your custom domain! 🎉**

---

## Understanding Front Matter

Front matter is the metadata at the top of each post, enclosed in `---`. It's **critical** to format it correctly.

### Front Matter Formats

Hugo supports three formats:

#### YAML (Recommended - Most Common)
```markdown
---
title: "My Post Title"
date: 2025-11-04
draft: false
tags: ["kubernetes", "tutorial"]
categories: ["Infrastructure"]
---
```
**Characteristics:**
- Uses colons: `key: value`
- More forgiving with syntax
- Most popular in Hugo community

#### TOML
```markdown
---
title = "My Post Title"
date = 2025-11-04
draft = false
tags = ["kubernetes", "tutorial"]
categories = ["Infrastructure"]
---
```
**Characteristics:**
- Uses equals: `key = value`
- Stricter syntax requirements
- Hugo's default when using `hugo new`

#### JSON
```markdown
---
{
  "title": "My Post Title",
  "date": "2025-11-04",
  "draft": false,
  "tags": ["kubernetes", "tutorial"]
}
---
```
**Characteristics:**
- Strict JSON syntax
- Rarely used for Hugo
- Better for programmatic generation

### Common Front Matter Fields

```yaml
---
# Required fields
title: "Your Post Title"           # Post title (required)
date: 2025-11-04T10:00:00-08:00   # Publication date (required)

# Publishing control
draft: false                       # true = not published, false = published
publishDate: 2025-11-05           # Optional: schedule for future

# Content organization
tags: ["kubernetes", "tutorial"]   # Array of tags
categories: ["Infrastructure"]     # Array of categories
series: ["K3s Journey"]           # Group related posts

# Metadata
description: "A short summary"     # Meta description for SEO
author: "Your Name"               # Post author
cover:
  image: "/images/cover.jpg"      # Featured image
  alt: "Image description"        # Alt text for accessibility
  caption: "Image caption"        # Optional caption

# Social
showToc: true                     # Show table of contents
TocOpen: false                    # TOC collapsed by default
hidemeta: false                   # Show post metadata
comments: true                    # Enable comments (if configured)
disableShare: false               # Show share buttons

# SEO
keywords: ["k3s", "kubernetes"]   # Additional keywords
url: "/custom-url-slug/"          # Custom URL (optional)
---
```

### Front Matter Best Practices

1. **Always Use Consistent Format**
   - Pick YAML or TOML, stick with it
   - YAML is recommended for beginners

2. **Date Format Matters**
   ```yaml
   date: 2025-11-04              # Simple (uses site timezone)
   date: 2025-11-04T14:30:00Z    # Specific time UTC
   date: 2025-11-04T09:30:00-08:00  # Specific timezone
   ```

3. **Draft Control**
   ```yaml
   draft: true   # Won't appear in production
   draft: false  # Will be published
   ```

4. **Tags vs Categories**
   - **Categories:** Broad topics (Infrastructure, Development, Tutorials)
   - **Tags:** Specific keywords (kubernetes, python, docker)

5. **Quote Complex Strings**
   ```yaml
   title: "Using 'Quotes' in: Titles"  # Wrap in quotes if special chars
   description: "Simple text works"    # Always safe to quote
   ```

### Common Front Matter Errors

**❌ Wrong:**
```yaml
---
title: My Post: A Guide  # Colon breaks YAML
date: 11/04/2025        # Wrong date format
draft = false           # Using TOML syntax in YAML
tags: kubernetes        # Should be array
---
```

**✅ Correct:**
```yaml
---
title: "My Post: A Guide"
date: 2025-11-04
draft: false
tags: ["kubernetes"]
---
```

### Testing Front Matter

Run local server and check for errors:
```bash
hugo server -D

# Look for errors like:
# Error: failed to parse front matter
# unmarshal failed: yaml: line X
```

**Pro Tip:** Use a YAML validator online if you're unsure: https://www.yamllint.com/

---

## Publishing Workflow

Once your blog is deployed to Cloudflare Pages, publishing new content is simple.

### Standard Workflow

#### 1. Create New Post
```bash
hugo new posts/my-new-post.md
```

#### 2. Write Your Content
```markdown
---
title: "My Awesome Post"
date: 2025-11-04
draft: true  # Keep as draft while writing
tags: ["tutorial", "kubernetes"]
---

## Introduction

Your content here...
```

#### 3. Preview Locally
```bash
hugo server -D
# Visit http://localhost:1313
# Make sure everything looks good
```

#### 4. Mark as Ready
```markdown
---
title: "My Awesome Post"
date: 2025-11-04
draft: false  # Changed from true
tags: ["tutorial", "kubernetes"]
---
```

#### 5. Commit and Push
```bash
git add content/posts/my-new-post.md
git commit -m "Publish: My Awesome Post"
git push origin main
```

#### 6. Automatic Deployment
- Cloudflare detects the push
- Runs `hugo --minify`
- Builds your site
- Deploys to production
- **Live in 1-3 minutes! 🚀**

### Quick Edits

For small typos or fixes:

```bash
# Edit the file directly
# Then:
git add .
git commit -m "Fix typo in latest post"
git push origin main
```

**Or edit directly on GitHub:**
1. Navigate to file on GitHub
2. Click pencil icon to edit
3. Commit changes
4. Auto-deploys in ~2 minutes

### Using Branches for Drafts

Work on multiple drafts without publishing:

```bash
# Create draft branch
git checkout -b draft/new-feature-post

# Write and commit
git add content/posts/new-feature.md
git commit -m "Draft: New feature post"
git push origin draft/new-feature-post
```

**Benefits:**
- Cloudflare creates preview URL for each branch
- Test before merging to main
- Collaborate with others on drafts

**When ready to publish:**
```bash
git checkout main
git merge draft/new-feature-post
git push origin main
# Goes live!
```

### Scheduling Posts

Set future date in front matter:

```yaml
---
title: "Scheduled Post"
date: 2025-11-10T09:00:00Z
draft: false
---
```

**How it works:**
- Post won't show until date/time arrives
- Requires a new build after that time
- Push any change to trigger rebuild
- Or set up GitHub Actions for automated builds

### Content Organization

```
content/
├── posts/
│   ├── 2025/
│   │   ├── 11/
│   │   │   ├── my-first-post.md
│   │   │   └── another-post.md
│   │   └── 12/
│   │       └── december-post.md
│   └── introduction.md
├── about.md
└── projects.md
```

**Benefits:**
- Organize by date
- Easy to find old posts
- Clean structure as blog grows

### Workflow Tips

1. **Write First, Publish Later**
   - Keep `draft: true` while writing
   - Change to `false` only when ready

2. **Use Meaningful Commit Messages**
   - `git commit -m "Publish: Phase 1 hardware setup"`
   - `git commit -m "Update: Fix code examples in K3s post"`
   - `git commit -m "Draft: Working on monitoring guide"`

3. **Preview Everything Locally**
   - Always run `hugo server -D` before pushing
   - Check formatting, images, links
   - Catch errors before deployment

4. **Check Cloudflare Build Logs**
   - If deployment fails, check logs
   - Common issues: front matter errors, missing images
   - Fix and push again

5. **Keep Images Organized**
   ```
   static/
   ├── images/
   │   ├── 2025/
   │   │   └── 11/
   │   │       ├── post-1-diagram.png
   │   │       └── post-1-screenshot.png
   │   └── logo.png
   └── favicon.ico
   ```

---

## Customization Tips

### Adding a Logo

#### Option 1: Simple Icon/Emoji (No Image File)
```toml
# In hugo.toml
[params]
  label.text = "🚀 My Blog"
  # or
  label.text = "⚙️ Tech Journey"
```

#### Option 2: Image Logo
```toml
# In hugo.toml
[params]
  label.text = "My Blog"
  label.icon = "/logo.png"
  label.iconHeight = 35
```

**Then:**
1. Create `static/` folder in your Hugo root
2. Add your logo: `static/logo.png`
3. Recommended size: 32-40px height
4. Format: PNG with transparency or SVG

#### Option 3: Custom CSS for Logo
Create `static/css/custom.css`:

```css
.logo a {
    display: flex;
    align-items: center;
    gap: 10px;
    font-weight: 600;
    font-size: 1.2rem;
}

.logo img {
    border-radius: 4px;
    transition: transform 0.2s;
}

.logo img:hover {
    transform: scale(1.05);
}
```

Reference in `hugo.toml`:
```toml
[params]
  assets.customCSS = ["css/custom.css"]
```

### Custom Colors

Create `assets/css/extended/custom.css`:

```css
:root {
    --primary: #FF8C42;
    --secondary: #00D9FF;
    --theme: var(--primary);
    --entry: #2e2e2e;
    --code-bg: #1e1e1e;
}

/* Custom link colors */
.post-content a {
    color: var(--primary);
}

.post-content a:hover {
    color: var(--secondary);
}

/* Custom button style */
.custom-button {
    background: var(--primary);
    color: white;
    padding: 10px 20px;
    border-radius: 5px;
    transition: all 0.3s;
}
```

### Adding Custom Pages

```bash
# Create About page
hugo new about.md

# Create Projects page
hugo new projects.md
```

Edit `about.md`:
```markdown
---
title: "About"
url: "/about/"
---

## About Me

Your bio and information here.
```

Add to menu in `hugo.toml`:
```toml
[[menu.main]]
  name = "About"
  url = "/about/"
  weight = 40
```

### Code Block Themes

Change syntax highlighting in `hugo.toml`:

```toml
[markup]
  [markup.highlight]
    style = "monokai"     # Options: monokai, dracula, github, 
                          # nord, gruvbox, solarized-dark
    lineNos = true        # Show line numbers
    lineNumbersInTable = false
```

**Popular styles:**
- `monokai` - Dark, colorful
- `dracula` - Purple-ish dark theme
- `github` - GitHub-like light theme
- `nord` - Blue-tinted dark theme
- `gruvbox` - Retro dark theme

### Adding Search

PaperMod has built-in search. Enable it:

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]  # JSON needed for search

[params]
  [params.fuseOpts]
    isCaseSensitive = false
    shouldSort = true
    location = 0
    distance = 1000
    threshold = 0.4
```

Add search page:
```bash
hugo new search.md
```

Edit `content/search.md`:
```markdown
---
title: "Search"
layout: "search"
---
```

Add to menu:
```toml
[[menu.main]]
  name = "Search"
  url = "/search/"
  weight = 30
```

### Social Media Cards

Add to `hugo.toml` for better social sharing:

```toml
[params]
  images = ["/og-image.png"]  # Default share image
  
[params.homeInfoParams]
  Title = "My Blog"
  Content = "Technical blog about building cool things"
  
[params.profileMode]
  enabled = false
```

Create `static/og-image.png` (1200x630px recommended)

### Analytics

Add Cloudflare Web Analytics (free, privacy-friendly):

```toml
[params]
  [params.analytics.cloudflare]
    token = "your-beacon-token"
```

Get token from: Cloudflare Dashboard → Web Analytics → Add Site

### Comments

Integrate comment systems:

**Option 1: Giscus (GitHub Discussions)**
```toml
[params.giscus]
  repo = "yourusername/blog-comments"
  repoId = "your-repo-id"
  category = "Announcements"
  categoryId = "your-category-id"
  mapping = "pathname"
  theme = "dark"
```

**Option 2: Utterances (GitHub Issues)**
```toml
[params.utteranc]
  repo = "yourusername/blog-comments"
  issueTerm = "pathname"
  theme = "github-dark"
```

### Favicon

Create favicon files and add to `static/`:
```
static/
├── favicon.ico
├── favicon-16x16.png
├── favicon-32x32.png
└── apple-touch-icon.png
```

Hugo automatically detects and uses them!

---

## Troubleshooting

### Build Fails on Cloudflare

**Problem:** "Hugo not found" or version mismatch

**Solution:**
1. Check `HUGO_VERSION` environment variable
2. Set to your local version: `hugo version`
3. In Cloudflare Pages → Settings → Environment variables
4. Add or update `HUGO_VERSION`

---

**Problem:** Theme not found

**Solution:**
```bash
# Ensure theme is committed
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# Or if submodule exists but empty:
git submodule update --init --recursive

# Commit and push
git add .
git commit -m "Fix theme submodule"
git push
```

---

**Problem:** Front matter parsing error

**Solution:**
- Check front matter syntax (YAML vs TOML)
- Validate at: https://www.yamllint.com/
- Look for unbalanced quotes
- Ensure proper date format: `YYYY-MM-DD`

---

### Posts Not Showing

**Problem:** Published post doesn't appear

**Check:**
1. Is `draft: false`?
2. Is date in the past?
3. Did deployment succeed?
4. Clear browser cache (Ctrl+F5)

---

### Images Not Loading

**Problem:** Images show broken link

**Solution:**
- Put images in `static/images/`
- Reference as: `/images/myimage.png`
- Check file name case sensitivity
- Ensure images are committed to Git

---

### Local Server Issues

**Problem:** Port 1313 already in use

**Solution:**
```bash
hugo server -D -p 1314  # Use different port
```

---

**Problem:** Changes not reflecting

**Solution:**
```bash
# Stop server (Ctrl+C)
# Clean build files
hugo mod clean
rm -rf public/ resources/

# Restart
hugo server -D
```

---

### Deployment is Slow

**Normal:** First deployment takes 2-5 minutes
**Subsequent:** Usually 1-2 minutes

**If taking >5 minutes:**
- Check Cloudflare status page
- Review build logs for errors
- Verify build command is correct

---

## Next Steps

### Enhance Your Blog

1. **Add more posts** - Document your journey!
2. **Customize theme** - Make it your own
3. **Add analytics** - Track your readers
4. **Enable comments** - Engage with audience
5. **Optimize images** - Compress for faster loading
6. **Add RSS feed** - Already included by default!

### Learn More

**Hugo Resources:**
- Official Docs: https://gohugo.io/documentation/
- PaperMod Wiki: https://github.com/adityatelange/hugo-PaperMod/wiki
- Hugo Discourse: https://discourse.gohugo.io/

**Cloudflare Pages:**
- Documentation: https://developers.cloudflare.com/pages/
- Community: https://community.cloudflare.com/

### Migration Path

When your self-hosted infrastructure is ready:

1. **Export content** - All posts are in Markdown (portable!)
2. **Deploy Ghost** - Follow your K3s implementation plan
3. **Import posts** - Use Ghost's Markdown import
4. **Update DNS** - Point domain to new infrastructure
5. **Keep Cloudflare** - Use it as staging/backup site

---

## Conclusion

You now have:
- ✅ A fast, free Hugo blog
- ✅ Automatic deployments from Git
- ✅ Global CDN and HTTPS
- ✅ Scalable workflow for publishing
- ✅ Foundation for future self-hosting

**Your publishing workflow is now:**
1. Write Markdown
2. Push to GitHub
3. Live in 2 minutes! 🚀

**Happy blogging!** Share your knowledge, document your journey, and build in public.

---

## About This Guide

This guide was created to help technical bloggers get started quickly with a modern, fast, and free blogging platform. Whether you're documenting a homelab project, sharing tutorials, or building a portfolio, Hugo + Cloudflare Pages gives you a professional foundation.

**Questions or improvements?** Open an issue or submit a PR on the GitHub repo for this guide!

---

*Last updated: November 2025*
#picluster