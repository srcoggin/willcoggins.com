# Deploying willcoggins.com

Follow this in order. **DNS is the last step, not the first** — get the site
working on a `.pages.dev` URL first, so if something breaks you know whether it's
the site or the DNS.

> ## ⚠️ Read this before you open the Cloudflare dashboard
>
> **Your email works now. There is exactly one way to break it: Cloudflare Email
> Routing.** The dashboard promotes it. Its setup adds *and locks* its own MX and
> SPF records, overwriting Zoho's, and your mail silently stops arriving.
> **If you see an Email Routing prompt, banner or "Get started" card — dismiss
> it.** Nothing here needs it.
>
> The other risk is a slip of the hand: you'll be deleting rows from a table that
> also holds your MX and SPF records. **Only ever delete rows whose Type column
> reads `A` or `AAAA`.** Read the Type on every single deletion.

---

## Step 0 — Snapshot your DNS (2 min)

Cloudflare dashboard → **willcoggins.com** → **DNS** → **Records** →
**Import and Export** → **Export**.

Save the file somewhere outside the browser. It records proxy state inline, so
it's a complete restore source if anything goes wrong.

Then take a command-line snapshot to compare against later:

```bash
for t in NS SOA A AAAA MX TXT; do echo "### $t"; nslookup -type=$t willcoggins.com 1.1.1.1; done
```

## Step 1 — Put the repo on GitHub

The site is committed locally on branch `main`. Create the remote:

1. Go to **https://github.com/new**
2. **Repository name:** `willcoggins.com` (or `portfolio` — your call)
3. **Public** is recommended. Your site links to your GitHub as the projects
   story, so a real repo there works in your favour. There are no secrets in it —
   your master CV is gitignored.
4. **Do not** tick "Add a README", "Add .gitignore" or "Choose a license" — the
   repo already has them and it'll cause a conflict.
5. **Create repository**

Then push (swap in your repo URL):

```bash
cd "C:/Users/Will/Portfolio Website"
git remote add origin https://github.com/srcoggin/willcoggins.com.git
git push -u origin main
```

## Step 2 — Cloudflare Pages project

1. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** →
   **Connect to Git**
2. Authorise GitHub, pick the repo
3. Build settings — this is a plain static site, so leave it all empty:
   - **Framework preset:** `None`
   - **Build command:** *(empty)*
   - **Build output directory:** `/`
4. **Save and Deploy**

## Step 3 — Verify on pages.dev BEFORE touching DNS

Open the `https://<project>.pages.dev` URL Cloudflare gives you. Check:

- the page renders, fonts load, dark mode toggle works
- **Download my CV** gives you `WillCoggins 2026 CV.pdf`

**Do not continue until this works.** This is the whole point of the ordering: it
separates "is the site broken" from "is the DNS broken".

## Step 4 — Delete the dead records

DNS → Records. The apex and `www` currently point at Cloudflare proxy IPs
(`104.21.x` / `172.67.x`) whose origin server is long dead — that's the timeout
you get at willcoggins.com today.

Delete **every** `A` and `AAAA` row on:
- `willcoggins.com` (shown as `@`)
- `www`

**Check the Type column on each one.** Leave MX, TXT, and everything else alone.

You cannot make the website worse by doing this — it's already dead. The only
thing with real downside is the mail records sitting next to them.

## Step 5 — Add the domain in Pages

**Workers & Pages** → your project → **Custom domains** → **Set up a domain** →
enter `willcoggins.com` → **Continue**.

Cloudflare creates the CNAME itself, proxied (orange cloud). Wait for **Active**.

**Do not hand-create the DNS record.** A CNAME pointing at Pages without the
domain being registered in the Pages project produces a **522** — the same error
your site shows now. The dashboard is what creates that binding. Let it.

Then repeat for `www.willcoggins.com`.

## Step 6 — Redirect www → apex

**Rules** → **Overview** → **Create rule** → **Redirect Rule**

- **Name:** `www to apex`
- **When incoming requests match** → Custom filter expression:
  `(http.host eq "www.willcoggins.com")`
- **Then** → **Target URL** → *Dynamic* →
  `concat("https://willcoggins.com", http.request.uri.path)`
- **Status code:** `302` **for now**
- **Preserve query string:** ON
- **Deploy**

**Use 302 first.** A 301 is cached near-permanently by browsers — get the target
wrong and you've poisoned every visitor with no way to recall it. Verify, *then*
edit the rule to 301.

## Step 7 — Verify

```bash
# Site serves on both hostnames
curl -sI https://willcoggins.com | head -1                      # expect 200
curl -sI https://www.willcoggins.com/ | grep -Ei "^HTTP|^location"  # expect 302 -> apex
curl -sI "https://www.willcoggins.com/foo?a=1" | grep -i "^location" # path+query preserved

# Apex resolves to A records (CNAME flattening), not a CNAME
nslookup -type=A willcoggins.com 1.1.1.1

# The PDF is reachable
curl -sI https://willcoggins.com/resume.pdf | head -1            # expect 200
```

**MX unchanged — run before, during and after.** Ask the authoritative server so
you bypass caching:

```bash
nslookup -type=MX willcoggins.com melina.ns.cloudflare.com
```

Expect exactly:
```
10 mx.zoho.com.au
20 mx2.zoho.com.au
50 mx3.zoho.com.au
```

**Then send yourself a real email at will@willcoggins.com.** DNS looking right is
necessary, not sufficient.

Once all that passes, go back and flip the redirect rule from `302` to `301`.

## Why the apex CNAME won't break your mail

Normally a CNAME can't coexist with other records at the same name, which would
kill your MX. Cloudflare's **CNAME flattening** resolves the CNAME at the edge
and hands resolvers synthesised `A` records — no resolver ever sees a CNAME at
your apex, so the rule is never violated. It's automatic on all plans and needs
no setting.

Do **not** turn on "Flatten all CNAMEs". You don't need it and it breaks
third-party hostname verification.

## Rollback

Re-create the A/AAAA records from your Step 0 export, proxied. Note this restores
the **broken** state — it's a rollback to the status quo, not to a working site.

## After it's live

- Google Search Console → add `willcoggins.com`, submit `sitemap.xml`
- Test the JSON-LD: https://search.google.com/test/rich-results
- Consider DMARC at `p=none` (monitor-only, changes no delivery behaviour) — but
  do it in a **separate session**, not alongside a DNS cutover. Different blast
  radius, different day.
