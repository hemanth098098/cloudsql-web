# CloudSQL Web — Oracle Fusion Query Tool

A browser-based SQL query tool for Oracle Fusion, deployable on Render via GitHub.

---

## 📁 File Structure

```
cloudsql-web/
├── index.html        ← Main web application (deploy this)
├── README.md         ← This guide
└── proxy/            ← Optional: Node.js backend for raw SQL (see below)
```

---

## 🚀 Step 1 — Push to GitHub

### If you're new to GitHub:

1. Go to https://github.com and sign in
2. Click **+** → **New repository**
3. Name it: `cloudsql-web`
4. Set to **Public** (required for free Render)
5. Click **Create repository**

### Upload your files:

**Option A — GitHub website (easiest):**
1. Open your new repo
2. Click **Add file** → **Upload files**
3. Drag `index.html` and `README.md` into the box
4. Click **Commit changes**

**Option B — Git command line:**
```bash
git init
git add index.html README.md
git commit -m "Initial CloudSQL web app"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/cloudsql-web.git
git push -u origin main
```

---

## 🌐 Step 2 — Deploy on Render

1. Go to https://render.com and sign in (use your GitHub account)
2. Click **New +** → **Static Site**
3. Click **Connect a repository** → select `cloudsql-web`
4. Fill in settings:
   - **Name:** `cloudsql-web` (or anything)
   - **Branch:** `main`
   - **Root Directory:** *(leave blank)*
   - **Build Command:** *(leave blank — no build needed)*
   - **Publish Directory:** `.`
5. Click **Create Static Site**
6. Wait ~1 minute — Render gives you a URL like:
   `https://cloudsql-web.onrender.com`

✅ Your app is now live! Open the URL and connect to Oracle Fusion.

---

## 🔗 Step 3 — Link GitHub + Render (Auto-Deploy)

Render automatically links to GitHub. Every time you push a change:
```bash
git add index.html
git commit -m "Update UI"
git push
```
→ Render detects the push and **auto-redeploys in ~30 seconds**.

To verify: In Render dashboard → your site → **Events** tab shows each deploy.

---

## 🔌 How to Connect

### Connection Type 1: Oracle Fusion REST API (built-in)
- Works for standard Oracle Fusion REST resources (Workers, Invoices, POs, etc.)
- Enter your Fusion URL, username, and password
- **Limitation:** Browser CORS policy blocks direct calls → use Option 2 for raw SQL

### Connection Type 2: Custom Backend Proxy (for raw SQL)
- Needed to run any SQL like `SELECT * FROM per_all_people_f`
- See "Backend Proxy Setup" below

---

## ⚙️ Backend Proxy Setup (for raw SQL queries)

The browser cannot directly call Oracle JDBC. A small Node.js proxy bridges the gap.

### proxy/server.js
```javascript
const express = require('express');
const cors    = require('cors');
const oracledb = require('oracledb');
const app = express();
app.use(cors()); app.use(express.json());

const DB = {
  user:             process.env.DB_USER,
  password:         process.env.DB_PASS,
  connectString:    process.env.DB_URL,   // host:port/servicename
};

app.get('/ping', (req,res) => res.json({ok:true}));

app.post('/query', async (req,res) => {
  const {sql, limit} = req.body;
  let conn;
  try {
    conn = await oracledb.getConnection(DB);
    const result = await conn.execute(sql, [], {
      maxRows: limit||100,
      outFormat: oracledb.OUT_FORMAT_ARRAY,
      fetchArraySize: 200,
    });
    res.json({
      columns: result.metaData.map(m=>m.name),
      rows:    result.rows,
    });
  } catch(e) {
    res.status(400).json({error: e.message});
  } finally {
    if(conn) await conn.close();
  }
});

app.listen(3000);
```

### proxy/package.json
```json
{
  "name": "cloudsql-proxy",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "oracledb": "^6.4.0"
  },
  "scripts": { "start": "node server.js" }
}
```

### Deploy proxy on Render:
1. Push `proxy/` folder to GitHub
2. Render → **New +** → **Web Service**
3. Connect the same repo, set **Root Directory** to `proxy`
4. **Build Command:** `npm install`
5. **Start Command:** `node server.js`
6. Add **Environment Variables:**
   - `DB_USER` → your Oracle username
   - `DB_PASS` → your Oracle password
   - `DB_URL`  → `hostname:1521/FUSIONDB`
7. Deploy → copy the URL (e.g. `https://cloudsql-proxy.onrender.com`)

### Connect in the app:
- Select **Custom Backend Proxy**
- Paste your proxy URL
- Click **Connect**

---

## 🔄 Making Updates

```bash
# Edit index.html locally, then:
git add index.html
git commit -m "Fix: column names now show correctly"
git push
# Render auto-deploys in ~30 seconds ✓
```

---

## ❓ Troubleshooting

| Problem | Solution |
|---|---|
| CORS error | Use Custom Backend Proxy connection type |
| 401 Unauthorized | Check Oracle Fusion username/password |
| 403 Forbidden | User needs REST API access in Fusion |
| Columns show wrong names | Check REST endpoint mapping in code |
| Render deploy fails | Check publish directory is set to `.` |

