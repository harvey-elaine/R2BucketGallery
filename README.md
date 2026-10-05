# R2 Bucket Manager

A single-page browser tool for listing, previewing, and selectively deleting objects in a Cloudflare R2 bucket. No servers, no build step, no dependencies to install — the whole app is one `index.html` that talks directly to R2's S3-compatible API using `aws4fetch` (loaded from a CDN) for SigV4 signing.

**Using this as part of a free Gyazo replacement?** See [`SHAREX-SETUP.md`](./SHAREX-SETUP.md) for the full ShareX + R2 capture-and-upload pipeline this tool is meant to complement — this tool handles browsing and cleanup once screenshots start piling up in the bucket. This is a functional replacement for Gyazo or similar screenshot sharing tools.

## What it does

- Lists objects newest first by R2 last-modified date, with a prefix filter. Fetches the complete matching listing before sorting, then displays 100 at a time with a Load-more button.
- Renders inline thumbnails for image content types (`png`, `jpg`, `gif`, `webp`, `svg`, `avif`, `bmp`, `tiff`, `heic`, `heif`).
- Click any card to select; batch-delete with a confirmation modal that shows the exact key list.
- Credentials live in the browser's `localStorage` for this origin — nothing is sent anywhere except directly to R2.

## Setup (one-time)

### 1. Create a scoped R2 API token

Cloudflare dashboard → **R2** → **Manage R2 API Tokens** → **Create API Token**.

- **Permissions:** Object Read & Write
- **Specify bucket:** the one you want to manage
- **TTL:** open-ended is fine for personal use; rotate on schedule if you prefer

The token page gives you three values — copy them into the app's Config panel:

- **Access Key ID** → *Access Key ID*
- **Secret Access Key** → *Secret Access Key*
- The endpoint's `<accountid>` in `https://<accountid>.r2.cloudflarestorage.com` → *Account ID*

Also enter the bucket name.

### 2. Configure CORS on the bucket

Cloudflare dashboard → **R2** → your bucket → **Settings** → **CORS Policy**. Paste (replace the origin):

```json
[
  {
    "AllowedOrigins": ["https://your-site.netlify.app"],
    "AllowedMethods": ["GET", "DELETE"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

If you also want to run this locally (opening the file directly), add `"http://localhost:*"` and consider serving via a simple local server rather than `file://`.

### 3. (Optional) Public URL prefix

If the bucket is fronted by a custom domain (e.g. `cdn.example.com`), enter that as **Public URL prefix**. Image previews will load directly from it, which is faster and avoids per-image signing. Without it, previews use short-lived presigned URLs — slower but works for private buckets.

## Deploy to Netlify

From this directory:

```powershell
netlify deploy --dir=. --site=<your-site-id-or-name>
netlify deploy --dir=. --site=<your-site-id-or-name> --prod
```

The first command deploys to a preview URL for a smoke test; the second promotes to the site's production URL. **Always pass `--site` explicitly** — the CLI's globally-linked project is unrelated and can quietly deploy to the wrong site.

## Security notes

- The credentials never leave the browser. This tool talks to R2 directly; there is no backend.
- Scope the token to a single bucket. Read & Write is the minimum for list + delete; do not use an account-wide token.
- `localStorage` is per-origin — anything else deployed to the same Netlify site would be able to read it. Keep this site dedicated to this tool.
- If you use a shared machine, click **Clear stored credentials** when done.

## Known limits (first cut)

- No rename, move, upload, or per-object download — this is a browse-and-delete tool.
- Refresh fetches all matching object metadata in pages of up to 1,000 to ensure global date ordering. Large buckets may take longer to refresh; thumbnails are displayed 100 at a time.
- Order is fixed to newest first by last-modified date (upload or replacement time), with filename as a tie-breaker. Prefix filter narrows the listing.
- Deletes are individual `DELETE` requests in parallel, not `POST ?delete` batch. Fine for tens of objects; if you want to delete thousands at once, this needs the batch API.
