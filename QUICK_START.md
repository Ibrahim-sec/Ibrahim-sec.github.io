# Quick Start Guide

## What's Been Customized

Your portfolio has been fully customized with:

✅ **Personal Information**
- Name: Ibrahim M. Gholamy
- GitHub: @Ibrahim-sec
- Bio: Cybersecurity Student & Bug Hunter

✅ **Content**
- Homepage with your introduction
- About Me page with your experience and achievements
- 3 sample blog posts:
  - NASA Vulnerability Report
  - CVE-2025-47545 WordPress Poll Maker
  - System Override CTF 2025

✅ **Projects Showcase**
- CVE-2025-47545
- NASA Security Report
- System Override CTF 2025
- Bug Bounty Research (400+ vulnerabilities)
- HackerOne Ambassador World Cup
- BarqSec

✅ **Configuration**
- Site metadata updated
- GitHub repository links configured
- Build scripts ready for deployment

## File Structure

```
portfolio/
├── content/
│   ├── images/          # Site images
│   ├── pages/           # Static pages (About Me)
│   ├── posts/           # Blog posts
│   └── thumbnails/      # Social media icons
├── src/
│   ├── components/      # React components
│   ├── data/           # Projects list
│   ├── pages/          # Page templates
│   └── utils/          # Configuration
├── static/             # Static assets
├── gatsby-config.js    # Gatsby configuration
├── package.json        # Dependencies and scripts
├── DEPLOYMENT_GUIDE.md # Detailed deployment instructions
└── README.md          # Project documentation
```

## Next Steps

### Option 1: Deploy to GitHub Pages (Recommended)

Follow the detailed instructions in `DEPLOYMENT_GUIDE.md`

**Quick version:**
1. Create repository: `Ibrahim-sec.github.io` on GitHub
2. Run these commands in your portfolio folder:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/Ibrahim-sec/Ibrahim-sec.github.io.git
   git push -u origin main
   npm run deploy
   ```
3. Configure GitHub Pages to use `gh-pages` branch
4. Visit: https://ibrahim-sec.github.io

### Option 2: Test Locally First

```bash
# Install dependencies (if not already done)
npm install

# Start development server
npm start

# Visit http://localhost:8000
```

### Option 3: Deploy to Other Platforms

- **Netlify**: Connect your GitHub repo to Netlify
- **Vercel**: Import your GitHub repo to Vercel
- **Custom Server**: Build with `npm run build` and serve the `public/` folder

## Customization

### Add New Blog Post

Create `content/posts/YYYY-MM-DD-your-post-title.md`:

```markdown
---
date: 2025-11-05
title: 'Your Post Title'
template: post
slug: your-post-slug
categories:
  - Technical
tags:
  - cybersecurity
  - bug-bounty
---

Your content here...
```

### Update Projects

Edit `src/data/projectsList.js`

### Change Homepage

Edit `src/pages/index.js`

### Update About Page

Edit `content/pages/me.md`

## Important Commands

```bash
npm start          # Start development server
npm run build      # Build for production
npm run clean      # Clean cache
npm run deploy     # Deploy to GitHub Pages
```

## Configuration Files

- `gatsby-config.js` - Main site configuration
- `src/utils/config.js` - Site metadata
- `package.json` - Dependencies and scripts

## Support Resources

- [Gatsby Documentation](https://www.gatsbyjs.com/docs/)
- [GitHub Pages Guide](https://docs.github.com/en/pages)
- [Markdown Guide](https://www.markdownguide.org/)

---

**Your portfolio is ready!** Choose your deployment method and get it online.
