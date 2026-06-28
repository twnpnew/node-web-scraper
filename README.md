# How to Scrape a Website with Node.js Without Getting Blocked: Step-by-Step Code, Common Anti-Bot Traps, and Which Tools Actually Hold Up at Scale

If you've ever written a perfectly clean `axios.get()` call in Node.js, run it once, watched it return a 200 with full HTML, and then run it again twenty minutes later only to get hit with a 403 — you already know the real problem. It's never the scraping logic. It's staying invisible long enough to finish the job.

This is the part most Node.js tutorials skip. They'll walk you through `cheerio`, show you how to grab a `<div class="price">`, and call it done. Then you deploy it, run it on a schedule, and the target site quietly decides your IP looks like a bot. No error message that makes sense. Just empty responses, CAPTCHA pages, or a 429 you didn't see coming.

Let's actually deal with that.

## Why Node.js Scrapers Get Blocked in the First Place

Websites aren't randomly blocking people. They're pattern-matching. A few signals get you flagged almost instantly:

- **Too many requests from one IP, too fast.** A real visitor doesn't load 200 product pages in 40 seconds.

- **Missing or generic headers.** A `User-Agent` like `axios/1.6.0` is basically a flashing sign that says "this is a script."

- **No cookies, no referrer, no session continuity.** Real browsers carry state between page loads. A bare HTTP client often doesn't.

- **Predictable timing.** Hitting a page every exactly 2.000 seconds is a dead giveaway — humans are messy.

- **JavaScript-rendered content that never finishes "rendering"** because you're using a plain HTTP client instead of a browser engine, and the site notices you never executed any of its scripts.

- **Bot-protection layers** like Cloudflare, DataDome, or PerimeterX, which actively fingerprint TLS handshakes, browser properties, and even mouse movement before deciding whether to serve real content.

None of this is exotic. It's standard practice on anything past a personal blog — e-commerce sites, job boards, real estate listings, social platforms.

## Step 1: Start Simple — Static HTML with Axios + Cheerio

For sites that don't lean on JavaScript to render content, this combo is still the fastest path:

javascript

const axios = require('axios');

const cheerio = require('cheerio');

async function scrapePage(url) {

const { data } = await axios.get(url, {

headers: {

'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36',

'Accept-Language': 'en-US,en;q=0.9',

},

});

const $ = cheerio.load(data);

const title = $('h1').first().text().trim();

return title;

}



This works great on day one. The problem is day thirty, once you're running this on a schedule against a site that's started watching your IP.

## Step 2: When the Page Needs JavaScript — Puppeteer or Playwright

If the data you want only appears after the page runs its own scripts (infinite scroll, lazy-loaded prices, single-page apps), `cheerio` can't help — it doesn't execute anything, it just parses static markup. That's where a headless browser comes in:

javascript

const puppeteer = require('puppeteer');

async function scrapeDynamic(url) {

const browser = await puppeteer.launch({ headless: 'new' });

const page = await browser.newPage();

await page.setUserAgent('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36');

await page.goto(url, { waitUntil: 'networkidle2' });

const data = await page.evaluate(() => {

return document.querySelector('.price')?.innerText;

});

await browser.close();

return data;

}



Playwright works almost identically and has slightly better cross-browser spoofing options if you want to rotate between Chromium, Firefox, and WebKit fingerprints. The tradeoff: headless Chromium can itself leak tells like `navigator.webdriver` being `true`, which some anti-bot systems check directly — so a raw Puppeteer instance isn't automatically "stealthy" just because it's a real browser engine.

## Step 3: The Part That Actually Stops the Blocking

This is where most guides get vague, so here's the concrete list, roughly in order of how much it matters:

1. **Rotate IPs through proxies — ideally residential, not datacenter.** Datacenter IP ranges are well-known and get pre-flagged by a lot of anti-bot tooling. Residential or mobile proxy pools look like ordinary home traffic.

2. **Rotate User-Agents.** Keep a pool of current, real browser UA strings and cycle through them — don't reuse one stale UA across thousands of requests.

3. **Randomize delays between requests.** A `setTimeout` with a random range (say, 1.5–4 seconds) is more convincing than a fixed interval.

4. **Respect and check `robots.txt`** — not just out of courtesy, but because ignoring obvious crawl-rate signals is itself a blocking trigger.

5. **Handle retries with backoff.** A 429 or 503 isn't necessarily permanent — back off, wait, retry with a different IP/UA combination.

6. **Solve or route around CAPTCHAs when they appear**, rather than letting your scraper silently fail.

7. **For sites behind Cloudflare/DataDome/PerimeterX specifically**, plain proxy rotation often isn't enough — these systems fingerprint TLS handshakes and browser behavior, not just IP reputation.

You *can* build all of this yourself. People do. But it's genuinely a second engineering project sitting on top of your actual scraper — proxy pool management, UA databases, retry logic, CAPTCHA-solving integration, and constant maintenance as anti-bot systems update their detection.

## Where a Scraping API Fits In

This is the point where a lot of Node.js developers stop building their own anti-blocking stack and instead route requests through a scraping API that already handles proxy rotation, headless rendering, and CAPTCHA bypass behind a single endpoint. **ScraperAPI** is one of the more widely used options for exactly this — you send it a URL, it handles the IP rotation, browser rendering, and retry logic on its side, and hands you back the HTML.

In Node.js, it collapses down to something like this:

javascript

const axios = require('axios');

async function scrapeWithAPI(targetUrl) {

const response = await axios.get('https://api.scraperapi.com', {

params: {

api_key: 'YOUR_API_KEY',

url: targetUrl,

render: true, // set true for JS-heavy pages

},

});

return response.data;

}



That single `render: true` flag is effectively replacing your entire Puppeteer + proxy-rotation + retry stack for that request.

If you want to try this without committing to anything, 👉 [grab 1,000 free API credits on ScraperAPI here](https://www.scraperapi.com/?fp_ref=coupons) and run it against a real target before deciding whether it's worth paying for.

## What It Actually Costs

ScraperAPI bills by credits rather than flat bandwidth, and the cost per request scales with how hard the target site is to scrape. A standard page costs 1 credit; tougher targets like Amazon cost more, and sites sitting behind heavy bot protection (Cloudflare, DataDome, PerimeterX) add credits on top when that protection has to be bypassed. Here's the current full plan lineup:

| Plan | Monthly Price | Annual Price | Credits / Month | Concurrent Threads | Notable Limits | Get Started |

|---|---|---|---|---|---|---|

| Free | $0 | — | 1,000 credits | 5 | Good for testing only | 👉 [Start free on ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) |

| Hobby | $49/mo | $44/mo | 100,000 credits | 20 | US & EU regions only | 👉 [See Hobby plan pricing](https://www.scraperapi.com/?fp_ref=coupons) |

| Startup | $149/mo | $134/mo | 1,000,000 credits | 50 | US & EU regions only | 👉 [See Startup plan pricing](https://www.scraperapi.com/?fp_ref=coupons) |

| Business | $299/mo | $269/mo | 3,000,000 credits | 100 | Country-level geotargeting | 👉 [See Business plan pricing](https://www.scraperapi.com/?fp_ref=coupons) |

| Scaling | $475/mo | $427/mo | 5,000,000 credits | 200 | Country-level geotargeting | 👉 [See Scaling plan pricing](https://www.scraperapi.com/?fp_ref=coupons) |

| Enterprise | Custom | Custom | Custom (3M+) | Custom | Dedicated support, custom features | 👉 [Contact sales via ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) |

> Note: ScraperAPI's pricing page doesn't expose stable per-plan product IDs in its URL structure, so each link above routes through the same tracked entry link rather than a fabricated deep link — clicking through still lands you on the live, current pricing page where you can select any tier.

A few things worth knowing before you pick a tier:

- The **free tier** (1,000 credits, 5 concurrent threads) is genuinely enough to validate that your scraping logic works against your actual target before you pay anything.

- **Annual billing** saves roughly 10% across every paid tier — worth it only if you're confident you'll use the service for the full year.

- If you blow through your monthly credits on Hobby, Startup, or Business, you can upgrade tiers mid-cycle; Scaling, Business-equivalent, and Enterprise customers get pay-as-you-go overage instead of a hard cutoff.

- Cost scales with *difficulty*, not just volume — scraping 100,000 plain blog pages is cheap; scraping 100,000 Amazon listings or pages behind Cloudflare costs noticeably more credits per request.

## DIY vs. API: The Honest Tradeoff

It's worth being straight about this instead of pretending one approach is universally right.

**Build it yourself (proxies + Puppeteer + retry logic) if:**

- You're scraping a handful of sites with light or no anti-bot protection

- You want full control over headers, fingerprinting, and request timing

- You have the engineering time to maintain the stack as sites change their defenses

**Use a scraping API if:**

- You're hitting sites with serious bot protection (Cloudflare, DataDome) and don't want to chase their updates

- Your team's time is better spent on the data pipeline than on proxy infrastructure

- You're scraping at a scale where occasional blocks translate into real lost data, not just an annoying retry

A decent rule of thumb that keeps coming up in developer write-ups: once you're spending more than a few hours a week just debugging blocks instead of improving your actual data pipeline, the API starts paying for itself in time alone — separate from whatever it costs in credits.

## A Realistic Node.js Setup That Combines Both

In practice, a lot of production scrapers end up layering things rather than picking one extreme:

javascript

const axios = require('axios');

const cheerio = require('cheerio');

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function scrapeWithRetry(url, attempt = 1) {

try {

const response = await axios.get('https://api.scraperapi.com', {

params: {

api_key: process.env.SCRAPERAPI_KEY,

url,

render: true,

},

timeout: 30000,

});

return cheerio.load(response.data);

} catch (err) {

if (attempt < 3) {

const backoff = attempt * 2000 + Math.random() * 1000;

await delay(backoff);

return scrapeWithRetry(url, attempt + 1);

}

throw new Error(`Failed to scrape ${url} after 3 attempts: ${err.message}`);

}

}



This keeps your own code simple — randomized backoff and retries on your end, while the proxy rotation, header management, and rendering complexity get handled upstream by the API. It's a pattern that holds up whether you're scraping ten pages a day or ten thousand.

## Quick Answers to Common Questions

**Is it legal to scrape a website with Node.js?**

Scraping publicly accessible data is generally treated differently from scraping data behind a login wall or violating a site's terms of service — but this varies by jurisdiction and by what you do with the data. Checking `robots.txt` and a site's terms before scraping at scale is good practice, not just politeness.

**Why does my scraper work locally but get blocked when deployed?**

Usually because your local IP hasn't been flagged yet, but your server's IP (especially cloud-hosted) is part of a known datacenter range that gets pre-emptively rate-limited or blocked by a lot of anti-bot systems.

**Do I need Puppeteer for every site?**

No — only for sites where the data you need is injected by JavaScript after page load. If `view-source:` on the page already shows the data you want, a plain HTTP request with `cheerio` is faster and cheaper.

**What's the single most common rookie mistake?**

Sending requests with no randomization at all — same User-Agent, same interval, same headers, every single request. That's the easiest pattern in the world to detect.

---

Getting blocked isn't a sign you're doing something wrong with your scraping logic — it's just the site doing its job. The fix is less about clever code and more about making your traffic look unremarkable: rotate what needs rotating, slow down where it matters, and offload the parts that are genuinely a full-time infrastructure problem if a managed service already solves them reliably. Whether you build the anti-blocking layer yourself or route through something like 👉 [ScraperAPI's free tier](https://www.scraperapi.com/?fp_ref=coupons) to test the difference, the actual scraping code in Node.js stays exactly as simple as it should be.
