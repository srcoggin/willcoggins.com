# willcoggins.com

Personal portfolio / résumé site. Plain HTML + CSS, no framework, no build step,
no dependencies. Deployed on Cloudflare Pages.

```
index.html      all content + Person JSON-LD
styles.css      screen styles + print stylesheet
resume.pdf      the public download (scrubbed — no phone, no address)
fonts/          IBM Plex Sans + Mono, self-hosted woff2
favicon.svg
robots.txt
sitemap.xml
resume-source/  master CV .docx — GITIGNORED, never published
```

## Editing content

Everything lives in `index.html`. There's no CMS and no templating — edit the
`<main>` block directly.

Two rules:

1. **Keep the JSON-LD in sync** with the visible content. It's the block at the
   top of `<head>`; it's what makes Google understand who you are.
2. **Nothing private goes in.** Email and suburb only. No phone, no street
   address, no referee names. The repo is public and so is the site.

## Deploying

Cloudflare Pages watches the default branch. Push and it redeploys.

```bash
git add -A
git commit -m "Update résumé"
git push
```

Build settings: framework preset `None`, no build command, output directory `/`.

## Updating the résumé PDF

The master is `resume-source/Will Coggins CV.docx` (gitignored — see
`resume-source/EDITING-NOTES.md`). Edit it in Word, export to PDF, and replace
`resume.pdf` in the repo root. Then update `index.html` to match so the page and
the download don't drift apart.

`Ctrl+P` on the live page also produces a clean résumé — the print stylesheet
hides the nav, keeps each job entry from splitting across a page break, and
prints link URLs. Useful as a check that the HTML and the PDF still agree.

## Local preview

```bash
python -m http.server 8000
# http://localhost:8000
```

Needs a real server, not `file://` — the absolute paths (`/styles.css`,
`/fonts/…`) won't resolve otherwise.

## Domain notes

DNS is on Cloudflare (`melina`/`harlan.ns.cloudflare.com`). The apex uses
CNAME flattening to point at Pages.

**⚠️ Email lives on this domain and it WORKS. Don't break it.**

`will@willcoggins.com` is live on **Zoho Mail AU** (verified by SMTP probe: real
address returns `250 OK`, a nonexistent one returns `550`, which is the healthy
pattern). Current mail records — leave all of these alone:

```
MX  10  mx.zoho.com.au          TXT  v=spf1 include:one.zoho.com.au include:zohomail.com.au ~all
MX  20  mx2.zoho.com.au         TXT  zoho-verification=zb78263985.zmverify.zoho.com.au   (live)
MX  50  mx3.zoho.com.au         TXT  zoho-verification=zb11659946.zmverify.zoho.com.au   (stale, harmless)
```

When touching DNS for the website:

- Only ever delete records whose **Type** column reads `A` or `AAAA`. The site
  records and the mail records share one table.
- **Never enable Cloudflare Email Routing.** Its onboarding adds *and locks* its
  own MX and SPF records, overwriting Zoho's and silently redirecting inbound
  mail. This warning is live again now that Zoho actually works — earlier notes
  in this repo said Email Routing was the plan, from when Zoho was broken. It
  isn't. Ignore that.
- Don't touch the SPF record. It's correct for the AU data centre
  (`zohomail.com` in Zoho's public docs is the **US** value and would be wrong).
- The `zb11659946` token is a leftover from the first, failed domain setup. It's
  inert — removing it is optional and gains nothing.
