# CAREER PILOT — PRODUCTION DEPLOYMENT & OPERATIONS MANUAL

## 1. Architectural Overview & Hosting Requirements

Career Pilot is built as a **modern, local-first Single-Page Application (SPA)** using React 19, TypeScript, and Vite.

### Key Deployment Characteristics:
- **Zero Server-Side State Required**: All calculations, cognitive evaluations, career matching algorithms, and pathway generation graph traversals execute deterministically client-side in the user's browser.
- **Zero Paid AI API Dependencies**: Does not require API keys or token limits from OpenAI, Google Gemini, Anthropic, or external inference APIs.
- **Client-Side Storage Enclave**: User profiles, custom signals, bookmarks, and milestone progress persist strictly in browser HTML5 LocalStorage with zero external telemetry.
- **Static Asset Serving**: The build output in `/dist` consists entirely of static HTML, CSS, JavaScript chunks, and SVG assets. It can be served from any global Edge CDN or static web server.

---

## 2. Pre-Deployment Verification Checklist

Before deploying any production release, execute the automated verification pipeline:

```bash
# 1. Static code quality and lint check (oxlint)
npm run lint

# 2. Comprehensive automated test suite (63/63 tests passing)
npm test -- --run

# 3. Production optimized bundle compilation (tsc -b && vite build)
npm run build
```

The build compiles into the `/dist` directory in approximately 300–400ms:
- `dist/index.html` (~1.2 kB)
- `dist/assets/index-*.css` (~64 kB, gzip: ~10 kB)
- Code-split route chunks: `DiscoveryView`, `ProfileView`, `AtlasView`, `PathwaysView`, `CompareView` (each < 100 kB)
- Vendor chunks: `vendor-react` (~210 kB, gzip: ~65 kB), `vendor-icons` (~23 kB, gzip: ~8 kB)

---

## 3. Deployment Guides

### Option A: Vercel (Recommended)

1. **Install Vercel CLI or Connect Git Repository**:
   ```bash
   npm i -g vercel
   vercel
   ```
2. **Build Settings**:
   - **Framework Preset**: `Vite`
   - **Root Directory**: `./`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
3. **SPA Routing Configuration (`vercel.json`)**:
   ```json
   {
     "rewrites": [
       { "source": "/(.*)", "destination": "/index.html" }
     ],
     "headers": [
       {
         "source": "/(.*)",
         "headers": [
           { "key": "X-Frame-Options", "value": "DENY" },
           { "key": "X-Content-Type-Options", "value": "nosniff" },
           { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
         ]
       }
     ]
   }
   ```

---

### Option B: Netlify

1. **Connect via Netlify CLI or Web UI**:
   ```bash
   npm i -g netlify-cli
   netlify init
   ```
2. **Configuration (`netlify.toml`)**:
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
       Referrer-Policy = "strict-origin-when-cross-origin"
       Content-Security-Policy = "default-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self' data:; img-src 'self' data: https:;"
   ```

---

### Option C: Cloudflare Pages

1. **Project Setup**:
   - Project Name: `career-pilot`
   - Production branch: `main`
   - Framework preset: `Vite`
   - Build command: `npm run build`
   - Build output directory: `dist`
2. **SPA Fallback Rule**:
   Create a `public/_redirects` file:
   ```
   /*    /index.html   200
   ```

---

### Option D: Docker & Nginx Container

For on-premise or cloud virtual machines (AWS EC2, GCP Compute Engine, Kubernetes), use the following multi-stage `Dockerfile`:

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx Alpine
FROM nginx:alpine-slim
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
CMD ["nginx", "-g", "daemon off;"]
```

**Accompanying `nginx.conf`**:
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
    gzip_min_length 1000;

    # Security Headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # SPA Routing Fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Static Asset Caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }
}
```

---

## 4. Security, Privacy & Compliance Posture

1. **Zero Secret Exposure**:
   - The frontend codebase contains zero server secrets, database connection strings, or third-party API credentials.
   - All network traffic is strictly static asset fetching.
2. **Data Sovereignty**:
   - User answers and generated pathways are kept in client-side HTML5 LocalStorage (`career_pilot_profile_v1`).
   - The user can export their complete profile as JSON (`career-pilot-profile-*.json`) or trigger a permanent purge ("Confirm Profile Zeroization") at any time.
3. **Content Security Policy (CSP)**:
   - `default-src 'self'`: Prevents arbitrary script injection.
   - `object-src 'none'`: Blocks Flash/Java plugins.
   - `script-src 'self'`: Allows only hashed and bundled scripts.
   - `img-src 'self' data: blob:`: Supports inline SVG icons and data URIs.

---

## 5. Maintenance, Backup & Recovery

- **Backup**:
  Because data is local-first, users back up their sessions via the in-app **[Export JSON]** control.
- **Recovery**:
  To restore a prior session on any browser or device, users click **[Import JSON]** and upload their exported archive. Schema validation sanitizes payloads and prevents prototype pollution.
- **Zeroization**:
  The **[Reset]** button in the top telemetry ribbon purges local browser storage and reinitializes default baseline states cleanly.
