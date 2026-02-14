# Anti-Bot Detection & Fallback

When crawling sites protected by anti-bot systems (Akamai, Cloudflare, PerimeterX, DataDome, Imperva, etc.), requests often get blocked with CAPTCHAs, 403 responses, or empty pages. Crawl4AI provides a layered retry and fallback system that automatically detects blocking and escalates through multiple strategies until content is retrieved.

## How Detection Works

After each crawl attempt, Crawl4AI inspects the HTTP status code and HTML content for known anti-bot signals:

- **HTTP 403/429** with short or empty response bodies
- **Challenge pages** — Cloudflare "Just a moment", Akamai "Access Denied", PerimeterX block pages
- **CAPTCHA injection** — reCAPTCHA, hCaptcha, or vendor-specific challenges on otherwise empty pages
- **Firewall blocks** — Imperva/Incapsula resource iframes, Sucuri firewall pages, Cloudflare error codes

Detection uses structural HTML markers (specific element IDs, script sources, form actions) rather than generic keywords to minimize false positives. A normal page that happens to mention "CAPTCHA" or "Cloudflare" in its content will not be flagged.

## Configuration Options

All anti-bot retry options live on `CrawlerRunConfig`:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `max_retries` | `int` | `0` | Number of retry rounds when blocking is detected. `0` = no retries. |
| `fallback_proxy_configs` | `list[ProxyConfig]` | `[]` | List of fallback proxies tried in order within each retry round. |
| `fallback_fetch_function` | `async (str) -> str` | `None` | Async function called as last resort. Takes URL, returns raw HTML. |

And on `ProxyConfig`:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `is_fallback` | `bool` | `False` | When `True`, this proxy is skipped on the first attempt and only activated after blocking is detected. |

## Escalation Chain

Each retry round tries the main proxy first, then each fallback proxy in order. If all rounds are exhausted and the page is still blocked, the fallback fetch function is called as a last resort.

```
For each round (1 + max_retries rounds):
    1. Try with main proxy_config (or no proxy if is_fallback=True on first round)
    2. If blocked → try fallback_proxy_configs[0]
    3. If blocked → try fallback_proxy_configs[1]
    4. ... continue through all fallback proxies
    5. If any attempt succeeds → done

If all rounds exhausted and still blocked:
    6. Call fallback_fetch_function(url) → process returned HTML
```

Worst-case attempts before the fetch function: `(1 + max_retries) x (1 + len(fallback_proxy_configs))`

## Usage Examples

### Simple Retry (No Proxy)

Retry the crawl up to 3 times when blocking is detected. Useful when blocks are intermittent or IP-based.

```python
from crawl4ai import AsyncWebCrawler
from crawl4ai.async_configs import BrowserConfig, CrawlerRunConfig

async with AsyncWebCrawler(config=BrowserConfig(headless=True)) as crawler:
    result = await crawler.arun(
        url="https://example.com",
        config=CrawlerRunConfig(max_retries=3),
    )
```

### Proxy as Fallback Only

Use `is_fallback=True` to skip the proxy on the first attempt. If the site doesn't block you, no proxy credits are consumed. If it does, the proxy activates on retry.

```python
from crawl4ai.async_configs import ProxyConfig

config = CrawlerRunConfig(
    max_retries=2,
    proxy_config=ProxyConfig(
        server="http://proxy.example.com:8080",
        username="user",
        password="pass",
        is_fallback=True,  # Only used when blocking is detected
    ),
)
```

### Fallback Proxy List

Try a cheaper proxy first, escalate to a premium proxy if it fails. Both are tried within each retry round.

```python
config = CrawlerRunConfig(
    max_retries=2,
    proxy_config=ProxyConfig(
        server="http://datacenter-proxy.example.com:8080",
        username="user",
        password="pass",
    ),
    fallback_proxy_configs=[
        ProxyConfig(
            server="http://residential-proxy.example.com:9090",
            username="user",
            password="pass",
        ),
    ],
)
```

With this setup, each round tries the datacenter proxy first, then the residential proxy. With `max_retries=2`, worst case is 3 rounds x 2 proxies = 6 attempts.

### Fallback Fetch Function

When all browser-based attempts fail, call a custom async function as a last resort. This function receives the URL and must return raw HTML as a string. The returned HTML is processed through the normal pipeline (markdown generation, extraction, etc.).

This is useful when you have access to a scraping API, a pre-fetched cache, or any other source of HTML.

```python
import aiohttp

async def my_scraping_api(url: str) -> str:
    """Fetch HTML via an external scraping API."""
    async with aiohttp.ClientSession() as session:
        async with session.get(
            "https://api.my-scraping-service.com/fetch",
            params={"url": url, "format": "html"},
            headers={"Authorization": "Bearer MY_TOKEN"},
        ) as resp:
            if resp.status == 200:
                return await resp.text()
            raise RuntimeError(f"API error: {resp.status}")

config = CrawlerRunConfig(
    max_retries=1,
    fallback_fetch_function=my_scraping_api,
)
```

The function can do anything — call an API, read from a database, return cached HTML, or make a simple HTTP request with a different library. Crawl4AI does not care how the HTML is obtained.

### Full Escalation (All Features Combined)

This example combines every layer: stealth mode, a fallback proxy that only activates when blocked, a list of escalation proxies tried each round, retries, and a final fetch function.

```python
import aiohttp
from crawl4ai import AsyncWebCrawler
from crawl4ai.async_configs import BrowserConfig, CrawlerRunConfig, ProxyConfig

# Last-resort: fetch HTML via an external service
async def external_fetch(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.post(
            "https://api.my-service.com/scrape",
            json={"url": url, "render_js": True},
            headers={"Authorization": "Bearer MY_TOKEN"},
        ) as resp:
            return await resp.text()

browser_config = BrowserConfig(
    headless=True,
    enable_stealth=True,
)

crawl_config = CrawlerRunConfig(
    magic=True,
    wait_until="load",
    max_retries=2,

    # Primary proxy — is_fallback=True means first attempt runs without it
    proxy_config=ProxyConfig(
        server="http://datacenter-proxy.example.com:8080",
        username="user",
        password="pass",
        is_fallback=True,
    ),

    # Fallback proxies — tried in order after main proxy fails each round
    fallback_proxy_configs=[
        ProxyConfig(
            server="http://residential-proxy.example.com:9090",
            username="user",
            password="pass",
        ),
    ],

    # Last resort — called after all retries and proxies are exhausted
    fallback_fetch_function=external_fetch,
)

async with AsyncWebCrawler(config=browser_config) as crawler:
    result = await crawler.arun(
        url="https://protected-site.com/products",
        config=crawl_config,
    )

    if result.success:
        print(f"Got {len(result.markdown.raw_markdown)} chars of markdown")
    else:
        print(f"All attempts failed: {result.error_message}")
```

**What happens step by step:**

| Round | Attempt | What runs |
|---|---|---|
| 1 | 1 | No proxy (is_fallback skips it) — blocked |
| 1 | 2 | Residential fallback proxy — blocked (bad IP) |
| 2 | 1 | Datacenter proxy activated — blocked |
| 2 | 2 | Residential fallback proxy — blocked |
| 3 | 1 | Datacenter proxy — blocked |
| 3 | 2 | Residential fallback proxy — blocked |
| - | - | `external_fetch(url)` called — returns HTML |

That's up to 6 browser attempts + 1 function call before giving up.

## Tips

- **Start with `max_retries=0`** and a `fallback_fetch_function` if you just want a safety net without burning time on retries.
- **Use `is_fallback=True`** on your proxy to avoid consuming proxy credits on sites that don't need them.
- **Order fallback proxies cheapest-first** — datacenter proxies before residential, residential before premium.
- **Combine with stealth mode** — `BrowserConfig(enable_stealth=True)` and `CrawlerRunConfig(magic=True)` reduce the chance of being blocked in the first place.
- **`wait_until="load"`** is important for anti-bot sites — the default `domcontentloaded` can return before the anti-bot sensor finishes.
- **You don't need a primary proxy to use fallback proxies.** If you skip `proxy_config` and only pass `fallback_proxy_configs`, the first attempt each round runs with no proxy. This is useful when you want to try direct access first and only escalate to proxies if blocked:
    ```python
    config = CrawlerRunConfig(
        max_retries=1,
        fallback_proxy_configs=[proxy_A, proxy_B],
    )
    # Round 1: no proxy → proxy_A → proxy_B
    # Round 2: no proxy → proxy_A → proxy_B
    ```

## See Also

- [Proxy & Security](proxy-security.md) — Proxy setup, authentication, and rotation
- [Undetected Browser](undetected-browser.md) — Stealth mode and browser fingerprint evasion
- [Session Management](session-management.md) — Maintaining sessions across requests
