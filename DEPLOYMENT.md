# Deployment & Infrastructure Runbook

How the **Swami Ishatmananda lecture archive** is hosted, and how to operate /
troubleshoot it. Written 2026-06-01.

---

## 1. Architecture at a glance

One custom domain serves three independent static sites via **GitHub Pages +
Cloudflare DNS**. A GitHub Pages custom domain can only bind to **one repo**, so
we use **subdomains** (not one-repo-with-subfolders).

| URL | Repo | Content |
|-----|------|---------|
| `https://swami-ishatmananda.org` | `nirupamchat/swami-ishatmananda-home` | Bilingual landing / language chooser |
| `https://bn.swami-ishatmananda.org` | `nirupamchat/bengali-lectures` | Bengali lecture archive |
| `https://en.swami-ishatmananda.org` | `nirupamchat/swami-ishatmananda` | English lecture archive |
| `https://www.swami-ishatmananda.org` | (alias of apex) | Redirects to landing |

- **Registrar + DNS:** Cloudflare (domain `swami-ishatmananda.org`, `.org`, ~$8.50/yr).
- **Hosting:** GitHub Pages (free).
- **TLS:** Free auto-renewing Let's Encrypt certs, issued automatically by GitHub.
- **Stack:** Pure static — vanilla HTML/CSS/JS, no build step, no dependencies.

---

## 2. DNS records (in Cloudflare)

All records must be **"DNS only" (grey cloud)** — NOT proxied (orange cloud).
Proxying breaks GitHub's HTTPS certificate provisioning.

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| A | `@` | `185.199.108.153` | DNS only |
| A | `@` | `185.199.109.153` | DNS only |
| A | `@` | `185.199.110.153` | DNS only |
| A | `@` | `185.199.111.153` | DNS only |
| AAAA | `@` | `2606:50c0:8000::153` | DNS only |
| AAAA | `@` | `2606:50c0:8001::153` | DNS only |
| AAAA | `@` | `2606:50c0:8002::153` | DNS only |
| AAAA | `@` | `2606:50c0:8003::153` | DNS only |
| CNAME | `bn` | `nirupamchat.github.io` | DNS only |
| CNAME | `en` | `nirupamchat.github.io` | DNS only |
| CNAME | `www` | `nirupamchat.github.io` | DNS only |

> The four `A` (IPv4) records are what GitHub validates for the apex. The four
> `AAAA` (IPv6) records are GitHub's recommendation but **not required** — the
> apex works on A records alone. The `bn`/`en`/`www` CNAMEs point at the GitHub
> Pages user host (`<username>.github.io`); they flatten to the same 185.199.x IPs.

---

## 3. GitHub Pages settings (per repo)

In each repo: **Settings → Pages**.

| Repo | Source | Custom domain | Enforce HTTPS |
|------|--------|---------------|---------------|
| `swami-ishatmananda-home` | Deploy from a branch: `main` / root | `swami-ishatmananda.org` | ✅ (after cert) |
| `bengali-lectures` | Deploy from a branch: `main` / root | `bn.swami-ishatmananda.org` | ✅ (after cert) |
| `swami-ishatmananda` | Deploy from a branch: `main` / root | `en.swami-ishatmananda.org` | ✅ (after cert) |

Setting a custom domain writes a `CNAME` file into the repo (committed
automatically). "Enforce HTTPS" only becomes available **after** the cert is
issued, which requires the DNS check to pass first.

---

## 4. How to update a site

Each site is its own repo. To publish a change (e.g. add a lecture series to
`data.js`, edit the landing page, add online-book links):

```bash
# inside the relevant repo folder
git add .
git commit -m "Describe the change"
git push
```

GitHub Pages auto-redeploys within ~1–2 minutes. No re-configuration needed.

Local repo folders on this machine:
- `C:\Users\0072938\Downloads\Swami Ishatmananda — Bengali Lecture Series` → `bengali-lectures`
- `C:\Users\0072938\Downloads\swami-ishatmananda-home` → landing page
- (English repo `swami-ishatmananda` — clone if not already local)

To preview locally before pushing:
```bash
python -m http.server 8000   # then open http://localhost:8000
```

---

## 5. Troubleshooting

### Symptom: GitHub shows "DNS check unsuccessful" / `InvalidDNSError`, and "Enforce HTTPS" is greyed out

This was the main hiccup during initial setup. The cause was **GitHub caching an
early failed DNS check** (run while DNS was still propagating), even though the A
records were correct. The dependency chain:

```
DNS check passes → GitHub issues Let's Encrypt cert → "Enforce HTTPS" unlocks
```

**The fix that worked — re-save the custom domain to force a clean re-check:**
1. Repo → **Settings → Pages → Custom domain**
2. **Clear** the domain field → **Save**
3. Wait ~10 seconds
4. **Re-enter** the domain → **Save**
5. The status changes to "DNS check in progress" → clears within a minute; cert
   issues a few minutes later; "Enforce HTTPS" then becomes tickable.

> Apex certs can legitimately take minutes up to ~24h, but if A records are
> correct and it's stuck, the re-save nudge clears it. **Don't chase the AAAA /
> IPv6 records** — they are not the blocker (apex works on A records alone).

### Symptom: HTTPS works but `http://` doesn't redirect
Tick **Enforce HTTPS** in that repo's Pages settings.

### Symptom: A subdomain returns HTTP 404 from GitHub
The DNS reaches GitHub but no repo claims that hostname — set the **Custom
domain** field on the intended repo (Section 3).

### Cloudflare gotchas
- **Grey cloud only.** Every record must be "DNS only," never proxied (orange),
  or GitHub HTTPS breaks.
- Delete any auto-created parking `A`/`AAAA` record on the apex.

---

## 6. Verification commands (PowerShell)

Confirm DNS resolves to GitHub's IPs:
```powershell
Resolve-DnsName swami-ishatmananda.org -Type A
Resolve-DnsName bn.swami-ishatmananda.org
```

Query Cloudflare's authoritative nameservers directly (bypasses caching):
```powershell
Clear-DnsClientCache
Resolve-DnsName swami-ishatmananda.org -Type A -Server daisy.ns.cloudflare.com
```

Check HTTPS + cert validity on all domains:
```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
foreach ($d in @("swami-ishatmananda.org","bn.swami-ishatmananda.org","en.swami-ishatmananda.org")) {
  try { $r = Invoke-WebRequest "https://$d" -UseBasicParsing -TimeoutSec 20; "$d -> HTTPS OK ($($r.StatusCode))" }
  catch { "$d -> FAIL: $($_.Exception.Message)" }
}
```

> Note: on some corporate networks, force TLS 1.2 (line above) for
> `Invoke-WebRequest`, and stderr from native git commands may appear red in
> PowerShell even on success.

---

## 7. Key facts to remember

- GitHub username: `nirupamchat`
- A GitHub Pages custom domain binds to exactly **one** repo → subdomains, not subfolders.
- The whole stack is free: domain ~$8.50/yr is the only recurring cost.
- Renewing the domain (Cloudflare) is the one thing that must not lapse.
