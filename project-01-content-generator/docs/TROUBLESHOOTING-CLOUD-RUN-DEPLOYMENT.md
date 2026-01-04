# Troubleshooting: Cloud Run Deployment - WebSocket Mixed Content Issue

**Date**: 2026-01-04
**Project**: AI Content Generator (Multi-Agent System)
**Environment**: Google Cloud Run
**Issue**: Empty/blank console page after deployment to Cloud Run

---

## Table of Contents
1. [Problem Summary](#problem-summary)
2. [Symptoms](#symptoms)
3. [Debugging Process](#debugging-process)
4. [Root Cause](#root-cause)
5. [Solution](#solution)
6. [Key Learnings](#key-learnings)
7. [Reference Commands](#reference-commands)

---

## Problem Summary

After successfully deploying the Content Generator application to Google Cloud Run:
- ✅ Homepage loaded correctly with form
- ✅ Backend API responded successfully
- ❌ Console page (`/console/:jobId`) showed only blue background with no content
- ❌ No headers, footers, or React components visible

**Critical detail**: Application worked perfectly in local Docker Desktop environment but failed in Cloud Run production deployment.

---

## Symptoms

### User-Reported Symptoms
1. **Homepage**: Working correctly
   - Form rendered
   - "AI Content Generator" header visible
   - Footer visible
   - Submit button functional

2. **Console Page**: Completely broken
   - URL: `https://content-generator-frontend-969486604732.us-central1.run.app/console/[job-id]`
   - Only blue background color visible
   - No headers, footers, or content
   - No JavaScript errors initially visible (misleading!)

3. **Network Behavior**:
   - Form submission succeeded (POST to `/api/generate` returned 200)
   - Redirect to console page occurred
   - Page loaded but rendered nothing

---

## Debugging Process

### Phase 1: Initial Investigation

#### Step 1: Verify Backend Functionality
**Claude Code Action**:
```bash
# Check if backend is running and job exists
curl -s "https://content-generator-backend-969486604732.us-central1.run.app/api/status/e12a2d59-db6f-49b9-b4be-2bca197732de" | jq .
```

**Result**: ✅ Backend working correctly
```json
{
  "job_id": "e12a2d59-db6f-49b9-b4be-2bca197732de",
  "status": "completed",
  "created_at": "2026-01-04T05:00:17.935067",
  "result": "# Health Tech Innovations: Transforming Healthcare in 2023\n\n..."
}
```

**Conclusion**: Backend and API working correctly. Issue is frontend-only.

---

#### Step 2: Check Frontend Proxy
**Claude Code Action**:
```bash
# Test if frontend proxy forwards requests correctly
curl -s "https://content-generator-frontend-969486604732.us-central1.run.app/api/status" | jq .
```

**Result**: ✅ Proxy working
```json
{
  "openai_connected": true,
  "demo_mode": false,
  "timestamp": "2026-01-04T05:04:15.321420"
}
```

**Conclusion**: Frontend proxy correctly forwarding API requests to backend.

---

#### Step 3: Check Cloud Run Logs

**Claude Code Action**:
```bash
# Check backend logs for request processing
gcloud run services logs read content-generator-backend --region us-central1 --limit 50
```

**Result**: Backend processing jobs successfully
```
2026-01-04 04:54:50 🚀 Crew: crew
2026-01-04 04:54:50 └── 📋 Task: research_task (ID: 5ec9b6e4-db71-443a-a948-e5e4a4ab8603)
2026-01-04 04:54:50     Status: ✅ Completed
```

**Claude Code Action**:
```bash
# Check frontend logs for errors
gcloud run services logs read content-generator-frontend --region us-central1 --limit 50
```

**Result**: Frontend serving files correctly
```
2026-01-04 04:54:21 POST 200 https://content-generator-frontend-.../api/generate
2026-01-04 04:54:22 GET 200 https://content-generator-frontend-.../api/status/9a85e572...
```

**Conclusion**: No obvious errors in Cloud Run logs. Both services responding with 200 status codes.

---

#### Step 4: Check for Warnings in Logs

**Claude Code Action**:
```bash
# Look for warnings or errors
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=content-generator-frontend AND severity>=WARNING" --limit=50
```

**Result**: Found 404 errors for old job IDs
```json
{
  "httpRequest": {
    "status": 404,
    "requestUrl": "https://content-generator-frontend-.../api/status/3a0fb979-1355-4d2f-a585-48bdfdd12549"
  }
}
```

**User Clarification**: "I'm trying to access `/console/3a0fb979-1355-4d2f-a585-48bdfdd12549`"

**Claude Code Response**: Old job ID doesn't exist. Created test with working job ID.

**Conclusion**: Initial confusion - user was testing with non-existent job IDs from before deployment.

---

### Phase 2: Browser Developer Tools Investigation

#### Step 5: Inspect HTML Structure

**Claude Code Request**: "Please check Developer Tools → Elements tab"

**User Response** (viewing `/console/[job-id]`):
```html
<html lang="en">
  <head>...</head>
  <body>
    <div id="root">
      <div class="App">
        <header class="App-header">
          <h1>🤖 AI Content Generator</h1>
        </header>
        <main class="App-main"></main>  <!-- EMPTY! -->
        <footer class="App-footer">
          <p>Powered by CrewAI • FastAPI • TypeScript</p>
        </footer>
      </div>
    </div>
  </body>
</html>
```

**Critical Finding**: `<main class="App-main">` was **completely empty**!

**Conclusion**: React app loading, header/footer rendering, but LiveConsole component not rendering anything inside `<main>`.

---

#### Step 6: Check Network Tab

**Claude Code Request**: "Check Network tab for API calls and WebSocket connections"

**User Response**:
```
main.99f8e156.js    304    script
main.408845a6.css   304    stylesheet
9a85e572...         200    fetch    LiveConsole.tsx:132
status              200    fetch    App.tsx:46
```

**Observations**:
- ✅ JavaScript and CSS loading successfully
- ✅ API calls from `LiveConsole.tsx:132` executing (component IS running!)
- ❓ No WebSocket connections visible

**Conclusion**: LiveConsole component mounting and making API calls, but not rendering HTML. WebSocket suspiciously absent.

---

#### Step 7: Compare Local vs Production

**User Statement**: "When I test on my local Docker Desktop, it was working."

**Critical Insight**:
- Local: Development mode (`npm start` with React dev server)
- Production: Built mode (`npm run build` served by Express)

**Claude Code Analysis**:
- Different Dockerfiles used
- Different serving mechanisms
- Production uses HTTPS (`https://`), local uses HTTP (`http://`)

---

### Phase 3: The Breakthrough - Mixed Content Error

#### Step 8: Deep Browser Console Inspection

**Claude Code Request**: "Check Console tab for ANY messages, not just errors"

**User Response** (screenshot provided):

```
Mixed content: load all resources via HTTPS to improve the security of your site

Even though the initial HTML page is loaded over a secure HTTPS connection,
some resources like images, stylesheets or scripts are being accessed over
an insecure HTTP connection. Usage of insecure resources is restricted to
strengthen the security of your entire site.

AFFECTED RESOURCES:
Name: ed3fa5e3-f704-475f-b7ea-adb54444efb0
Restriction Status: blocked
```

**🎯 ROOT CAUSE IDENTIFIED!**

The resource being blocked was the WebSocket connection to `/ws/console/[job-id]`.

---

## Root Cause

### The Problem: WebSocket Protocol Mismatch

**File**: `frontend/src/components/LiveConsole.tsx:162`

**Original Code** (broken in production):
```typescript
const wsUrl = `ws://${window.location.host}/ws/console/${jobId}`;
const ws = new WebSocket(wsUrl);
```

**Why it failed**:
1. Cloud Run serves pages over HTTPS (`https://`)
2. Code hardcoded WebSocket to use `ws://` (insecure WebSocket)
3. Browser's **Mixed Content Security Policy** blocks insecure WebSocket (`ws://`) connections from secure pages (`https://`)
4. WebSocket connection blocked → LiveConsole component couldn't establish real-time connection
5. Component failed silently (no error thrown), rendered nothing

**Why it worked locally**:
- Local development uses HTTP (`http://localhost:3000`)
- `ws://` is the correct protocol for HTTP pages
- No mixed content restriction

### Browser Security Policy

Modern browsers enforce **Mixed Content** restrictions:
- **HTTPS pages** can only load resources over HTTPS/WSS
- **HTTP resources** (including `ws://`) are blocked on HTTPS pages
- This prevents man-in-the-middle attacks

Reference: [MDN Mixed Content](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content)

---

## Solution

### Code Fix

**File**: `frontend/src/components/LiveConsole.tsx`

**Changed**:
```typescript
// Before (BROKEN in production)
const wsUrl = `ws://${window.location.host}/ws/console/${jobId}`;

// After (WORKS everywhere)
// Use wss:// for HTTPS pages, ws:// for HTTP pages
const wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
const wsUrl = `${wsProtocol}//${window.location.host}/ws/console/${jobId}`;
const ws = new WebSocket(wsUrl);
```

**Why this works**:
- Detects page protocol (`window.location.protocol`)
- Uses `wss://` (secure WebSocket) when page is HTTPS
- Uses `ws://` (insecure WebSocket) when page is HTTP
- Works in both development and production

### Deployment After Fix

```bash
cd /Users/pwu/MASProjects/pingai/MACA-Course/maca/project-01-content-generator

# Redeploy frontend with WebSocket fix
gcloud run deploy content-generator-frontend \
  --source ./frontend \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --memory 1Gi \
  --cpu 1 \
  --timeout 120 \
  --set-env-vars BACKEND_URL=https://content-generator-backend-969486604732.us-central1.run.app,NODE_ENV=production \
  --max-instances 5 \
  --min-instances 0
```

**Result**: ✅ Application working correctly in production!

---

## Key Learnings

### 1. Local vs Production Differences Matter

**Issue**: "It works on my machine" ≠ "It works in production"

**Lesson**: Always test with production-like settings:
- Use HTTPS locally (not just HTTP)
- Test production builds (`npm run build`) before deploying
- Consider using `docker-compose.prod.yml` that mirrors production

### 2. Browser Security Policies Are Strict

**Issue**: Mixed content blocked WebSocket without obvious error

**Lesson**:
- Always check browser Developer Tools → Console for security warnings
- Look for **yellow** warnings, not just red errors
- Mixed content warnings often appear collapsed - expand them!

### 3. WebSocket Protocol Must Match Page Protocol

**Rule**:
- `https://` pages → use `wss://` (secure WebSocket)
- `http://` pages → use `ws://` (insecure WebSocket)

**Best Practice**:
```javascript
// ✅ GOOD - Protocol-aware
const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
const wsUrl = `${protocol}//${window.location.host}/ws/path`;

// ❌ BAD - Hardcoded protocol
const wsUrl = `ws://${window.location.host}/ws/path`;
```

### 4. Component Mounting ≠ Component Rendering

**Issue**: Component was mounting (making API calls) but not rendering

**Lesson**:
- Check Network tab to see if component code is executing
- Check Elements tab to see if HTML is being generated
- A component can run JavaScript without rendering DOM

### 5. Cloud Run Logs Don't Show Client-Side Errors

**Issue**: Cloud Run logs showed 200 status codes (success)

**Lesson**:
- Server logs only show HTTP requests/responses
- JavaScript errors happen in the browser, not the server
- Always check **browser console**, not just server logs

### 6. Systematic Debugging Process

**Effective Approach**:
1. ✅ Verify backend is working
2. ✅ Verify frontend is serving files
3. ✅ Check server logs for errors
4. ✅ Inspect HTML in browser
5. ✅ Check Network tab for failed requests
6. ✅ Check Console tab for ALL messages (not just errors)
7. ✅ Compare local vs production environments

---

## Reference Commands

### Deployment Commands

```bash
# Initial setup
gcloud config set project swd-agent-jan2026
gcloud config set run/region us-central1

# Enable required APIs
gcloud services enable run.googleapis.com
gcloud services enable containerregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable secretmanager.googleapis.com

# Create secrets
echo -n "$OPENAI_API_KEY" | gcloud secrets create openai-api-key \
  --replication-policy="automatic" \
  --data-file=-

echo -n "$SERPER_API_KEY" | gcloud secrets create serper-api-key \
  --replication-policy="automatic" \
  --data-file=-

# Grant secret access to Cloud Run service account
gcloud secrets add-iam-policy-binding openai-api-key \
  --member="serviceAccount:969486604732-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding serper-api-key \
  --member="serviceAccount:969486604732-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Deploy backend
gcloud run deploy content-generator-backend \
  --source . \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --memory 2Gi \
  --cpu 1 \
  --timeout 300 \
  --set-secrets OPENAI_API_KEY=openai-api-key:latest,SERPER_API_KEY=serper-api-key:latest \
  --set-env-vars ENVIRONMENT=production,LLM_MODEL=gpt-4o-mini \
  --max-instances 10 \
  --min-instances 0

# Deploy frontend
gcloud run deploy content-generator-frontend \
  --source ./frontend \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --memory 1Gi \
  --cpu 1 \
  --timeout 120 \
  --set-env-vars BACKEND_URL=https://content-generator-backend-969486604732.us-central1.run.app,NODE_ENV=production \
  --max-instances 5 \
  --min-instances 0
```

### Debugging Commands

```bash
# View service details
gcloud run services describe content-generator-frontend --region us-central1

# Check environment variables
gcloud run services describe content-generator-frontend \
  --region us-central1 \
  --format="value(spec.template.spec.containers[0].env)"

# View logs (last 50 lines)
gcloud run services logs read content-generator-backend --region us-central1 --limit 50
gcloud run services logs read content-generator-frontend --region us-central1 --limit 50

# View logs with filters
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=content-generator-frontend AND severity>=WARNING" --limit=50

# Test endpoints
curl -s "https://content-generator-backend-969486604732.us-central1.run.app/" | jq .
curl -s "https://content-generator-frontend-969486604732.us-central1.run.app/api/status" | jq .

# Update service without rebuild
gcloud run services update content-generator-backend \
  --region us-central1 \
  --update-env-vars CORS_ALLOW_ORIGINS=https://content-generator-frontend-969486604732.us-central1.run.app

# Delete and recreate service
gcloud run services delete content-generator-frontend --region us-central1 --quiet
# Then redeploy with gcloud run deploy...
```

### Browser Developer Tools Checklist

**Console Tab**:
- [ ] Look for red errors
- [ ] Look for yellow warnings (especially "Mixed content")
- [ ] Check for component mount logs
- [ ] Expand collapsed warning messages

**Network Tab**:
- [ ] Check if JavaScript bundles load (200 status)
- [ ] Check if API calls succeed (200 status)
- [ ] Look for WebSocket connections (WS tab)
- [ ] Check for failed requests (red status)

**Elements Tab**:
- [ ] Verify `<div id="root">` exists
- [ ] Check if React components render HTML
- [ ] Look for empty elements that should have content
- [ ] Inspect CSS classes and styles

---

## Conclusion

**Total debugging time**: ~2 hours
**Key breakthrough**: Finding the Mixed Content warning in browser console
**Final fix**: 3 lines of code to make WebSocket protocol dynamic

**Success metrics after fix**:
- ✅ Homepage loads correctly
- ✅ Form submission works
- ✅ Console page renders with live updates
- ✅ WebSocket connection establishes (`wss://`)
- ✅ Real-time agent progress displays
- ✅ Completed content shows correctly

**Deployed URLs**:
- Frontend: https://content-generator-frontend-969486604732.us-central1.run.app/
- Backend: https://content-generator-backend-969486604732.us-central1.run.app/

---

**Document Prepared By**: Claude Code
**User**: Ping Wu
**Project**: Multi-Agent Content Generator
**Date**: 2026-01-04
**Status**: ✅ Resolved
