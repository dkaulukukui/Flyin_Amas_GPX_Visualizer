# Deployment Guide

## Quick Deploy to GitHub Pages

### 1. Commit Your Changes

```bash
git add .
git commit -m "Organize for GitHub Pages deployment"
```

### 2. Push to GitHub

If this is a new repository:
```bash
# Create a new repository on github.com first, then:
git remote add origin https://github.com/YOURUSERNAME/Flyin_Amas_GPX_Visializer.git
git branch -M main
git push -u origin main
```

If you already have a repository:
```bash
git push origin master
```

### 3. Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** tab
3. Scroll to **Pages** section (left sidebar)
4. Under **Source**, select:
   - Branch: `main` (or `master`)
   - Folder: `/ (root)`
5. Click **Save**

### 4. Access Your Site

Your site will be available at:
```
https://YOURUSERNAME.github.io/Flyin_Amas_GPX_Visializer/
```

It may take 1-2 minutes for the initial deployment.

## Alternative: Quick Test Locally

```bash
# Python 3
python -m http.server 8000

# Then open: http://localhost:8000
```

## Updating Your Site

After making changes:
```bash
git add .
git commit -m "Description of changes"
git push
```

GitHub Pages will automatically redeploy (takes 1-2 minutes).

## Custom Domain (Optional)

1. Buy a domain (e.g., from Namecheap, Google Domains)
2. Add a `CNAME` file to repository root with your domain
3. Configure DNS records with your domain provider
4. In GitHub Settings > Pages, add your custom domain

## Troubleshooting

**Site not loading?**
- Wait 2-3 minutes after first deployment
- Check Settings > Pages shows green "Your site is published"
- Verify you're using the correct URL

**Video export not working?**
- Ensure you're accessing via HTTPS (GitHub Pages URL)
- Try from a different browser (Chrome recommended)
- Check browser console for errors

**CORS issues?**
- Only occur when opening `index.html` directly (file://)
- GitHub Pages (HTTPS) resolves this automatically
