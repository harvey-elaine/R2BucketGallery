# ShareX + Cloudflare R2 — a free-ish Gyazo replacement

Gyazo's outage in September 2026 motivated this: a Windows-based hotkey screenshot tool that
uploads instantly and puts a shareable link on your clipboard, without a subscription, and in your control. This is
the pipeline, built on the free ShareX capture tool and Cloudflare R2 for storage. R2 has zero
egress fees, so serving images out costs nothing.

**This repo's main tool (`index.html`) is the companion to this setup**: R2 has no easy
gallery, so once screenshots are piling up in the bucket, use the bucket manager to browse
thumbnails, get the public links to any of them, and delete old ones.

**End result:** press a hotkey, drag a region, and a direct image link (`https://img.yourdomain.com/abc123.png`)
is on your clipboard, with a local copy saved automatically.

## Is it actually free?

Mostly, not entirely:

- **R2 storage/requests:** free up to 10 GB stored, 1M writes/month, 10M reads/month, and egress
  (serving the images) is *always* free regardless of tier — that's R2's whole pitch versus S3.
  Ordinary screenshot volume (a few hundred a month) takes a couple of years to fill 10 GB, and
  even if it does, storage beyond that is fractions of a cent per GB/month.
- **A domain you control:** R2's custom-domain feature requires a domain whose DNS is on
  Cloudflare. If you don't already have a spare one, a `.com` costs roughly $10–11/year at
  Cloudflare Registrar (registered at cost, no markup). This is the one real recurring cost.
- **ShareX:** free, open source.

So: ~$10–11/year if you need to buy a domain, ~$0 if you already have a spare one you can point at
Cloudflare.

⚠️ **Use a domain — or a subdomain slice of one — that isn't carrying email or a live site.**
Pointing a domain's DNS at Cloudflare moves the *whole* zone there. If Cloudflare's automatic scan
misses an MX or SPF/DKIM record when importing, mail on that domain can silently start failing. A
spare/parked domain, or a freshly bought one used only for this, avoids the risk entirely.

## What you need

- A [Cloudflare account](https://dash.cloudflare.com/sign-up) (free).
- A domain you're willing to delegate to Cloudflare's nameservers (buy one there, or use a spare).
- [ShareX](https://getsharex.com/) installed (Windows only).

## Setup

### 1. Get a domain onto Cloudflare

- **Simplest:** buy one directly through Cloudflare Registrar (Dashboard → **Domain Registration**
  → **Register Domains**). It's born on Cloudflare's DNS already, so there's no migration risk.
  Turn on **auto-renew** — if the domain ever lapses, every link you've ever shared breaks at once.
- **Alternative:** point an existing unused domain's nameservers at Cloudflare. Cloudflare scans
  and imports existing DNS records when you add the domain — verify the scan caught everything
  (especially `MX` and any `TXT` records for SPF/DKIM) *before* changing nameservers at your
  registrar, since that's the point where a missed record starts to bite.

### 2. Create the R2 bucket

1. Cloudflare dashboard → **R2 Object Storage** → enable/subscribe (a payment method is required
   even to stay on the free tier, but you won't be charged within the free allowance).
2. **Create bucket**. Any name (e.g. `screenshots`). Leave location Automatic, storage class
   Standard.
3. Open the bucket → **Settings** → **Public access** → **Custom Domains** → **Connect Domain**.
   Enter a subdomain, e.g. `img.yourdomain.com`. Cloudflare creates the DNS record for you — wait
   for the status to read **Active** (can take a few minutes for the certificate).

### 3. Create a scoped API token

1. R2 Object Storage overview → **Manage API tokens** → **Create API token**.
2. Permission: **Object Read & Write**.
3. Scope it to the one bucket you just created — not account-wide.
4. Copy the **Access Key ID**, **Secret Access Key**, and the endpoint's account ID
   (`https://<account-id>.r2.cloudflarestorage.com`) somewhere safe. **The secret is shown once**;
   if you lose it, delete the token and make a new one.

### 4. Install and configure ShareX

1. Install: `winget install --id ShareX.ShareX` (or the installer from
   [getsharex.com](https://getsharex.com/)).
2. **Destinations → Destination settings → Amazon S3**, and fill in:
   - Access Key ID / Secret Access Key — from step 3
   - Endpoint — `<account-id>.r2.cloudflarestorage.com` (host only, no scheme, no bucket)
   - Region — `auto`
   - Bucket name — the one from step 2
   - Custom domain — `img.yourdomain.com`, and enable "use custom domain for URLs"
   - Leave the AWS-regions dropdown alone if ShareX shows one — picking a region there can
     overwrite the endpoint you just set.
3. **Destinations → Image uploader → Amazon S3**, and **Destinations → File uploader → Amazon
   S3** as well (some ShareX versions route through the file uploader for non-image-specific
   tasks; setting only one leaves the other pointed at its default, which errors).
4. **After capture tasks**, tick both **Save image to file** and **Upload image to host** — the
   first is what gives you a local copy on disk alongside the upload (see the note under "Optional:
   automatic expiry" below).
5. Set your preferred hotkeys under **Hotkey settings** (ShareX ships with region/full-screen/
   window capture and video/GIF recording hotkeys out of the box).

### 5. Verify

1. Capture a region. Confirm a URL lands on the clipboard using *your* domain, not
   `r2.cloudflarestorage.com`.
2. Paste the URL into a browser — the image should load directly, not a download prompt or an
   XML error.
3. Paste it into Discord/Slack/wherever and confirm it renders inline as an image, not a bare
   link — some hosts that look fine in a browser fail this check.

### Browsing and cleaning up the bucket

R2 has no built-in gallery — the Cloudflare dashboard lists objects by name and size with no
thumbnails, so once screenshots start piling up there's no easy way to review or trim them from
there. This repo's main tool (see the [top-level README](./README.md)) is a single-page app for
exactly that: browse thumbnails, click to select, batch-delete. It has no build step, so any simple
static host works — the README's instructions use Netlify as the example, but GitHub Pages,
Cloudflare Pages, or opening the file locally all work too.

### Optional: automatic expiry

R2 buckets support lifecycle rules (Bucket → **Settings** → **Object lifecycle rules**) to
auto-delete objects after N days — useful if you don't want screenshots to accumulate forever and
don't need every one kept long-term. Because step 4 above also saves a local copy on capture, an
expiry rule only removes the *cloud* copy and its public link — nothing is lost, so there's no
separate backup to maintain. Not required either way; the free tier's headroom is large enough that
most people won't need this for years.

## Troubleshooting notes

- **Windows only saves ShareX's config to `Documents` if that folder is redirected (e.g. to
  OneDrive)** — worth moving ShareX's personal folder to a local path if you'd rather its config
  and credentials not sync to cloud storage. **Driving this change through ShareX's own settings UI
  is safer than hand-editing its JSON config** — a malformed hand edit can silently revert to
  defaults and lose the folder-move setting.
- A brand-new Cloudflare zone can take up to ~30 minutes to start resolving DNS, even after the
  custom domain shows **Active**. If the image link 404s or times out right after setup, wait and
  retry before assuming something's misconfigured.
- ShareX only writes its config to disk on exit — if you're inspecting its JSON files while it's
  running, you'll see stale values.

## Links

- [ShareX](https://getsharex.com/) — capture tool
- [Cloudflare R2 docs](https://developers.cloudflare.com/r2/)
- [Cloudflare Registrar](https://developers.cloudflare.com/registrar/) — at-cost domain registration
