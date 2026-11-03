---
layout: post
title: "A public IP and device lookup tool on Cloudflare Workers, and why public is fine"
date: 2026-11-03 09:00:00 +1000
categories: [tools]
---

## Why I did this

Plenty of places have a "what's my IP" page that returns a single line. I wanted one that answers the next three questions people always ask: where does this connection appear to be, who is the provider, and what does the browser on the other end give away about itself. Cloudflare already knows most of this about every request it handles, so the question became how little code it takes to surface it.

The shape I wanted was close to what speedtest sites do: a fixed, memorable address, nothing to log in to, and a page that is useful in a browser and still scriptable from a terminal.

## What I built / tried

**The idea.** A Cloudflare Worker is a small piece of JavaScript that runs on Cloudflare's edge. Every request that reaches it arrives with a `request.cf` object that Cloudflare fills in from its own network data: country, region, city, timezone, ASN (the number that identifies a network), the operator's name, coordinates, and the TLS and HTTP details of the connection. The caller's address is in the `CF-Connecting-IP` header. There is no lookup API, no database and no server to run.

```js
const cf = request.cf || {};
const ip = request.headers.get("CF-Connecting-IP");
// cf.country, cf.city, cf.asn, cf.asOrganization, cf.timezone ...
```

**One Worker, three behaviours.** Content negotiation does the routing. `curl` gets a plain one-liner with just the IP. `/json` returns the server-side data as JSON. A browser asking for HTML gets the full page.

**The page.** I wanted it dark with a cyber and glitch feel: scanline overlay, a grid background, cyan and magenta accents, and an RGB-split glitch effect on the big IP header. It is plain HTML and CSS in a single template string, with no framework and no build step.

**Server-side versus client-side.** The edge can tell you about the network. It cannot tell you about the device. A server only sees what a browser volunteers in headers. Anything richer has to come from JavaScript running in the visitor's own browser: screen and viewport size, pixel ratio, CPU thread count, approximate memory, GPU renderer, languages, browser timezone, connection type, battery, and a WebRTC test that shows whether a VPN is leaking the real address. All of it stays in the browser.

**Two views from one source.** I split it into an `ip` flavour and a `device` flavour. The first is connectivity only: network, location and a map. The second shows everything, with the client-side cards first. It is one `worker.js` deployed twice, with a `MODE` variable selecting the view. Copying the file would have meant fixing every bug twice.

**A map without a map library.** The map is a 3-by-5 grid of OpenStreetMap tiles placed with percentages, recoloured dark with a CSS filter. A pin marks the coordinates and a dashed ring about 25 km across reminds the viewer that this is city-level, not street-level. It is about thirty lines of code.

**Short aliases.** A third, tiny Worker redirects shorter alias hostnames to the real ones. It only knows an allowlist of hostnames, so it cannot be used as an open redirect.

**Deploying.** `wrangler` is Cloudflare's CLI. One config per Worker, then `wrangler deploy`. A Custom Domain on the Worker makes Cloudflare create the DNS record and the certificate itself.

## Why it's public, on purpose

Cloudflare Access would put a login in front of it. I decided against that because of what this tool actually does.

- **It only reflects what the caller already sent.** The IP and the User-Agent belong to the person making the request. The edge data is Cloudflare's own estimate about that IP. Nothing private to me is behind the page.
- **There is nothing to protect on the server side.** The Worker has no storage bindings, no secrets, and nothing it can reach other than the request in front of it.
- **A login would defeat the point.** The whole value is "open a browser or run `curl` and get an answer". A sign-in wall breaks the scriptable half.
- **The failure mode is cheap.** If someone hammers it, the free plan caps requests per day and the tool just stops answering. There is no bill and nothing leaks.
- **It uses no origin of mine.** Cloudflare terminates the connection at the edge. No port forwarding, no tunnel into the homelab, no exposed host.

Public does not mean careless. I kept the footprint small on purpose:

- Nothing is stored or logged. Observability is off and the code has no `console.log`.
- Server-side fields are limited to what is useful. I dropped the postcode and the User-Agent echo, and rounded coordinates to two decimals (about 1 km).
- A strict Content-Security-Policy with a fresh nonce on every response, plus `no-store` caching, `nosniff` and a locked-down Permissions-Policy.
- The WebRTC leak test is opt-in. The page makes no third-party STUN request unless you click the button.
- The CORS wildcard on `/json` is gone, so another website's script cannot read it from a visitor's browser.

## What broke (or surprised me)

- **My first API token was read-only.** It could list Workers and zones, but the deploy returned a 403. The fix was a separate token scoped to Workers Scripts, Workers Routes and DNS on the two zones involved, not a wider all-purpose one.
- **I had to be able to test without the real edge.** `request.cf` only exists on Cloudflare's network, so locally I mocked it. That proves the page and the logic, not the geo data. I only believed the geolocation after hitting the real deployment.
- **A strict CSP forced a cleanup.** Inline `style=` attributes are blocked unless you allow unsafe inline styles. I moved them into classes and set positions through the DOM, so the page needs no relaxation of the policy.
- **Two classic injection traps.** Embedding server JSON into a `<script>` tag means escaping `<` or a hostile header can close the tag. And `String.replace` treats `$&` specially in the replacement text, so a User-Agent containing it can mangle the page unless the replacement is a function.
- **OpenStreetMap blocked my first tiles.** I send `Referrer-Policy: no-referrer` everywhere, and OSM refuses tile requests with no referrer. Only the map images now send the bare origin.
- **I misread my own requirement once.** The connectivity page was supposed to hold network and location only, but it showed everything in a different order. The fix was that the client-side code should not run at all in that mode, not just be hidden.
- **Removing and re-adding a hostname leaves it broken for a while.** I removed one Custom Domain and recreated it shortly after as a redirect. Resolvers had cached "does not exist" and kept answering that for up to the zone's negative-cache TTL, about 30 minutes. The record and Worker were correct from the start. Checking the authoritative nameserver and a few public resolvers is what showed it was caching and not config.
- **A redirect cannot answer a `curl`.** `curl alias.example.net` gets a 302 with an empty body unless you add `-L`. For a one-word terminal command you have to serve the real Worker on that hostname instead of redirecting.

## What I learned

- Cloudflare gives you the network side for free. The device side needs the browser, so decide early which half you are building.
- If a value does not need to leave the server, it should not. Dropping fields is cheaper than defending them.
- Verify against the real edge before trusting any geo data, and verify DNS against the authoritative server before blaming the config.
- For a tool like this, "public on a hardened edge" was less risky than "private and clever".
- Geolocation is approximate. Show the uncertainty in the interface, as with the map ring, rather than implying precision.

## Next

More small tools alongside it, each one a Worker on its own subdomain: DNS lookup, header echo, TLS inspection. The code is in the `scripts` repo under `cloudflare-ip-tool`.
