---
title: "From a Crashed Server to Cloudflare: A Migration Story"
date: 2026-07-16 00:00:00 +0900
categories: [DevOps, Infrastructure]
tags: [cloudflare, pm2, log-rotation, dns, ssl, workers, migration]
---

## Part 1: The Server Went Down

It started with a simple problem — our company's web server was unreachable. No graceful error page, no timeout warning. It had just stopped responding.

### Digging Into the Access Logs

The first move was SSH into the box and start looking at what happened right before the crash. The access logs told the story pretty quickly: there was a sustained pattern of requests coming in at a rate of more than **2 requests per second, continuously**, from what looked like a crawling bot rather than a human visitor.

At first glance this doesn't sound catastrophic — most servers can handle a couple requests per second without breaking a sweat. The actual problem was downstream of the traffic itself.

### The Real Cause: A Runaway Log File

Every one of those requests was being written to a log file. With no log rotation in place, that file grew and grew, request after request, day after day. Eventually it crossed the disk quota enforced by our hosting provider — and once that happened, the server couldn't write anymore, and everything ground to a halt.

So the actual root cause wasn't the bot traffic itself. It was the complete absence of a log rotation strategy. A relatively mundane, low-severity source (an aggressive crawler) turned into a full outage because nothing was capping how large a single log file was allowed to grow.

### The Fix

The recovery process was straightforward once the cause was clear:

1. **SSH into the server** and locate the offending log file.
2. **Back it up** before touching anything, in case there was anything worth investigating later (rate-limiting patterns, IP ranges, etc.).
3. **Delete the oversized log file** to immediately free up disk space.
4. **Install `pm2-logrotate`**, a module for the PM2 process manager that automatically rotates and truncates log files based on size and age, so this exact failure mode couldn't happen again.
5. **Restart the server process** and confirm it was serving requests normally again.

The `pm2-logrotate` piece was the important long-term fix here — it's easy to patch a symptom by deleting a big file once, but that does nothing to prevent the same thing from happening again next month. Automated log rotation means log files get capped and archived/compressed on a schedule, so no single file can silently balloon until it takes down the whole server.

**Takeaway:** if you're running anything long-lived behind PM2 (or really any process manager) and haven't explicitly set up log rotation, it's worth checking right now. It's an easy thing to overlook until it becomes an outage.

---

## Part 2: Migrating Hosting to Cloudflare

Once the immediate fire was out, it was a good moment to reconsider the whole hosting setup rather than just patch around the same infrastructure indefinitely. The decision was made to move the company's hosting away from the previous provider entirely and onto Cloudflare's platform (Workers, serving static assets).

This is a rundown of what that migration actually involved — mostly because a few of the steps weren't obvious going in.

### Step 1: Connecting the Domain to Cloudflare

The domain itself stayed registered with the original registrar — there was no need to transfer registration. All that changes is *who resolves DNS for the domain*. In Cloudflare's dashboard, this is the "Connect a domain" flow (as opposed to "Transfer a domain," which actually moves registration, or "Buy a domain").

After adding the domain, Cloudflare scans and imports the existing DNS records, then hands back two nameserver addresses that need to be set at the registrar.

### Step 2: Nameserver Propagation

This is the part that requires patience. After updating the nameservers at the registrar, there's a propagation window before Cloudflare recognizes the change:

- Locally, you can check with `nslookup -type=NS yourdomain.tld` — if it already returns Cloudflare's nameservers, the change has reached your local resolver.
- Cloudflare's own dashboard status (which shows "Active" once it recognizes the switch) can lag behind that, since it does its own independent verification from multiple locations. In practice this took a fairly short window, but the official upper bound Cloudflare cites is up to 24–48 hours.
- During this transition period, some visitors may still be served by the old host and others by the new one, depending on which nameserver answer their local DNS resolver has cached. It's not a hard outage — just an inconsistent window.

### Step 3: SSL Certificate Issuance

Once the domain shows as "Active," Cloudflare automatically starts issuing a Universal SSL certificate for the zone. This goes through a validation step (visible in the dashboard as "Pending Validation") before flipping to "Active." No manual certificate request or installation is required — it's fully automatic once the domain is under Cloudflare's management, and it applies to any proxied ("orange cloud") DNS record automatically.

### Step 4: The 522 Error

With the nameservers switched and SSL active, the site still didn't load — instead it returned a Cloudflare **522 "Connection timed out"** error. This error specifically means: the request successfully reached Cloudflare, but Cloudflare's attempt to reach the origin server timed out.

The root cause turned out to be how the DNS record was set up. The domain's CNAME record pointed directly at the `*.workers.dev` subdomain of the deployed Worker. While that address was reachable on its own, routing to it as a plain external CNAME meant Cloudflare was treating it as an outside destination and trying to reach it over the open internet — which doesn't work reliably for `workers.dev` addresses used this way.

The fix was to use Cloudflare Workers' **Custom Domains** feature instead of a manual CNAME: adding the domain directly under the Worker's "Domains" settings connects the domain to the Worker internally, without needing to route back out to the public internet. Once that was set up (for both the root domain and the `www` subdomain), the site loaded correctly.

One snag along the way: the existing CNAME record had to be deleted before Custom Domain registration would succeed, since Cloudflare detected the naming conflict. The first deletion attempt appeared to fail silently (the change hadn't actually propagated), which caused Custom Domain setup to fail with no clear error message — worth double-checking that a DNS record is actually gone (not just visually removed) before retrying.

### Cleanup

A few other DNS records were also cleaned up as part of the move — old mail-related `MX` and `TXT` records that were no longer in use, since email was being handled elsewhere and there was no reason to keep stale records pointing at a service no longer in use.

---

At this point, the site was fully migrated: domain resolving through Cloudflare, SSL issued and active, and the Worker serving requests directly through a Custom Domain rather than a routed CNAME. The next step was setting up CI/CD so that pushes to the repository would deploy automatically — but that's a separate story.
