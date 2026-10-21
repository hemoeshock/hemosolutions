# Deploying CertDecode (`tls-cert-errors`)

**Status: NOT DEPLOYED.** Cloudflare credentials were not available in the agent environment
(`CF_API_TOKEN` / `CF_ACCOUNT_ID` are not set), so the site is finished, validated locally and
staged for a human to publish. Nothing here is public yet.

The site is **plain static files with no build step**: `index.html`, 9 topic pages, `styles.css`,
`sitemap.xml`, `robots.txt`. There is no npm, no bundler, no framework. Whatever serves a
directory of files can host it.

---

## Option A — Cloudflare Pages, drag and drop (fastest, 2 minutes)

1. Sign in at <https://dash.cloudflare.com>.
2. **Workers & Pages** → **Create application** → **Pages** → **Upload assets**.
3. Project name: `tls-cert-errors` (this becomes `https://tls-cert-errors.pages.dev`).
   If that name is taken, use `certdecode` — the site does not care about the subdomain,
   but see "After deploy: fix the canonical URLs" below.
4. Drag the **contents of `sites/tls-cert-errors/`** (the files, not the folder itself) into the
   upload box.
5. Click **Deploy site**. The site is live at `https://<project>.pages.dev`.

No build command, no output directory — the uploaded files are served as-is.

---

## Option B — Cloudflare Pages, direct upload API (repeatable)

Set the credentials first:

```bash
export CF_API_TOKEN="<your scoped API token>"      # Cloudflare → My Profile → API Tokens → Create
export CF_ACCOUNT_ID="<your account id>"           # Cloudflare dashboard → Account ID
export SITE_DIR="sites/tls-cert-errors"
export PROJECT="tls-cert-errors"
```

### B1. Create the project (once)

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/pages/projects" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data "{\"name\":\"$PROJECT\",\"production_branch\":\"main\"}"
```

A `400` with "project already exists" means it is fine — continue to B2.

### B2. Build the asset manifest

The Pages direct-upload API wants a JSON object mapping each asset path to the SHA-256 of its
contents:

```bash
cd "$SITE_DIR"
manifest=$(python3 - <<'PY'
import hashlib, json, os
m = {}
for root, dirs, files in os.walk("."):
    dirs[:] = [d for d in dirs if not d.startswith(".")]
    for f in files:
        p = os.path.relpath(os.path.join(root, f), ".")
        with open(p, "rb") as fh:
            m[p] = hashlib.sha256(fh.read()).hexdigest()
print(json.dumps(m))
PY
)
echo "$manifest" > manifest.json
```

### B3. Create the deployment

```bash
deploy=$(curl -sS -X POST \
  "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/pages/projects/$PROJECT/deployments" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -F "branch=main" \
  -F "manifest.json=@manifest.json;type=application/json")

echo "$deploy" > deployment.json
upload_url=$(echo "$deploy" | python3 -c 'import json,sys; print(json.load(sys.stdin)["result"]["upload_url"])')
```

### B4. Upload the assets

```bash
args=()
while IFS= read -r f; do args+=(-F "file=@$f;filename=$f"); done < <(find . -maxdepth 1 -type f ! -name manifest.json ! -name deployment.json)

curl -sS -X POST "$upload_url" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -F "hashes=$manifest" \
  "${args[@]}"
```

The response contains the deployment id and the live URL.

### B5. Verify

```bash
curl -sI "https://$PROJECT.pages.dev/" | head -1        # expect HTTP/2 200
curl -sS "https://$PROJECT.pages.dev/sitemap.xml" | head -5
```

---

## Option C — Git-based deploy (recommended for future edits)

1. Create an empty repository (private is fine).
2. Push the contents of `sites/tls-cert-errors/` to its root.
3. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**,
   pick the repo.
4. Build settings: **Framework preset: None**, **Build command: leave empty**,
   **Build output directory: `/` (or `.`)**, root directory blank.
5. Every push to the production branch redeploys.

---

## Option D — any other static host

The files are host-agnostic. GitHub Pages, Netlify, Vercel, S3 + CloudFront and plain nginx all
work with no changes:

```nginx
server {
    listen 443 ssl;
    server_name certdecode.example.com;
    root /var/www/certdecode;
    index index.html;
    location / { try_files $uri $uri.html =404; }
}
```

---

## After deploy: fix the canonical URLs

Every page carries a canonical link pointing at `https://certdecode.pages.dev/`. If the live
subdomain differs (for example `https://certdecode.pages.dev` vs `https://tls-cert-errors.pages.dev`),
or if you attach a custom domain, update them in one pass before submitting the sitemap:

```bash
cd "$SITE_DIR"
OLD="https://certdecode.pages.dev"
NEW="https://<your-real-host>"
grep -rl "$OLD" . --include="*.html" | xargs sed -i "s|$OLD|$NEW|g"
grep -rl "$OLD" . --include="*.xml" --include="*.txt" | xargs sed -i "s|$OLD|$NEW|g"
```

Then re-upload. The same replacement is needed in `robots.txt` (the `Sitemap:` line) and
`sitemap.xml` (every `<loc>`).

## After deploy: tell the search engines

- Google Search Console → **Sitemaps** → submit `https://<your-real-host>/sitemap.xml`.
- Bing Webmaster Tools → **Sitemaps** → same URL.
- Optional: `curl -s https://<your-real-host>/robots.txt` to confirm it is served, and
  check the site in Google's URL Inspection tool once it has crawled the page.

---

## Monetization placeholders (human step)

The pages contain placeholder comments that are invisible to visitors:

| Placeholder | Where | What to do |
|---|---|---|
| `<!-- ADSENSE-SLOT -->` | twice on the index page, once on each topic page (11 total) | Replace with the AdSense unit markup once the AdSense account exists. Do not add ad code before the account is approved. |
| `<!-- AFFILIATE: certificate-and-tls-tooling -->` | index page | Replace with a relevant affiliate link (certificate/ACME tooling). |
| `<!-- AFFILIATE: private-pki-tooling -->` | self-signed page | Replace with a private-PKI affiliate link. |
| `<!-- AFFILIATE: managed-certificate-service -->` | no-local-issuer page | Replace with a managed certificate service link. |
| `<!-- AFFILIATE: certificate-verification-tooling -->` | OpenSSL codes page | Replace with a certificate verification tool link. |
| `<!-- AFFILIATE: node-tls-tooling -->` | Node.js page | Replace with a Node.js/TLS tooling link. |
| `<!-- AFFILIATE: python-tls-tooling -->` | Python page | Replace with a Python TLS tooling link. |
| `<!-- AFFILIATE: tls-debugging-tooling -->` | curl page | Replace with a TLS debugging tool link. |
| `<!-- AFFILIATE: developer-workstation-tooling -->` | Git page | Replace with a developer workstation tool link. |
| `<!-- AFFILIATE: certificate-management-tooling -->` | openssl cheatsheet | Replace with a certificate management tool link. |
| `<!-- AFFILIATE: acme-certificate-provider -->` | Let's Encrypt page | Replace with an ACME certificate provider link. |

A search for `ADSENSE-SLOT` and `AFFILIATE:` in the directory lists every slot.
Two editorial rules for whoever fills them in: only link products you would use yourself, and keep
the link inside the section that discusses that exact problem. The site is positioned as a neutral
reference; undisclosed affiliate links would undermine that.

---

## Local preview (no deploy needed)

```bash
cd sites/tls-cert-errors
python3 -m http.server 8000
# open http://localhost:8000/
```

`index.html` and every topic page are plain HTML with relative links, so the directory can be
opened directly from disk as well.

---

## Pre-publish checklist

- [ ] `index.html` + 9 topic pages open without broken links (all internal links are relative and verified).
- [ ] `styles.css` loads — the site is unstyled if it 404s.
- [ ] `robots.txt` and `sitemap.xml` are served from the root.
- [ ] Canonical URLs updated to the real host.
- [ ] Sitemap submitted to Search Console and Bing.
- [ ] Monetization placeholders left empty until the human has the accounts.
- [ ] The footer disclosure ("drafted with AI assistance and reviewed against upstream sources")
      stays in place — do not remove it.
