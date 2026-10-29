# SixCheck – IPv6 Website and Dependency Readiness Auditor

**AIORI Hackathon 3 · Problem Statement PS-027 · Internet Measurement · Team ID `A3-PS027-TC126`**

**Live demo: https://airoi-hckathon.vercel.app**

SixCheck audits a website for IPv6 readiness. Enter a domain and it checks DNS, mail, web servers, TLS and every third-party host the home page loads from, then returns a 0–100 score and a prioritised fix list.

<!-- Add a screenshot: save it as docs/screenshot.png and uncomment the next line -->
<!-- ![SixCheck screenshot](docs/screenshot.png) -->

## Why it matters

A site that works on IPv4 can still break for IPv6-only users. The usual causes are a missing AAAA record, nameservers without IPv6, a hard-coded IPv4 address, or one IPv4-only third-party script that stalls the page. SixCheck finds these before users do.

## What it checks

| # | Check | Weight | What is tested |
|---|-------|--------|----------------|
| 1 | Apex AAAA | 15 | Root domain resolves to an IPv6 address |
| 2 | www AAAA | 10 | `www` host resolves over IPv6 |
| 3 | Nameservers | 15 | Authoritative nameservers have AAAA records (2 or more for a pass) |
| 4 | Mail (MX) | 5 | Mail exchangers have AAAA records |
| 5 | HTTPS on IPv6 | 20 | TCP and TLS connection to port 443 over IPv6 |
| 6 | TLS on IPv6 | 10 | Certificate is valid for the hostname and not near expiry |
| 7 | Dependencies | 15 | Third-party hosts found on the home page have AAAA records |
| 8 | IPv6-only test | 10 | Static analysis: IPv4 literals in the page, IPv4-only dependencies |

Each check ends as **Pass**, **Review**, **Fail** or **Not tested**. Score = weighted sum of passed checks (Review counts half), scaled to 100. Checks marked Not tested are left out of the score.

The score maps to a verdict: 85 and above is *IPv6 ready*, 60 to 84 is *Partly ready*, below 60 is *Not ready*.

## How it works

1. **Resolve:** A, AAAA, NS and MX records are looked up through public resolvers.
2. **Crawl:** the home page is fetched over HTTPS and the hosts of scripts, styles, images and frames are collected (up to 20 third-party hosts).
3. **Probe:** a TLS connection is opened to the site's IPv6 address, and the certificate is inspected.
4. **Analyse:** the page is checked for IPv4 literals and IPv4-only dependencies.
5. **Score:** the eight results are weighted, ranked and returned as JSON.

The frontend renders the result as an IPv6-style address (one group per check), a score ring, a checks table, a dependency table and a short fix list.

## Tech stack

- **Frontend:** plain HTML, CSS and JavaScript (no build step)
- **Backend:** one Node.js function, no npm dependencies (`node:dns`, `node:net`, `node:tls`, `fetch`)
- **Hosting:** Vercel (serverless), or any Linux VPS with IPv6 via Docker

## Project structure

```
ps027-ipv6-auditor/
├── api/audit.js         audit backend: GET /api/audit?domain=example.org
├── public/
│   ├── index.html       frontend
│   ├── css/style.css
│   └── js/app.js
├── server.js            standalone server for a VPS (static files + API + rate limit)
├── Dockerfile
├── docker-compose.yml
├── vercel.json
├── package.json
├── DEPLOY-VPS.md        full-IPv6 deployment guide
└── README.md
```

## API

```
GET /api/audit?domain=wikipedia.org
```

Response (shortened):

```json
{
  "domain": "wikipedia.org",
  "mode": "live",
  "ipv6Egress": false,
  "score": 100,
  "verdict": "IPv6 ready",
  "checks": [
    { "k": "Apex AAAA", "w": 15, "status": "pass", "detail": "AAAA: 2001:db8::1", "fix": "" }
  ],
  "deps": [
    { "host": "fonts.googleapis.com", "v6": true, "v4": true }
  ]
}
```

`status` is one of `pass`, `warn`, `fail`, `info` (info means not tested). Invalid or private targets return HTTP 400.

## Run locally

```bash
npm i -g vercel
vercel dev          # open http://localhost:3000
```

Or, without Vercel:

```bash
node server.js      # Node 18 or newer, open http://localhost:3000
```

Opening `public/index.html` directly also works, but without the backend it shows clearly labelled simulated demo results.

## Deploy

**Vercel:** import the repository at vercel.com, keep the framework as *Other*, leave the build command empty and press Deploy.

**VPS with IPv6 (all 8 checks):**

```bash
docker compose up -d --build
```

See [DEPLOY-VPS.md](DEPLOY-VPS.md) for the full steps, HTTPS with Caddy and how to confirm IPv6 works.

## Limitations

- Vercel serverless functions have no outbound IPv6. The backend detects this, marks **HTTPS on IPv6** and **TLS on IPv6** as *Not tested* and excludes them from the score. Run the Docker version on an IPv6-enabled VPS to test them.
- The IPv6-only check is static analysis of the home page, not a live DNS64/NAT64 network.
- Only the home page is crawled; dependencies loaded later by JavaScript are not seen.
- Rate limiting (20 audits per minute per client) is built into `server.js`. The Vercel deployment has none.

## Security

The backend only audits public domains. Localhost, private, loopback, link-local and similar addresses are blocked, including after redirects, to prevent server-side request forgery. Page downloads are capped in size and time.

## Standards referenced

RFC 8200 (IPv6), RFC 3596 (AAAA records), RFC 3901 and RFC 9386 (IPv6 in DNS and deployment), RFC 8305 (Happy Eyeballs v2), RFC 6146 and RFC 6147 (NAT64 and DNS64), RFC 6724 (address selection).

## Team

AIORI Hackathon 3, PS-027, Team ID `A3-PS027-TC126`.
