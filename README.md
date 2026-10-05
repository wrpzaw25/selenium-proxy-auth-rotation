# selenium python proxy: Fix the Auth Popup, Rotate Exit IPs, and Keep Headless Chrome Unblocked

Your script works perfectly until you add a proxy. Then Chrome throws up a native authentication dialog that Selenium has no method to click, the page never loads, and after thirty seconds you get a timeout that tells you nothing about what actually went wrong.

That single problem — authenticated HTTP proxies in Selenium — is the reason so many tutorials still tell you to install `selenium-wire`, a library that stopped being maintained and now routinely fails on import with newer `blinker` releases. Copy-pasting a 2021 blog post gets you a stack trace, not a scraper.

This guide covers the three auth patterns that actually run today, how rotation interacts with page loads, and how to keep a headless Chrome job from eating your RAM and your proxy budget along with it.

## The `--proxy-server` flag is fine — right up until the proxy wants a password

If your proxy authenticates by IP whitelist, life is easy. You add your server's IP to the provider's dashboard, then hand Chrome a host and port with no credentials at all:

python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--proxy-server=http://gw.dataimpulse.com:823")

driver = webdriver.Chrome(options=options)
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)
driver.quit()


No extensions, no extra packages, no debugging. The whole session routes through the gateway.

The moment you need a username and password, that simplicity disappears. `ChromeOptions` has no field for proxy credentials, and Chrome does not read them out of the URL. So this line, which looks completely reasonable, does not work:

python
# Chrome ignores the user:pass portion and pops a native auth dialog instead
options.add_argument("--proxy-server=http://user:pass@gw.dataimpulse.com:823")


That dialog is browser chrome, not page DOM. Selenium cannot see it, cannot screenshot it, cannot dismiss it. Every request hangs or returns 407.

Three workarounds exist. Pick based on whether your machine has a static IP.

**1. IP whitelist.** Best option when it's available, because it removes credentials from your code entirely. DataImpulse supports both username/password authentication and IP whitelisting, which means you can run the four-line setup above and skip the rest of this section — 👉 [check the residential proxy setup options on DataImpulse](https://bit.ly/dataimPulse).

**2. A framework that handles auth internally.** SeleniumBase takes a proxy string with credentials as a single argument and manages the handshake for you:

python
from seleniumbase import Driver

# __cr.us targets a US exit; append any country code
driver = Driver(proxy="YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823")
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)
driver.quit()


**3. Credential-injecting extensions.** The classic trick: build a tiny Manifest V3 extension that answers `onAuthRequired` with your username and password, then load it via `--load-extension`. It carried the ecosystem for years. It is now the least reliable of the three, because recent Chrome builds removed the `--load-extension` command-line switch that the trick depends on, and SeleniumBase itself moved to setting proxy auth through CDP mode once CDP is activated.

One more constraint worth knowing before you pick a port: Chromium does not support authenticated SOCKS5. If your residential proxies require credentials, keep them on an HTTP/HTTPS endpoint and reserve SOCKS5 for whitelisted access.

## Rotating versus sticky: the mistake that breaks logins

Here's a detail most tutorials skip. With Selenium, a single `driver.get()` fires dozens or hundreds of HTTP requests — the document, every image, every script, every XHR. If your gateway rotates the exit IP on every request, those requests leave from different addresses. Sites that tie a session to an IP see a user teleporting across three countries mid-page-load, and the anti-bot system responds accordingly.

Rotating per-request is right for `requests` and `httpx`. It is wrong for a browser.

What you want in a Selenium Python proxy setup is **sticky sessions scoped to the browser session**, with rotation happening between jobs. DataImpulse runs sticky sessions in the 1–120 minute range, with a 30-minute default when no interval is specified, and rotating connections that change the exit IP on every new request.

Sticky connections sit in a specific port band — ports 10000 to 20000 on DataImpulse — so sticky and rotating traffic don't collide.

### Two ways to rotate between jobs

**Reassign the proxy at runtime.** Cheaper and faster, because the browser stays warm. The cookie jar does not reset, which matters.

python
base = "http://login:password@gw.dataimpulse.com:823"
opts = {"proxy": {"http": base, "https": base, "no_proxy": "localhost,127.0.0.1"}}

driver = webdriver.Chrome(seleniumwire_options=opts)

# Same browser, new exit country, cookies preserved
uk = "http://login:password@gw.dataimpulse.com:823"
driver.proxy = {"http": uk, "https": uk, "no_proxy": "localhost,127.0.0.1"}
driver.get("https://api.ipify.org")


**Rebuild the driver per job.** Heavier — Chrome startup costs a couple of seconds each time — but you get a guaranteed clean profile, clean local storage, and a fresh exit IP.

python
def fresh_driver():
    return webdriver.Chrome(seleniumwire_options=opts)

for _ in range(3):
    d = fresh_driver()
    d.get("https://api.ipify.org")
    print(d.find_element("tag name", "body").text)
    d.quit()


A workable rule: reassign the proxy while you're collecting anonymous public pages, and rebuild the driver every time you cross an account or login boundary. A cookie that leaks out of one run should not follow you into the next one — fraud systems watch for exactly that mismatch, where the IP changes but the session identifier doesn't.

## Headless runs die of memory, not of proxies

If your job survives an hour and then gets killed, the prime suspect is not the proxy. Selenium Wire buffers every request and response by default, and that buffer grows without limit on long runs. Constrain it:

python
chrome_options = Options()
chrome_options.add_argument("--headless=new")
chrome_options.add_argument("--no-sandbox")
chrome_options.add_argument("--disable-dev-shm-usage")
chrome_options.add_argument("--disable-gpu")
chrome_options.add_argument("--window-size=1920,1080")

driver = webdriver.Chrome(
    options=chrome_options,
    seleniumwire_options={
        "proxy": {"http": base, "https": base, "no_proxy": "localhost,127.0.0.1"},
        "request_storage": "memory",
    },
)

# Capture only the API traffic you actually need
driver.scopes = [r".*/api/.*"]

driver.get("https://example.com/products")
for request in driver.requests:
    print(request.url)

del driver.requests  # clear the buffer between pages


`driver.scopes` is a list of URL regexes. Anything outside them passes through but is never stored. On a scraper that only cares about JSON responses, this cuts resident memory hard.

It also cuts your bill. Every image, font, analytics beacon, and tracking pixel that Chrome downloads is traffic flowing through your proxy, and proxy traffic is billed per gigabyte. Blocking images in headless mode isn't just a speed trick — it's a direct reduction in cost, and it's the difference between a job that costs a few dollars and one that costs thirty.

## Verify the exit IP before blaming the target site

The first diagnostic step, every time, is a one-line check:

python
driver.get("https://httpbin.org/ip")


If the body shows your real IP, the proxy was never applied. Don't debug the target site until this prints something else.

Common failures and what they actually mean:

- **`ERR_PROXY_CONNECTION_FAILED`** — the endpoint is unreachable, or a firewall is blocking the port. Test the same host and port with `curl --proxy` before touching Selenium code.
- **`TimeoutException` on a page that loads fine in `requests`** — Chrome is rendering JavaScript, images, and fonts, so it moves far more data than a text request. Raise the ceiling with `driver.set_page_load_timeout(30)`.
- **The proxy authenticates in `requests` but not in Chrome** — this is the credentials problem from the top of this article, not a proxy problem.
- **The same IP on every run** — either the gateway has rotating disabled at the plan level, or your endpoint is a sticky port that pins one exit IP.
- **Auth worked yesterday, fails today** — you upgraded Chrome. Extension-based auth is the first thing to break when the browser tightens extension loading.

## What it actually costs

Pricing below comes from DataImpulse's current public pricing structure. All four proxy types run on pay-as-you-go billing: no subscription, no monthly reset, and purchased traffic does not expire.

| Proxy type | Entry package | Standard rate | Volume rate | Billing model | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 / 5 GB | $1.00/GB | $0.80/GB at 1 TB+ ($800/TB) | Pay-as-you-go, traffic never expires | [ Grab the $5 residential intro pack](https://bit.ly/dataimPulse) |
| Datacenter | $5 / 10 GB | $0.50/GB | $0.45/GB at 1 TB ($450); $50/100 GB; custom from $2,250 at 5 TB+ | Pay-as-you-go, traffic never expires | [ Start on the datacenter tier](https://bit.ly/dataimPulse) |
| Mobile | $5 / 2.5 GB | $2.00/GB | $1.60/GB at 1 TB ($1,600); $50/25 GB; custom from $8,000 at 5 TB+ | Pay-as-you-go, traffic never expires | [ Compare the mobile proxy tier](https://bit.ly/dataimPulse) |
| Premium Residential | $5 / 1 GB | $5.00/GB | $50/10 GB; custom pricing from $20,000 at 5 TB+ | Pay-as-you-go, traffic never expires | [ See the premium residential pool](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

$5 is the minimum purchase across all four types. There is no free tier and no zero-payment trial — the cheapest way to test the integration is to buy the 5 GB residential pack and burn it on a real job.

Two things about the geographic targeting matter more than the headline rate. Country selection is included in the base price. State, city, ZIP, and ASN targeting on residential plans bill at twice the standard per-GB rate, so a heavily city-targeted job at $1/GB effectively runs at $2/GB. If your Selenium script only ever needs country-level exits, you pay the advertised rate.

For a decision that hinges on total cost rather than rate, the arithmetic is simple: 1 TB of residential traffic costs $800 here versus roughly $3,310 at the rates published by IPRoyal, or around $8/GB at Oxylabs-level enterprise pricing.

## Which proxy type for which Selenium job

The choice is not about which one is "better." It's about how much the target cares where your traffic comes from.

**Datacenter at $0.50/GB** is the correct starting point for a surprising number of Selenium tasks: staging environments, internal dashboards, uptime checks, your own test suites, and public pages that don't inspect ASN. It's fast, it's half the price of residential, and there's no point paying for residential legitimacy you don't need.

**Residential at $1/GB** is what you move to when sites start returning soft blocks — the pages that load but render empty results, or the checkouts that fail silently. Residential IPs come from real consumer connections, so they don't trip the ASN-scoring checks that flag datacenter ranges before the page even renders. Note the timeline: datacenter proxies usually fail loudly with a 403. Residential failures are quieter and harder to notice.

**Mobile at $2/GB** is for targets that fingerprint aggressively and treat carriers differently from consumer broadband — app-driven sites, mobile web flows, and the anti-bot systems that weight mobile IPs more heavily. It costs double, so use it deliberately rather than by default.

**Premium Residential at $5/GB** adds a dedicated account manager and includes all targeting filters at no surcharge, which changes the math if you were going to pay the 2× city/state/ZIP multiplier anyway. It's aimed at workloads where standard residential exits aren't holding up.

## Fine print that changes the budget

**Non-expiring traffic is the real differentiator.** Buy 50 GB, use 10 GB this week and 40 GB across the next two months. Nothing resets on a calendar. If your Selenium scraping is bursty — heavy during a data refresh, idle in between — this is worth more than a few cents per gigabyte, because subscription traffic that expires unused is money you simply lose.

**Refunds are conditional.** Intro plans paid by card come with a 7-day money-back window, provided less than 80% of the traffic has been consumed. Crypto purchases on Intro plans are non-refundable. Read that as: burn less than four of your five intro gigabytes before asking for a refund, and don't pay in crypto if a refund might matter.

**There are no public coupon codes.** DataImpulse's own comparisons lean on the flat $1/GB rate rather than promo cycles, so if you find a "DataImpulse coupon" page promising 40% off, treat it with suspicion. Discounting happens at volume — 1 TB and above — not through codes.

**Authentication is your choice per project.** Username/password or IP whitelist. Whitelisting is what makes the simplest Selenium configuration possible, and it's the option to take when your scraping node has a fixed address.

**No free tier exists.** The smallest commitment is $5, which buys 5 GB of residential, 10 GB of datacenter, or 2.5 GB of mobile traffic — 👉 [start with the $5 pack on DataImpulse](https://bit.ly/dataimPulse) and validate your success rate on the specific targets you care about before scaling up.

## The short version

The Selenium Python proxy problem is really three problems wearing one coat. First, credentials: Chrome can't take them through `ChromeOptions`, so use IP whitelisting, or SeleniumBase, or accept that you're fighting the browser. Second, rotation: per-request rotation breaks page loads, so keep stickiness tied to the browser session and rotate between jobs. Third, bandwidth: a headless browser downloads ten to fifty times more data than a plain HTTP request, so scope your capture, block images, and match the proxy type to how suspicious the target actually is.

Get those three right and the code is about fifteen lines. Get them wrong and you'll spend a weekend reading Stack Overflow threads about a library that hasn't been maintained in years.
