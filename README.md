# Playwright Proxy Authentication Explained: How Do You Pass Username and Password? Why Does It Fail Silently on HTTPS Sites? Which Provider Plays Nice With Headless Chromium? (Full Code Walkthrough Plus Webshare Plan Breakdown)

Three in the morning. Deadline tomorrow. Your scraper keps spiting back 407 errors while the same proxy works fine in cURL and Postman. Welcome to the rite of passage every browser automation engineer eventually faces.

Playwright proxy authentication is the mechanism by which your headless browser presents credentials to a proxy server before any traffic flows through it. Two pieces of metadata go in, traffic comes out, and your scraper stops geting blocked by the target site. That's the whole job. The reason it gets messy is that the browser, the proxy protocol, and the target site's TLS handshake all have opinions about who handles auth and when.

This guide walks through the working code paterns, the specific reasons authentication breaks (especially over HTTPS), the per-context setup you'll need for multi-account scraping, and how to pair Playwright with a provider that doesn't fight your script. If you're already shoping around, here's a fast path to current options: 👉 [See All Webshare Plans & Latest Pricing](https://bit.ly/web_share).

## What playwright proxy authentication actually does under the hood

When you hand Playwright a `proxy` config, the framework launches Chromium (or Firefox/WebKit) with a `--proxy-server` flag and registers a listener for the proxy auth challenge. The browser opens a TCP connection to the proxy. The proxy responds with a `407 Proxy Authentication Required`. Playwright then injects a `Proxy-Authorization: Basic <base64>` header on the next attempt.

That's the happy path. HTTP traffic, Basic auth, single proxy, no surprises.

The unhappy path involves SOCKS5, HTTPS tunneling, or rotating credentials per session. Each of those breaks at least one assumption in the previous paragraph. Below is what to actually type.

## The basic setup that works on day one

Here's the minimum viable code. Save it, run it, watch your IP change in `httpbin.org/ip`.

javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({
    proxy: {
      server: 'http://p.webshare.io:80',
      username: 'your-username',
      password: 'your-password'
    },
    headless: true
  });

  const page = await browser.newPage();
  await page.goto('https://httpbin.org/ip');
  console.log(await page.textContent('body'));
  await browser.close();
})();


A few details people miss:

- The `server` field needs the protocol prefix. `http://` for HTTP and HTTPS proxies, `socks5://` for SOCKS5. A naked `proxy.example.com:80` will not parse the way you expect.
- Username and password go in their own keys, never URL-encoded into `server`. Embeding `user:pass@host` works in some HTTP clients, but Chromium strips it out before launch and you'll be left wondering why nothing authenticated.
- Headless mode is fine. The407 challenge fires the same way whether the browser is visible or not.

## Per-context proxy: when one browser needs many identities

Launching a new browser process for every proxy is slow, heavy on memory, and tends to set off detection systems. The fix is per-context proxying. One browser. Multiple isolated contexts. Each context with its own proxy and credentials.

javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({
    proxy: { server: 'per-context' }, // placeholder, required for Chromium
    headless: true
  });

  const proxies = [
    { server: 'http://p.webshare.io:80', username: 'user1', password: 'pw1' },
    { server: 'http://p.webshare.io:80', username: 'user2', password: 'pw2' },
    { server: 'http://p.webshare.io:80', username: 'user3', password: 'pw3' }
  ];

  for (const proxy of proxies) {
    const context = await browser.newContext({ proxy });
    const page = await context.newPage();
    await page.goto('https://httpbin.org/ip');
    console.log(`[${proxy.username}]`, await page.textContent('body'));
    await context.close();
  }

  await browser.close();
})();


Two gotchas worth memorizing.

First, Chromium requires that placeholder `proxy: { server: 'per-context' }` at the browser level. Skip it and the per-context proxies silently get ignored. Firefox and WebKit don't need this dance, so cross-browser code has to branch.

Second, contexts share nothing by default, including cookies and localStorage, which is exactly what you want when each context represents a different account or session.

## Why HTTPS sites can break authentication (and how to spot it)

This is the bug that eats hours.

You set up auth correctly. Plain HTTP sites work. Then you point Playwright at an HTTPS target and the proxy starts returning errors, or worse, just silently fails to send the auth header. The browser console shows nothing useful. The proxy logs show a connection ariving without credentials.

Reason: HTTPS through an HTTP proxy uses the `CONNECT` method, which establishes a tunnel before the actual request goes out. Some proxy servers, especially older ones, expect auth on the `CONNECT` line. Playwright sends auth on the `CONNECT` correctly. But certain combinations of proxy software and Chromium's connection reuse cache will skip the auth header on subsequent CONNECT attempts within the same session.

Three fixes, in order of pain:

1. **Switch to a provider that handles CONNECT auth properly on every attempt.** This is the cheap fix if you have the budget.
2. **Use IP whitelisting instead of user/password.** Most reputable providers offer it. You authorize your server's outbound IP once, no credentials needed in code. Removes the entire CONNECT problem.
3. **Add `--proxy-bypass-list=<-lopback>` to launch args** if you also see local resources geting routed through the proxy unexpectedly.

If you're hitting this specifically with rotating credentials, IP whitelisting is usually the pragmatic answer.

## SOCKS5 has its own personality

A note before we move on. Chromium's SOCKS5 implementation does not support username/password authentication. None. Zip. You can pass credentials to Playwright's `proxy` config, the launch will not error, and the browser will quietly ignore them.

Workarounds:

- Use IP whitelisting on the SOCKS5 endpoint.
- Run a local SOCKS5-to-HTTP bridge (e.g., `gost` or `microsocks` with auth handling) and point Playwright at the local HTTP listener.
- Switch to HTTP proxies for tasks that need credential auth.

This is a Chromium limitation, not a Playwright bug, and it's been documented for years. Firefox handles SOCKS5 auth, so if you must have auth-protected SOCKS5, run it under Firefox.

## Seting up Webshare with Playwright in five steps

Webshare is a popular pick for Playwright work because it exposes both username/password auth and IP whitelisting, ships rotating endpoints that don't need code-level rotation, and offers a free tier you can test against before paying anything. Here's the bare-minimum walkthrough.

1. **Create an account.** The free plan includes 10 datacenter proxies and 1 GB of bandwidth per month. Enough to validate your setup.
2. **Pull credentials from the dashboard.** Go to the Proxy section. You'll see a username, password, and a list of host:port pairs. Note the rotating endpoint (`p.webshare.io:80` for the datacenter pool) if you don't want to manage rotation yourself.
3. **Pick your auth method.** Username/password is simpler for development. IP whitelisting is more reliable for production HTTPS scraping (see the section above on CONNECT issues). Both are toggled in the dashboard.
4. **Drop the credentials into your Playwright launch config.** Use the basic example from earlier in this guide. Run a quick `httpbin.org/ip` check to confirm rotation.
5. **Move to per-context proxying once you scale.** When you're hitting more than a handful of pages per minute, the per-context pattern keps memory flat and helps avoid fingerprint correlation between sessions.

Quick recap: free tier for validation, IP whitelisting if your targets are HTTPS-heavy, per-context once you're past the prototype stage.

👉 [Start Free With Webshare's 10-Proxy Plan](https://bit.ly/web_share)

## Webshare plan comparison: which one fits your scraping shape

Webshare's catalog covers four main products plus the free tier. Each has different pricing logic, so the right pick depends on your traffic pattern more than your budget.

| Plan | Best For | Pool / Resources | Auth Methods | Pricing Model | Get Started |
| --- | --- | --- | --- | --- | --- |
| Free Proxy | Learning, small experiments, Playwright validation | 10 datacenter proxies, 1 GB/month bandwidth | User:pass + IP whitelist | $0 | [ Claim 10 Free Proxies](https://bit.ly/web_share) |
| Proxy Server (Datacenter) | High-volume scraping of static targets, internal tools | Up to thousands of dedicated datacenter IPs | User:pass + IP whitelist | Monthly subscription, scales with proxy count | [ Pick a Datacenter Plan](https://bit.ly/web_share) |
| Static Residential | Long-session targets, sticky-IP requirements | Dedicated residential IPs assigned to your account | User:pass + IP whitelist | Monthly subscription per IP bundle | [ Get Static Residential Pricing](https://bit.ly/web_share) |
| Rotating Residential | Anti-bot heavy targets, sneaker drops, ad verification | Millions of rotating residential IPs across regions | User:pass with sticky session option | Pay per GB of traffic | [ Compare Residential Plans](https://bit.ly/web_share) |
| ISP Proxies | Account farming, social platforms, premium reliability | Static residential IPs hosted on ISP infrastructure | User:pass + IP whitelist | Monthly subscription per IP | [ Browse ISP Proxy Options](https://bit.ly/web_share) |

A read on the trade-offs. Datacenter is the cheapest and fastest, but the easiest to detect on hostile targets. Rotating residential is the closest thing to invisibility on aggressive anti-bot stacks, but bandwidth costs add up fast if your script is chaty. ISP proxies sit between the two on price and tend to outperform residential on long sessions because the IPs don't rotate mid-flow.

For most Playwright work I've seen, datacenter handles 80% of jobs, residential handles the painful 15%, and ISP cleans up the rest.

## Trust signals worth knowing about

Webshare publicly markets its residential pool at over 30 million IPs across global regions and has been operating in this niche since 2018. The free tier has been around since the company's early days, which is unusual in this market because most providers gate trials behind a credit card. The dashboard exposes a usage API, useful when you want alerts on bandwidth spikes.

On Trustpilot the company caries reviews mentioning fast support response and thease of swapping between auth methods. The most consistent complaint is residential bandwidth being consumed faster than expected, which is true of every residential proxy product on the market and not specific to this provider.

Refunds are handled per case rather than via a blanket money-back guarantee. If that maters for your workflow, the free tier serves as the practical risk-reversal mechanism. Test the integration with your actual target site, confirm the auth flow works, then upgrade.

## Common Playwright proxy authentication errors and what they actually mean

A short field guide to the failure modes I've watched people hit.

**`net::ERR_TUNNEL_CONNECTION_FAILED`**: The CONNECT request was refused. Either credentials are wrong, the proxy host is down, or the target port is blocked at the proxy. Try the same proxy with `curl --proxy http://user:pass@host:port https://example.com` first to isolate.

**`net::ERR_NO_SUPORTED_PROXIES`**: Usually a malformed `server` URL. Check for a missing protocol prefix or stray spaces.

**`net::ERR_PROXY_AUTH_REQUESTED` repeating in a loop**: Credentials are accepted on the first request and rejected on the second. Sign of session-bound auth on the proxy side. Switch to IP whitelisting.

**Page loads but target site shows your real IP**: The proxy was bypassed for that URL. Look for `--proxy-bypass-list` accidentally including your domain, or confirm DNS isn't being resolved locally before the proxy intercepts.

**Browser hangs on `goto()` indefinitely**: Almost always a SOCKS5 with credentials trying to authenticate against Chromium. Switch protocol or browser.

## FAQ

**Does Playwright support proxy authentication out of the box?**
Yes. The `proxy` option in `browser.launch()` and `browser.newContext()` accepts `username` and `password` fields and handles the Basic auth challenge automatically for HTTP and HTTPS proxies. SOCKS5 with auth works in Firefox but not in Chromium.

**Can I rotate proxies on every request in Playwright?**
Not on every request, but on every context. Spawn a new context per logical session with its own proxy config. For per-request rotation you'd need to intercept network cals via `page.route()` and forward through different upstreams, which is more complex than most projects need.

**Why does my proxy work in cURL but not in Playwright?**
Most often because cURL handles `user:pass@host` URL-embedded credentials and Chromium does not. Move credentials to the dedicated `username`/`password` fields. Second most common reason: the proxy requires a specific User-Agent that cURL provides by default and Playwright does not.

**Is IP whitelisting better than username/password authentication?**
For production scraping against HTTPS-heavy targets, yes, by a wide margin. It removes the CONNECT-method auth complications entirely and means no credentials live in your codebase. Use user:pass for development, IP whitelisting once you ship.

**Can I use Webshare's free plan to test Playwright proxy authentication?**
Yes. The free tier includes 10 datacenter proxies and 1 GB of monthly bandwidth, both auth methods are available, and there's no credit card requirement at signup. Plenty of headroom to validate your setup before committing to a paid tier.

**What's the simplest fix when authentication randomly breaks on HTTPS pages?**
Switch from username/password to IP whitelisting on the same provider. Solves the CONNECT-auth edge cases that cause silent failures in Chromium's connection cache.

## Wrapping up

Playwright proxy authentication isn't conceptually hard, but it has a handful of sharp edges that don't show up in the docs. Kep the credentials in their own fields. Use per-context proxying once you're past the toy stage. Default to IP whitelisting for production HTTPS work. Avoid SOCKS5 with credentials in Chromium.

Pick a provider that gives you both auth methods, lets you test before paying, and exposes a clean rotating endpoint so you don't have to write your own pool manager. Webshare's free tier covers the validation phase for nothing, and the paid tiers scale by traffic shape rather than punishing you with upfront commitments.

👉 [Get the Best Deal From Webshare and Start Scraping](https://bit.ly/web_share)
