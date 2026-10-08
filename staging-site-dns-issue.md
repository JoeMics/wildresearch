# stagingsite.wildresearch.ca — DNS issue summary

_Last checked: 2026-10-07_

## Problem

`http://stagingsite.wildresearch.ca/` does not load. The hostname does not resolve (NXDOMAIN) on any DNS resolver.

## How the domain is set up

| Role | Provider | Notes |
|---|---|---|
| Registrar | IONOS (Domain Robot / InterNetX) | Only holds the registration and nameserver setting |
| DNS (authoritative) | Web Hosting Hub, `ns1/ns2.webhostinghub.com` | Confirmed at the .ca registry |
| Web hosting | Web Hosting Hub, server `whub59` (`209.182.197.191`) | Main site and staging WordPress files are here |

**IONOS DNS settings are inactive.** Changes in the IONOS DNS or Subdomains tabs have no effect while the nameservers point to Web Hosting Hub.

## Root cause

- The cPanel Zone Editor has the correct record: `stagingsite.wildresearch.ca  A  209.182.197.191`.
- The live nameservers (`ns1/ns2.webhostinghub.com`) don't serve it. Their copy of the zone is stale: the SOA serial is still `2022091803` (Sept 2022).
- So cPanel DNS changes are not syncing to the nameservers. This is a Web Hosting Hub problem, and only their support can fix it.
- The same thing affects `staging.wildresearch.ca` and `coha.wildresearch.ca`.

## Status

- [x] Support ticket opened with Web Hosting Hub, asking them to resync the `wildresearch.ca` zone from whub59 to ns1/ns2
- [x] **Fixed as of 2026-10-07.** SOA serial is now `2026100708` on ns1 and ns2. `stagingsite.` and `coha.` resolve publicly to `209.182.197.191`, and `https://stagingsite.wildresearch.ca/` returns 200 with a valid certificate.

### Live zone check (2026-10-07)

| Record | Live value | Note |
|---|---|---|
| A `@`, CNAME `www` | `209.182.197.191` | OK |
| MX `@` | `0 wildresearch.ca.` | OK (mail stays on Web Hosting Hub) |
| A `stagingsite`, `coha` | `209.182.197.191` | OK |
| A `staging` | missing | Dead. `stagingsite` is the staging site going forward. |
| TXT `@` | two `google-site-verification=` values (`KAR7tJ…`, `b6NGyN…`) | **Done.** Google verification completed (2026-10-07) via a new process. |
| TXT `@` SPF | **missing** | **FLAGGED FOR LATER.** The SPF line seen on 2026-10-06 is gone. Re-add it (cPanel → Email Deliverability), or outgoing mail may land in spam. |
| CNAME `77wvqt2e4gvf` | missing | Not needed (verification done another way) |
| CNAME `k2._domainkey` | missing | Mailchimp DKIM, still to add |

### Message sent to support

> My domain wildresearch.ca is on server whub59 and uses ns1/ns2.webhostinghub.com. The cPanel Zone Editor has A records for stagingsite.wildresearch.ca (and staging., coha.) pointing to 209.182.197.191, but ns1/ns2.webhostinghub.com don't serve them (they return NXDOMAIN, and the SOA serial is still 2022091803). Please resync the wildresearch.ca DNS zone from my server to your nameservers.

## IONOS side

- [x] **Delete the `stagingsite` subdomain in IONOS.** (Done 2026-10-07.) It points to `74.208.215.84` (an IONOS server, wrong target). Deleting it also removes the greyed-out auto-created `stagingsite` records.
- **Do not change the nameservers to IONOS.** The IONOS default records (`@`/`www` → `74.208.215.84`, MX → `mx00/mx01.ionos.com`) would take over and break the main site and email.
- Leave the IONOS default records alone; they are inactive and harmless.
- Ask other users not to make further changes in IONOS (nameservers, "Adjust destination", SSL).

### Records added in IONOS that are NOT live

Users added these in IONOS, so they aren't working. Re-add them in the **cPanel Zone Editor** after support fixes the sync:

| Type | Host | Value | Purpose |
|---|---|---|---|
| TXT | `@` | `google-site-verification=ufXHF7Wm5cfnZYtlb…` (copy full value from IONOS) | Google Search Console / Workspace verification |
| CNAME | `77wvqt2e4gvf` | `gv-e3olvo55kmvlyp.dv.googlehosted.com` | Google domain verification |
| CNAME | `k2._domainkey` | `dkim2.mcsv.net` | Mailchimp DKIM |

## After support fixes the sync

1. Verify the record is live:
   ```
   dig +short stagingsite.wildresearch.ca @ns1.webhostinghub.com   # expect 209.182.197.191
   dig +short stagingsite.wildresearch.ca                           # public; may lag a few hours (negative cache)
   ```
2. cPanel → **SSL/TLS Status** → run **AutoSSL** for the subdomain.
3. Add the three Google/Mailchimp records above in the cPanel Zone Editor, then re-run verification in Google and Mailchimp.
4. Clean up: two staging WordPress installs exist (`staging.` and `stagingsite.`). Keep one and remove the other via Softaculous.
5. Hide staging from search engines: WordPress → Settings → Reading → "Discourage search engines", and/or cPanel → Directory Privacy (password).

## Google for Nonprofits / Workspace verification

The same DNS problem is blocking Google domain verification for `wildresearch.ca`.

- The Google verification records were added in **IONOS**, which isn't the active DNS, so Google can't see them.
- Checked on Google's resolver (`8.8.8.8`) and on both Web Hosting Hub nameservers: the only TXT record is the existing SPF line. The `google-site-verification` TXT and the `77wvqt2e4gvf` CNAME are both missing.
- Adding them in cPanel won't work until support fixes the zone sync.

### Steps

1. Wait for Web Hosting Hub to fix the zone sync (and mention to support that Google verification is also blocked).
2. Add in the **cPanel Zone Editor**:
   - TXT `@` → `google-site-verification=…` (full value from IONOS or the Google admin console)
   - CNAME `77wvqt2e4gvf` → `gv-e3olvo55kmvlyp.dv.googlehosted.com`
3. Check they're public:
   ```
   dig +short TXT wildresearch.ca @8.8.8.8
   dig +short CNAME 77wvqt2e4gvf.wildresearch.ca @8.8.8.8
   ```
4. Click **Verify** in Google.

### Caution: email (MX)

Verifying the domain is safe. Setting up **Gmail** means changing MX records, which moves *all* email for `wildresearch.ca`; it currently goes to the Web Hosting Hub server. Plan that separately and move existing mailboxes first.

### Fallback if support stalls

Move DNS to IONOS: recreate every live record there (main site A, `www`, mail/MX, SPF, staging, Google), replace the IONOS defaults that point to `74.208.215.84` / `mx00.ionos.com`, then switch the nameservers in IONOS. More work and more risk, so use it only if Web Hosting Hub can't fix the sync.

## Previous support attempt (~2 months ago, AI chat agent)

Their AI agent said that because the domain doesn't appear in AMP, DNS is "managed at the registrar." **That was wrong:** AMP only lists domains *registered* with Web Hosting Hub. DNS is handled by whichever nameservers the domain points to, here `ns1/ns2.webhostinghub.com` (check with `dig NS wildresearch.ca @any.ca-servers.ca`). This is probably why records were added in IONOS. Ask for a human agent this time.

## Useful commands

```
# Does the staging record exist on the live nameserver? (look for NXDOMAIN + stale SOA serial)
dig A stagingsite.wildresearch.ca @ns1.webhostinghub.com

# Zone serial only (cPanel format YYYYMMDDNN; 2022091803 = not updated since Sept 2022)
dig +short SOA wildresearch.ca @ns1.webhostinghub.com

# Which nameservers the .ca registry delegates to
dig NS wildresearch.ca @any.ca-servers.ca
```

## Resume the Claude Code session

From `/home/jmicla/Repositories/wildresearch`:

```
claude --resume cbf36b43-e14b-40c5-bf94-9cf7a2e64d43
```

(or `claude --continue` if it's the most recent session in that directory)

## Side note

The server returns `406 Not Acceptable` to non-browser clients (e.g. plain `curl`), probably a ModSecurity rule. Browsers get `200`, so it doesn't cause this problem.
