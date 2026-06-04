# Deployment Guide - ITR-1 Chat UI

## Pre-Deployment Checklist

- [ ] All dependencies installed (`npm install`)
- [ ] Production build successful (`npm run build`)
- [ ] Schema file in `public/` folder
- [ ] All tests passing
- [ ] No console errors in production build
- [ ] Accessibility audit passed
- [ ] Mobile responsive verified
- [ ] Privacy policy updated
- [ ] Security headers configured

## Build Commands

```bash
# Install dependencies
npm install

# Build for production
npm run build

# Preview production build locally
npm run preview
```

The build output will be in the `dist/` directory.

## Deployment Options

### Option 1: Vercel (Recommended)

**Why Vercel:**
- Zero configuration for Vite apps
- Automatic HTTPS
- Global CDN
- Free tier available

**Steps:**
```bash
# Install Vercel CLI
npm install -g vercel

# Deploy
cd itr-chat
vercel

# Follow prompts:
# - Set up and deploy: Yes
# - Link to existing project: No
# - Project name: itr-chat
# - Directory: ./
# - Build command: npm run build
# - Output directory: dist
```

**Environment:**
No environment variables needed for basic deployment.

**Custom Domain:**
1. Go to Vercel dashboard
2. Select project → Settings → Domains
3. Add custom domain
4. Update DNS records as instructed

### Option 2: Netlify

**Steps:**

1. **Via CLI:**
```bash
# Install Netlify CLI
npm install -g netlify-cli

# Build
npm run build

# Deploy
netlify deploy --prod --dir=dist
```

2. **Via Web UI:**
- Connect GitHub repository
- Build command: `npm run build`
- Publish directory: `dist`
- Click "Deploy site"

**netlify.toml** (optional):
```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    X-XSS-Protection = "1; mode=block"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

### Option 3: GitHub Pages

**Steps:**

1. **Install gh-pages:**
```bash
npm install --save-dev gh-pages
```

2. **Update package.json:**
```json
{
  "scripts": {
    "deploy": "npm run build && gh-pages -d dist"
  },
  "homepage": "https://yourusername.github.io/itr-chat"
}
```

3. **Deploy:**
```bash
npm run deploy
```

4. **Configure GitHub:**
- Go to repository Settings → Pages
- Source: gh-pages branch
- Wait for deployment

**Note:** Update `base` in `vite.config.js`:
```javascript
export default {
  base: '/itr-chat/'  // Your repo name
}
```

### Option 4: Cloudflare Pages

**Steps:**

1. **Via Dashboard:**
- Connect GitHub repository
- Framework preset: Vite
- Build command: `npm run build`
- Build output directory: `dist`

2. **Via Wrangler CLI:**
```bash
npm install -g wrangler

# Build
npm run build

# Deploy
wrangler pages publish dist --project-name=itr-chat
```

### Option 5: AWS S3 + CloudFront

**Steps:**

1. **Build:**
```bash
npm run build
```

2. **Create S3 Bucket:**
```bash
aws s3 mb s3://itr-chat-app
aws s3 website s3://itr-chat-app --index-document index.html
```

3. **Upload:**
```bash
aws s3 sync dist/ s3://itr-chat-app --acl public-read
```

4. **Create CloudFront Distribution:**
- Origin: S3 bucket
- Viewer Protocol Policy: Redirect HTTP to HTTPS
- Default Root Object: index.html

5. **Custom Error Pages:**
- 403 → /index.html (for SPA routing)
- 404 → /index.html

### Option 6: Self-Hosted (Nginx)

**Steps:**

1. **Build:**
```bash
npm run build
```

2. **Copy to Server:**
```bash
scp -r dist/* user@server:/var/www/itr-chat/
```

3. **Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name itr-chat.yourdomain.com;

    root /var/www/itr-chat;
    index index.html;

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Gzip compression
    gzip on;
    gzip_types text/css application/javascript application/json;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|json)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

4. **Enable HTTPS (Let's Encrypt):**
```bash
sudo certbot --nginx -d itr-chat.yourdomain.com
```

## Post-Deployment Configuration

### 1. Security Headers

Add to your hosting platform:

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:;
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

### 2. Performance Optimization

**Vite Build Optimizations** (already configured):
```javascript
// vite.config.js
export default {
  build: {
    minify: 'terser',
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'charts': ['recharts']
        }
      }
    }
  }
}
```

**Enable Compression:**
- Gzip/Brotli at server level
- Most hosting platforms enable automatically

### 3. Analytics (Optional)

**Plausible (Privacy-focused):**
```html
<!-- In index.html -->
<script defer data-domain="yourdomain.com" 
  src="https://plausible.io/js/plausible.js"></script>
```

**Google Analytics (if needed):**
```html
<!-- In index.html -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>
```

### 4. Error Monitoring

**Sentry:**
```bash
npm install @sentry/react
```

```javascript
// main.jsx
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: "YOUR_SENTRY_DSN",
  environment: "production"
});
```

### 5. Custom Domain Setup

**DNS Configuration:**
```
Type    Name    Value
A       @       [Your IP]
CNAME   www     [Your domain]
```

**SSL Certificate:**
- Automatic with Vercel/Netlify
- Let's Encrypt for self-hosted
- ACM for AWS

## Environment-Specific Builds

### Development
```bash
npm run dev
```

### Staging
```bash
VITE_ENV=staging npm run build
```

### Production
```bash
VITE_ENV=production npm run build
```

**Access in code:**
```javascript
const env = import.meta.env.VITE_ENV;
```

## Monitoring & Maintenance

### Health Checks

**Basic Status Page:**
Create `public/health.json`:
```json
{
  "status": "ok",
  "version": "1.0.0",
  "timestamp": "2025-11-12T00:00:00Z"
}
```

**Monitor:**
- Uptime Robot
- Pingdom
- Status Cake

### Updates

**Schema Updates:**
1. Replace `public/ITR-1_2025_Main_V1.1.json`
2. Test locally
3. Deploy

**Dependency Updates:**
```bash
# Check outdated
npm outdated

# Update safely
npm update

# Major version updates
npm install package@latest
```

## Rollback Procedure

### Vercel
```bash
# List deployments
vercel ls

# Promote previous deployment
vercel promote [deployment-url]
```

### Netlify
- Go to Deploys tab
- Find previous successful deploy
- Click "Publish deploy"

### GitHub Pages
```bash
# Revert to previous commit
git revert HEAD
git push origin main
npm run deploy
```

## Troubleshooting

### Build Fails
```bash
# Clear cache
rm -rf node_modules package-lock.json
npm install
npm run build
```

### 404 on Refresh
- Configure SPA fallback (see hosting-specific configs above)

### Static Assets Not Loading
- Check `base` path in vite.config.js
- Verify assets copied to dist/

### Schema File 404
- Ensure `public/ITR-1_2025_Main_V1.1.json` exists
- Check file is included in build output

## Production Checklist

Before going live:

- [ ] Test schema loading
- [ ] Test all form fields
- [ ] Test validation (valid + invalid inputs)
- [ ] Test download functionality
- [ ] Test on Chrome, Firefox, Safari, Edge
- [ ] Test on mobile (iOS + Android)
- [ ] Test localStorage persistence
- [ ] Verify no API keys exposed
- [ ] Check all console logs removed
- [ ] Verify error handling works
- [ ] Test with slow 3G network
- [ ] Run Lighthouse audit (score > 90)
- [ ] Test accessibility with screen reader
- [ ] Verify GDPR compliance (if applicable)
- [ ] Set up monitoring/alerts
- [ ] Document known issues

## Performance Targets

- **First Contentful Paint:** < 1.5s
- **Time to Interactive:** < 3.5s
- **Lighthouse Score:** > 90
- **Bundle Size:** < 500KB (gzipped)
- **API Response:** < 200ms (if backend added)

## Backup & Recovery

### Backup User Data
If adding backend:
```bash
# Backup database
pg_dump dbname > backup.sql

# Backup user files
tar -czf backup.tar.gz /path/to/data
```

### Recovery
```bash
# Restore database
psql dbname < backup.sql

# Restore files
tar -xzf backup.tar.gz -C /path/to/restore
```

## Legal & Compliance

### Privacy Policy
Update with:
- Data collection practices (localStorage)
- No server-side storage (if true)
- Third-party services used (analytics, etc.)

### Terms of Service
Include:
- Disclaimer about tax filing accuracy
- Recommendation to consult tax professional
- No warranty clause

### License
Add LICENSE file:
```
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge...
```

## Support & Documentation

### User Support
- Create FAQ page
- Add help tooltips in app
- Provide contact email

### Technical Support
- GitHub Issues (if open source)
- Email support
- Documentation wiki

## Cost Estimates

**Free Tier (Hobby):**
- Vercel: Free (100GB bandwidth/month)
- Netlify: Free (100GB bandwidth/month)
- GitHub Pages: Free

**Paid Tier (Production):**
- Vercel Pro: $20/month
- Netlify Pro: $19/month
- AWS: ~$5-50/month (varies with traffic)

---

**Ready to Deploy?** Follow the checklist and choose your hosting option. Start with Vercel for easiest deployment!
