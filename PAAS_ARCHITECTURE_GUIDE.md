# Modern PaaS & BaaS Architecture Guide
**Production Reference for Multi-Tenant SaaS Deployments**

*Documenting the architecture, setup workflows, operational nuances, and comparative evaluation of modern cloud platforms (Render, Vercel, Supabase, GitHub Actions, Firebase, and Cloudflare).*

---

## 1. Executive Summary & Architecture Overview

This project implements a **Presentation-First, Edge-Distributed Micro-SaaS**. Rather than managing virtual machines, container orchestrators (like Kubernetes), or custom web servers, the entire infrastructure runs on **Serverless Edge CDNs** and **Backend-as-a-Service (BaaS)** providers.

### The Production Topology
```
[ User Browser / Mobile Device ]
              │
              ├──► DNS & Edge SSL: Cloudflare Proxy
              │
              ├──► Frontend Edge Host: Render / Vercel
              │       └── Serves single lightweight template (index.html, CSS, JS)
              │
              └──► Database & API: Supabase (PostgreSQL + PostgREST)
                      └── Dynamic REST endpoint queries (/rest/v1/cards?slug=eq...)
                              ▲
                              │
[ GitHub Actions Cron ] ──────┘ (Sends 48-hour authenticated keep-alive heartbeat)
```

---

## 2. Platform Deep-Dives & Setup Runbooks

### A. Render (Frontend Edge Hosting)

#### 1. Core Mechanics
Render is a cloud application platform providing two distinct service models:
* **Web Services (Containers/Node/Python):** Provision dedicated execution environments. On the Free Tier, these spin down (sleep) after **15 minutes** of inactivity, causing a 30–50s cold start on the next request.
* **Static Sites (HTML/CSS/JS):** Distributed across global Anycast CDN networks. **Static sites never sleep, never spin down, and have zero cold-start delay.**

#### 2. How Our Setup Works
* **Repository Linked:** `ashish-marsani/virtual-card` on GitHub (`main` branch).
* **Build Command:** *(None required for pure HTML/CSS/JS)*.
* **Publish Directory:** `./` (root).
* **Custom Domain:** `ashishmarsani.com.np` pointed via Cloudflare CNAME to Render's custom host (`rndr.id`).
* **Continuous Deployment (CD):** Every `git push origin main` triggers an automatic zero-downtime rebuild and CDN cache invalidation.

---

### B. Vercel (Edge Routing & Serverless Deployment)

#### 1. Core Mechanics
Vercel is built for frontend and edge runtime optimization. It uses a proprietary Edge Network with native route rewrite capabilities.

#### 2. How Our Setup Works
In our repository, [`vercel.json`](./vercel.json) defines single-template dynamic routing:
```json
{
  "cleanUrls": true,
  "rewrites": [
    {
      "source": "/:slug",
      "destination": "/index.html"
    }
  ]
}
```

* **The Problem It Solves:** If a customer visits `yourdomain.com/elena` or `yourdomain.com/demo`, traditional web servers look for a physical folder named `/elena/index.html` and return `404 Not Found`.
* **The Vercel Solution:** The rewrite rule maps any URL pattern `/:slug` back to the single root `index.html`. 
* **Client-Side Hydration:** The browser JavaScript parses the URL path or query parameter (`?card=...`), fetches the corresponding card from Supabase, and morphs the DOM on the fly.

---

### C. Supabase (PostgreSQL Backend-as-a-Service)

#### 1. Core Mechanics
Supabase is an open-source Firebase alternative built entirely on **native PostgreSQL 15+**. Every project includes:
* **PostgreSQL Engine:** Full relational SQL, indexes, schemas, and triggers.
* **PostgREST:** High-performance proxy that automatically exposes your database tables as secure RESTful APIs.
* **Row-Level Security (RLS):** Granular access control policies enforced directly at the Postgres kernel level.
* **GoTrue Auth & S3-Compatible Storage:** Built-in identity and object buckets.

#### 2. Security Configuration: Row-Level Security (RLS)
Public web applications must expose API keys to client browsers. Supabase enables safe public access via RLS:
```sql
-- 1. Enable RLS on the table
ALTER TABLE public.cards ENABLE ROW LEVEL SECURITY;

-- 2. Allow unrestricted read access ONLY for published cards
CREATE POLICY "Allow public read access on published cards" 
ON public.cards 
FOR SELECT 
USING (is_published = true);
```
* **Security Result:** The public `anon` key can only execute `SELECT` queries on rows where `is_published = true`. Any attempt to execute `INSERT`, `UPDATE`, or `DELETE` using the public key is rejected with Postgres error `42501 (insufficient privilege)`.
* **Admin Operations:** New customers are added directly inside the Supabase Table Editor or SQL Editor, bypassing public exposure.

#### 3. Free Tier Inactivity Policy
* If a Supabase Free project receives **zero API calls and no dashboard activity for 7 consecutive days**, it enters a **"Paused"** state.
* Once paused, API requests fail until an administrator manually logs into the dashboard and clicks "Restore Project" (taking ~1–2 minutes).

---

### D. GitHub Actions (Automated Keep-Alive Engine)

To eliminate the risk of Supabase pausing due to inactivity, we built an automated CI/CD workflow in [`.github/workflows/keep-alive.yml`](./.github/workflows/keep-alive.yml):

```yaml
name: Keep Supabase Awake

on:
  schedule:
    # Executes every 48 hours at 04:00 UTC
    - cron: '0 4 */2 * *'
  workflow_dispatch: # Allows manual trigger from GitHub UI

jobs:
  ping-supabase:
    runs-on: ubuntu-latest
    steps:
      - name: Send API Heartbeat to Supabase
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "apikey: ${{ secrets.SUPABASE_ANON_KEY }}" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_ANON_KEY }}" \
            "https://frvjnvfhuiiwlqshlwnv.supabase.co/rest/v1/cards?select=id&limit=1")
          echo "Supabase Response HTTP Code: $STATUS"
          if [ "$STATUS" -ne 200 ]; then exit 1; fi
```

* **How It Works:** Every 48 hours, a GitHub runner boots up, queries a single lightweight indexed row (`select=id&limit=1`), and exits.
* **Resource Consumption:** Takes ~3 seconds of execution time out of GitHub's free 2,000 minutes/month allowance ($0 cost).
* **Result:** Supabase's 7-day inactivity timer is permanently reset.

---

## 3. Platform Comparison Matrix (For Future Architecture Learning)

| Category | Render | Vercel | Supabase | Google Firebase | Cloudflare Pages & Workers |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Platform Type** | PaaS (Static + Web + DB) | Frontend & Edge PaaS | BaaS (PostgreSQL) | BaaS (NoSQL / Firestore) | Edge Compute & Storage |
| **Database Engine** | Managed PostgreSQL (Paid) | None (Connects to DBs) | Full PostgreSQL 15+ | Cloud Firestore (NoSQL) | Cloudflare D1 (SQLite) |
| **API Layer** | Custom backend code | Serverless Functions | Auto-generated PostgREST | Firebase Client SDKs | Workers (JS / Rust) |
| **Sleep / Inactivity** | Web: 15 mins<br>Static: Never | Functions: Cold starts<br>Static: Never | Pauses after 7 days | Never pauses (Pay-as-you-go) | Never pauses (Always edge) |
| **Dynamic Routing** | Redirects / Rewrites tab | `vercel.json` rewrites | N/A | `firebase.json` rewrites | `_routes.json` / Worker |
| **Best Used For** | Microservices, Docker, Static | Next.js, React, Edge HTML | Relational data, Auth, APIs | Mobile apps, Realtime NoSQL | Global ultra-low latency edge |

---

## 4. Comparing Supabase vs. Firebase (Next Evolution)

As you expand into **Google Firebase**, here is how it compares directly with your current **Supabase** stack:

| Dimension | Supabase (What you have now) | Google Firebase (Future study) |
| :--- | :--- | :--- |
| **Data Model** | **Relational (SQL)**. Tables, foreign keys, complex joins, schemas. | **Document-based (NoSQL)**. Collections, documents, and sub-collections. |
| **Querying** | Full SQL (`JOIN`, `GROUP BY`, aggregations, window functions). | Limited querying. Filtering requires compound indexes; no native joins. |
| **Security** | Native PostgreSQL **Row-Level Security (RLS)** in SQL. | Proprietary **Firebase Security Rules** syntax in JSON/DSL. |
| **Vendor Lock-in** | Very Low: Standard Postgres can be exported and run on AWS, Docker, or bare metal. | High: Proprietary Google APIs tightly coupled to GCP infrastructure. |
| **Offline Sync** | Basic caching. | Best-in-class mobile offline sync for Android/iOS apps. |
| **Pricing Model** | Predictable monthly compute tiers. | Usage-based: Charges per document read, write, and delete. |

---

## 5. Maintenance Checklist

1. **Deploying Code Updates:** Push commits directly to `main` via Git (`git push origin main`). Render deploys automatically within 30 seconds.
2. **Adding Customer Cards:** Open Supabase Dashboard → `cards` table → click `Insert row` → populate fields → Save.
3. **Updating Assets:** Store optimized images (e.g. `ashish_profile.jpg`) directly in the Git repository for instant CDN delivery.
4. **Verifying Keep-Alive:** Check [GitHub Actions](https://github.com/ashish-marsani/virtual-card/actions) to review the 48-hour automated heartbeat log.
