# Swami Ishatmananda — Lecture Archive (landing page)

The bilingual landing page served at the apex domain **swami-ishatmananda.org**.
It lets visitors choose between the two lecture archives:

- **বাংলা / Bengali** → `bn.swami-ishatmananda.org` (repo: `bengali-lectures`)
- **English** → `en.swami-ishatmananda.org` (repo: `swami-ishatmananda`)

Pure static site — vanilla HTML/CSS/JS, no build step. Shares the parchment +
terracotta design language and Hind Siliguri / Zodiak / Satoshi fonts with the
two archives.

## DNS / hosting

Hosted on GitHub Pages. The `CNAME` file binds this repo to the apex domain.
The two subdomains are configured as the custom domains of their own repos.

## Run locally

```bash
python -m http.server 8001
# then visit http://localhost:8001
```
