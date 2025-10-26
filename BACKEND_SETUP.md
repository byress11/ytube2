# Backend API Setup Guide

## ⚠️ CRITICAL SECURITY ISSUES

### Current Problems:
1. **API Key Exposed**: Your YouTube API key is visible in client-side code
2. **No Video Download**: YouTube ToS forbids downloading via API
3. **Quota Limits**: Direct client API calls can exhaust your quota quickly

---

## 🔒 Secure Backend Solution

### Step 1: Create Backend Server (Node.js + Express)

```bash
mkdir vidflow-backend
cd vidflow-backend
npm init -y
npm install express cors dotenv
```

### Step 2: Create `.env` File

```env
YOUTUBE_API_KEY=AIzaSyCC0F6nmwpMoxt4bghLAaxUwo2V4QwxuLI
PORT=3001
ALLOWED_ORIGINS=http://localhost:3000
```

**⚠️ NEVER commit `.env` to git!**

### Step 3: Create `server.js`

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const fetch = require('node-fetch');

const app = express();
const PORT = process.env.PORT || 3001;

// CORS configuration
app.use(cors({
    origin: process.env.ALLOWED_ORIGINS.split(',')
}));

app.use(express.json());

// Proxy YouTube API requests
app.get('/api/youtube/:endpoint', async (req, res) => {
    try {
        const { endpoint } = req.params;
        const queryParams = new URLSearchParams(req.query);
        queryParams.append('key', process.env.YOUTUBE_API_KEY);

        const url = `https://www.googleapis.com/youtube/v3/${endpoint}?${queryParams}`;
        const response = await fetch(url);
        const data = await response.json();

        res.json(data);
    } catch (error) {
        console.error('API Error:', error);
        res.status(500).json({ error: 'Failed to fetch from YouTube API' });
    }
});

// Rate limiting
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);

app.listen(PORT, () => {
    console.log(`🚀 Backend server running on port ${PORT}`);
});
```

### Step 4: Update Frontend

Replace direct API calls in `index.html`:

```javascript
// OLD - Insecure
const API_KEY = 'AIzaSyCC0F6nmwpMoxt4bghLAaxUwo2V4QwxuLI';
const API_BASE_URL = 'https://www.googleapis.com/youtube/v3';

// NEW - Secure
const API_BASE_URL = 'http://localhost:3001/api/youtube';

async function fetchYouTubeAPI(endpoint, params) {
    const url = new URL(`${API_BASE_URL}/${endpoint}`);
    Object.entries(params).forEach(([key, value]) => {
        url.searchParams.append(key, value);
    });
    // No API key needed - backend handles it

    const response = await fetchWithRetry(url.toString());
    return await response.json();
}
```

---

## 🚫 Video Download - Legal Alternatives

### ❌ What You CANNOT Do:
- Download YouTube videos via API (violates ToS)
- Use `ytdl-core` or `yt-dlp` for public access
- Distribute downloaded content

### ✅ What You CAN Do:

#### Option 1: User Upload Feature
Allow users to upload their own videos:

```javascript
<input type="file" accept="video/*,audio/*" onchange="handleUserUpload(event)">

async function handleUserUpload(event) {
    const file = event.target.files[0];
    const videoInfo = {
        id: 'local_' + Date.now(),
        title: file.name,
        channel: 'My Videos',
        thumbnail: await generateThumbnail(file)
    };
    await saveToIndexedDB(videoInfo, file, 'video');
}
```

#### Option 2: YouTube Offline Feature (Premium Users)
Inform users about YouTube Premium's offline feature:

```javascript
function showOfflineInfo() {
    alert('YouTube Premium kullanıcıları, YouTube uygulamasında videoları çevrimdışı izleyebilir.');
}
```

#### Option 3: Bookmark/Save Feature (Current)
Keep the current "save" feature that only bookmarks videos:

```javascript
// Already implemented - just saves metadata
function toggleVideoSave(video) {
    // Saves only: id, title, thumbnail, channel
    // Does NOT download actual video
}
```

---

## 🛡️ Security Best Practices

### 1. Regenerate Your API Key
```bash
# Go to: https://console.cloud.google.com/apis/credentials
# Delete current key
# Create new key with restrictions:
# - HTTP referrers: your-domain.com
# - API restrictions: YouTube Data API v3 only
```

### 2. Add to `.gitignore`
```bash
echo ".env" >> .gitignore
echo "BACKEND_SETUP.md" >> .gitignore
git rm --cached index.html  # If API key already committed
```

### 3. Use Environment Variables
```javascript
// Never do this:
const API_KEY = 'AIzaSyCC0F6nmwpMoxt4bghLAaxUwo2V4QwxuLI';

// Always do this (backend only):
const API_KEY = process.env.YOUTUBE_API_KEY;
```

---

## 📊 Deployment Considerations

### For Production:

1. **Backend Hosting**: Vercel, Railway, Heroku
2. **Environment Variables**: Set in hosting platform
3. **HTTPS**: Required for production
4. **CORS**: Restrict to your domain only
5. **Rate Limiting**: Prevent quota exhaustion

### Example: Vercel Deployment

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy backend
cd vidflow-backend
vercel --prod

# Set environment variables
vercel env add YOUTUBE_API_KEY
```

---

## ⚖️ Legal Disclaimer

This application:
- Uses YouTube Embedded Player (legal)
- Does NOT download videos (compliance with ToS)
- IndexedDB feature is for demo purposes only
- Users must respect copyright and ToS

For actual offline viewing, users should:
1. Subscribe to YouTube Premium
2. Use official YouTube offline feature
3. Respect content creators' rights

---

## 📞 Support

For issues or questions:
1. Check YouTube API documentation
2. Review Google Cloud Console quotas
3. Ensure ToS compliance

**Remember**: Never expose API keys in client-side code!
