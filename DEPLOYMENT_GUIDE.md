# Deployment Guide for GitHub Pages

This guide will help you deploy your customized portfolio to GitHub Pages.

## Prerequisites

1. **GitHub Account**: You need access to your GitHub account (@Ibrahim-sec)
2. **GitHub Repository**: Create a repository named `Ibrahim-sec.github.io` (this is your GitHub Pages user site)
3. **Git Installed**: Git should be installed on your local machine

## Step-by-Step Deployment Instructions

### Step 1: Create GitHub Repository

1. Go to [GitHub](https://github.com)
2. Click the "+" icon in the top right and select "New repository"
3. Name it **exactly**: `Ibrahim-sec.github.io`
4. Make it **Public** (required for free GitHub Pages)
5. Do NOT initialize with README, .gitignore, or license (we already have these)
6. Click "Create repository"

### Step 2: Download and Upload Your Portfolio

Since this portfolio was built in a sandbox environment, you'll need to:

1. **Download the portfolio folder** from this session
2. **Extract it** on your local machine
3. **Navigate to the folder** in your terminal/command prompt

### Step 3: Connect to GitHub and Deploy

Open a terminal in your portfolio folder and run these commands:

```bash
# Initialize git (if not already done)
git init

# Configure git with your information
git config user.name "Ibrahim M. Gholamy"
git config user.email "your-email@example.com"

# Add all files
git add .

# Commit the changes
git commit -m "Initial commit: Customized portfolio"

# Add your GitHub repository as remote
git remote add origin https://github.com/Ibrahim-sec/Ibrahim-sec.github.io.git

# Push to main branch
git branch -M main
git push -u origin main

# Deploy to GitHub Pages
npm run deploy
```

### Step 4: Configure GitHub Pages

1. Go to your repository on GitHub: `https://github.com/Ibrahim-sec/Ibrahim-sec.github.io`
2. Click on "Settings" tab
3. Scroll down to "Pages" in the left sidebar
4. Under "Source", select branch: `gh-pages`
5. Click "Save"

### Step 5: Access Your Website

Your portfolio will be live at:
**https://ibrahim-sec.github.io**

It may take 2-5 minutes for the site to be published after the first deployment.

## Updating Your Portfolio

Whenever you want to update your portfolio:

```bash
# Make your changes to files
# Then commit and deploy:

git add .
git commit -m "Update: describe your changes"
git push origin main
npm run deploy
```

## Customization Tips

### Adding Blog Posts

Create new markdown files in `content/posts/`:

```markdown
---
date: 2025-11-05
title: 'Your Post Title'
template: post
slug: your-post-slug
categories:
  - Technical  # or Personal
tags:
  - tag1
  - tag2
---

Your content here...
```

### Updating Projects

Edit `src/data/projectsList.js` to add or modify projects.

### Changing About Page

Edit `content/pages/me.md` to update your about page.

### Modifying Homepage

Edit `src/pages/index.js` to change the homepage content.

## Troubleshooting

### Site Not Loading
- Wait 5-10 minutes after first deployment
- Check GitHub Pages settings (Settings → Pages)
- Ensure gh-pages branch exists and is selected

### Build Errors
- Run `npm run clean` then `npm run build`
- Check for missing images in content/images/
- Verify all markdown files have proper frontmatter

### Images Not Showing
- Place images in `content/images/` or `static/`
- Reference them correctly in markdown: `![alt](../images/image.png)`

## Custom Domain (Optional)

To use a custom domain like `gholamy.com`:

1. Buy a domain from a registrar
2. Add a `CNAME` file in the `static/` folder with your domain
3. Configure DNS settings at your registrar:
   - Add a CNAME record pointing to `ibrahim-sec.github.io`
4. In GitHub repository settings → Pages, add your custom domain

## Support

For issues with:
- **GitHub Pages**: Check [GitHub Pages Documentation](https://docs.github.com/en/pages)
- **Gatsby**: Check [Gatsby Documentation](https://www.gatsbyjs.com/docs/)
- **Portfolio Customization**: Review the code and make changes as needed

## Important Files

- `gatsby-config.js` - Site metadata and configuration
- `src/utils/config.js` - Site title and URL
- `src/pages/index.js` - Homepage
- `src/data/projectsList.js` - Projects list
- `content/pages/me.md` - About page
- `content/posts/` - Blog posts

---

**Your portfolio is ready to deploy!** Follow the steps above to get it live on GitHub Pages.
