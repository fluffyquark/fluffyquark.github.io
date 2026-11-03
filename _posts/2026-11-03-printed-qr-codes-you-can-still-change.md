---
layout: post
title: "Printed QR codes you can still change: a redirect Worker and open-source styling"
date: 2026-11-03 12:30:00 +1000
categories: [tools]
---

## Why I did this

A QR code is a picture of some text. Once it's printed on a poster, a flyer or a sign, that text is fixed. If the page it points to moves, or you want it to point somewhere new, the only option is to reprint. I wanted printed codes that could outlive the pages behind them, and I wanted them to look good enough that people would be happy to put them on things.

## The trick: put a redirect in the middle

The code doesn't encode the real destination. It encodes a short link on a domain I control, such as `qr.example.com/wifi`. That link never changes. A small Cloudflare Worker sits behind it and decides, at scan time, where the visitor actually lands. Change the destination, redeploy, and every code I've ever printed follows.

Paid "dynamic QR code" products are built on the same idea: a short URL on their domain that redirects to yours. That works well until the subscription lapses or the terms change, because the printed codes point at *their* domain, not yours. Owning the hostname means the codes stay under my control for as long as I keep the domain.

## What I built

**The redirector.** It's about 25 lines. It takes the first part of the URL path, looks it up in a fixed list of links, and sends back a redirect. No database, no storage, no logging.

```js
const LINKS = {
  website: "https://www.example.org/",
  wifi: "https://www.example.org/guest-wifi",
};
// slug -> LINKS[slug] -> 302 redirect, or 404 if it isn't in the list
```

Three choices in it are deliberate:

- **302, not 301.** Browsers and scanner apps cache a 301 permanently, so a destination change wouldn't reach anyone who had scanned once already. A 302 is re-checked each time, and the extra hop costs almost nothing.
- **A fixed list, never a parameter.** The destination comes only from the list, never from the URL. A redirector that accepts `?url=` is an open redirect, which is a gift to phishers.
- **Forgiving slugs.** Case and trailing slashes are ignored, because codes get retyped and re-cased by humans.

**The generators.** The redirector solves the "content" half. The "look" half is where open-source tooling does the work. A plain QR code is square black modules on white. Styling means changing how the modules are drawn and coloured, while leaving the grid itself alone. Several open-source libraries do this:

- **python-qrcode** is pure Python with a BSD-style licence. Its `StyledPilImage` factory takes a module drawer (square, rounded, circle, gapped, bars), a colour mask (solid, radial, square, horizontal and vertical gradients, or an image), and an embedded centre logo. That's the one I scripted.
- **qr-code-styling** is a TypeScript and JavaScript library under the MIT licence. It runs in the browser and offers dot styles such as square, dots, rounded, extra-rounded and classy, plus separately styled corners, gradients, and a logo. Because it runs client-side, the link never has to leave the visitor's machine.
- **segno** is pure Python with no dependencies, under BSD-3-Clause. It writes SVG and other formats and also handles Micro QR. It's less about decoration and more about getting correct output in many formats.
- **Nayuki's QR Code generator** is a small reference implementation in many languages under the MIT licence. It has no styling, but it's the easiest one to read if you want to understand how the encoding actually works.

None of these need an account, an API, or a subscription, and none of them see what you encode.

## What breaks if you style too hard

A QR code carries redundant data so it can survive damage. There are four error-correction levels, L, M, Q and H, which recover from roughly 7%, 15%, 25% and 30% of damaged data. Styling spends that budget. A logo in the middle destroys the modules underneath it, and the scanner has to rebuild them from the redundancy.

So I tested rather than guessed. I generated codes for one link, covered the centre with a logo of increasing size, and tried to decode each with OpenCV, using its two QR detectors:

- **Level L:** read fine with a logo 20% of the code's width, and failed from 25%.
- **Level M:** read fine at 25%, and failed from 30%.
- **Level H:** read fine up to 40% of the width, which covers about 16% of the area, and failed from 50%.

Those numbers are from one link, one logo and one decoder, so treat them as a shape, not a rule. The practical takeaway is to use level H for anything with a logo, and keep the logo well under the point where it broke, so a bad camera or a scuffed print has room to spare.

Two more things failed hard:

- **Light modules on a dark background** (inverted) couldn't be read by either decoder. Plenty of scanners expect dark on light, so I'd leave dark themes out of the printed code and keep them on the web page next to it.
- **No quiet zone.** With the white margin around the code removed completely, both decoders failed. The spec asks for four modules of margin. One module still worked in my test, but I'd keep the four.

One result surprised me in the opposite direction. A very pale grey code (about 220 out of 255) on white was still read by OpenCV. That does not mean it's fine. OpenCV is more forgiving than a phone camera in dim light, so a decoder passing says little about contrast. Keep it strong.

![Plain, rounded, and gradient-with-logo QR codes for the same link]({{ '/assets/img/qr-styles.png' | relative_url }})

All three codes above decode correctly with both OpenCV detectors. The third uses level H, a gradient and a centre logo.

## What broke (or surprised me)

- **A lookup bug in my own redirector.** The first version looked slugs up with a plain object, so a request for `/constructor` or `/__proto__` found something that is not a link, and the Worker crashed with a 500 instead of answering 404. Harmless, but it was caught only because I wrote tests that asked for names I had not defined. The published version uses `Object.hasOwn`, so only real entries count.
- **A passing decode isn't a pass.** The pale-grey result is the clearest case. I now treat the verify script as a gate that catches clear failures, not as proof, and I still scan one printed copy with a real phone.
- **Styling and reliability pull against each other.** Every flourish costs error-correction budget. Decide how much beauty the code can afford before you design it, not after it fails.

## What I learned

- The valuable part isn't the code, it's the hostname. Whoever owns the short link owns every code printed with it.
- Open-source generators are good enough that there's no need to hand a link to a third party just to get rounded corners.
- Measure the limits on your own link instead of trusting rules of thumb, and then add margin.
- Redirects are cheap to repeat, so don't let anything cache them.

## Next

An in-browser generator page, built on qr-code-styling and served as a static page, so the link is never sent anywhere and the styling can be adjusted visually before downloading. The code is in the `scripts` repo under `cloudflare-qr-redirect`.
