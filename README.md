# Sunrise FinOps — Azure Cost Intelligence

A static single-page web app that reads Azure Cost Management **actual-cost** CSV exports **directly from Blob Storage**, auto-crawls every subscription folder, and gives you filtering + charts. No backend, no Excel.

## How it reads the storage account

The app calls the Azure Blob REST API straight from the browser:
- **List**: `GET {account}/{container}?restype=container&comp=list&delimiter=/&prefix=...`
- **Download**: `GET {account}/{container}/{blobName}`

It walks the folder tree the same way your exports are laid out:

```
finops/                                  ← container
  management/                            ← subscription folder
    management-actual-cost/              ← matches "actual-cost" suffix
      20250501-20250531/                 ← period folder (latest auto-picked)
        290a1306-c001-.../               ← GUID run folder
          part_0_0001.csv                ← the CSV it downloads
  dev/  prod/  uat/  ...                 ← every other subscription, same pattern
```

**Auto-crawl** picks the **latest period** in each `*-actual-cost` folder, finds the GUID run folder(s), downloads every `part_*.csv`, and merges them. Each file is tagged with its subscription + period.

## ⚠️ Two things MUST be set up on the storage account

A browser cannot bypass these — they are Azure security rules, not app limitations.

### 1. SAS token (Read + List)
The `finops` container is private, so the app needs a SAS token.

**Portal → Storage account → Shared access signature:**
- Allowed services: **Blob**
- Allowed resource types: **Container** + **Object**
- Allowed permissions: **Read** + **List**
- Set an expiry, click **Generate SAS and connection string**
- Copy the **SAS token** (the `?sv=...&sig=...` string) and paste it into the app

> Prefer a narrower **per-container** or stored-access-policy SAS in production. Anyone with the token can read the container until it expires.

### 2. CORS (or the browser blocks every request)

**Portal → Storage account → Settings → Resource sharing (CORS) → Blob service:**

| Setting | Value |
|---|---|
| Allowed origins | `https://<your-app-name>.azurewebsites.net` (or `*` for testing) |
| Allowed methods | `GET`, `OPTIONS` |
| Allowed headers | `*` |
| Exposed headers | `*` |
| Max age | `3600` |

Without this, you'll see "Failed to fetch" / CORS errors in the crawl log.

## Using the app

1. Open the deployed URL
2. Confirm **Storage Account URL** and **Container** (pre-filled for `sunrisefinopscostmgmt` / `finops`)
3. Paste your **SAS token**
4. Click **⚡ Auto-Crawl All Subscriptions** — watch the live log
5. Filter and explore

To pin a different cost type, change **"Cost folder suffix to match"** (default `actual-cost`; the app skips amortized/focus folders unless you change this).

## Features

- **Auto-crawl** every subscription's latest actual-cost export
- **Multi-select** filters with checkboxes + search: subscriptions, resource groups, service family, location, meter category
- **Tag filters** with selectable multiple values: **Department**, **Environment**, **Application**
- **Combined total across multiple selected subscriptions** — shown in the blue KPI and the "Cost by Subscription" banner with a grand total
- **Full dollar amounts with cents everywhere** (e.g. `$2,979.72`, never `$2.8K`) — only chart axis ticks abbreviate to stay readable
- Charts: service family donut, location bar, daily trend, top 10 resource groups, top 10 meter categories
- Resources tab: group by resource / RG / service / meter / location / department / subscription
- Detail rows: paginated, searchable, sortable, with environment + department badges
- **Manual upload** fallback (drag & drop CSVs) if you'd rather not use a SAS token

## Files

```
finops-app/
├── index.html   ← entire app
└── web.config   ← Azure App Service (Windows/IIS) static hosting
```

## Deploy to Azure App Service

```bash
cd finops-app
zip -r deploy.zip index.html web.config
az webapp deployment source config-zip \
  --resource-group <your-rg> \
  --name <your-app-service-name> \
  --src deploy.zip
```

Or drag both files into **Kudu → site/wwwroot**, or use the VS Code Azure App Service extension.

## Security notes

- All processing is client-side; cost data never leaves the browser except the direct call to your own storage account.
- The SAS token lives only in the browser field — it is **not** saved or transmitted anywhere else. For a no-token-in-browser setup, a small backend holding the access key is the alternative (not included in this static build).
