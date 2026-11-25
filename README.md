# Redfin Scraper: How Many Listings You Can Actually Pull, What 1,000 Homes Cost, and Where Coverage Breaks

Redfin has no public API for individual homes. Its Data Center publishes aggregate market statistics by metro, county and ZIP, with no addresses and no MLS numbers, and the Data Center API sits behind a waitlist aimed at enterprise users. Every tool that returns one row per house is either calling the map-search JSON endpoint Redfin's own site loads, or charging you to keep that layer maintained.

Two numbers shape most projects. The search surface stops serving new homes at **350 per region**, and one metered actor documented here delivers rows at **$0.0003 each**. What follows is what Redfin actually exposes, where each route loses coverage, and how to price a run before you commit.

> **Quick pick:** if you want a few hundred listings as JSON or CSV without maintaining parsers, ScraperAPI runs a dedicated Redfin Listing Search endpoint that takes a Redfin search URL and returns structured rows, with a per-URL credit cost you can check before launching a batch. 👉 [See ScraperAPI's plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans: what changes as you climb

ScraperAPI's pricing page runs a free plan plus seven paid tiers: Hobby, Startup, Business, Scaling, Professional, Advanced and Enterprise. Four things move between them — the credit allowance, the thread cap, how precisely you can geotarget, and how far back the analytics dashboard reaches. The published checkpoints:

| Included API credits | Concurrent threads | Geotargeting | Analytics retention | Get started |
| --- | --- | --- | --- | --- |
| 1,000 (free plan) | 5 | Not listed | Not listed | [Create a free ScraperAPI account](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| 100,000 | 20 | US & EU, region-level | Last 30 days | [Look at the 100,000-credit tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| 1,000,000 | 50 | US & EU, region-level | Last 30 days | [Look at the 1,000,000-credit tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| 10,500,000 | 300 | Country-level | Unlimited | [Compare the upper tiers](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| 22,000,000+ (Enterprise) | 500+ | Country-level | Unlimited | [See Enterprise options](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Dollar prices per tier live on the pricing page and shift with billing cycle, so read them there rather than trusting a figure quoted in an article.

Credits matter more than the headline allowance, because a credit isn't a fixed unit of work. One credit buys a standard page request. Sites sitting behind Cloudflare, Datadome or PerimeterX add 10 credits per request when the bypass runs, so the same URL can cost 11. Your dashboard's Domain Cost Estimator returns the exact figure for any URL, and the `max_cost` parameter caps a single scrape so one expensive request can't drain a batch.

For a Redfin job that produces a range instead of a number. 100,000 credits is 100,000 Redfin pages at the standard rate, or roughly 9,000 if every request needs a bypass. The documentation doesn't say which side of that range Redfin lands on, so push one URL through the estimator before sizing a plan. 👉 [Size a credit allowance against your listing volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

Thread caps act as a hard stop rather than a surcharge: exceed your concurrency and the request comes back as a 429 asking you to stay inside the limit, with no extra charge for the attempt. When credits run out, scraping pauses until renewal unless you turn on auto-upgrade (available on Hobby through Professional) or pay-as-you-go (Scaling and above). Annual plans can also renew early to refill.

## What Redfin actually exposes

Search results come back as JSON with a `{}&&` prefix, a guard against JSON hijacking. Strip the prefix and the listings sit under `["payload"]["homes"]`. Detail pages are ordinary HTML, so field extraction is selector work: price from `div[data-rf-test-id='abp-price']/div`, estimated monthly payment from `span.est-monthly-payment`, the address by joining `div.street-address` and `div.cityStateZip`, the remarks block from `#marketing-remarks-scroll p span`, photos from every `img.widenPhoto` src, key facts from `div .keyDetails-value`, and amenities by grouping `.amenity-group ul div.title` and reading the `li/span` items that follow each title.

Rentals take an extra hop. The listing page's `meta[property='og:image']` content carries a 36-character rental id after the `rent/` segment, and that id feeds `stingray/api/v1/rentals/{rental_id}/floorPlans` for floor-plan data. No rental id, no floor plans.

Whether you need a bypass layer depends on which surface you hit. A published Python example sets anti-scraping-protection bypass and a US exit country as required configuration for its Redfin jobs. Two other projects call the map-search JSON straight from the local machine with a normal browser user agent and report that no proxy is needed. That gap can't be settled from documentation alone, so budget for a US IP and a bypass option, then test a single URL before building a pipeline around it.

Sorting is capped by the endpoint itself. Redfin's GIS accepts recommended, price ascending or descending, square footage descending, and lot size descending. `newest` and `days-on-market-asc` are rejected and fail with an explicit input error, so "show me the freshest listings" has to be handled by filtering on your side after the pull.

## How many homes you actually get

The 350-per-region ceiling is the number that reorganizes most projects, and no amount of pagination gets past it.

| Limit | What it does to your pull |
| --- | --- |
| 350 homes per region | Hard ceiling per region per run. More regions is the only way around it |
| Later pages return page one | Raising the page cap adds nothing; duplicates get dropped and you aren't charged twice |
| 20,000 homes per run | Ceiling across all regions combined in a single run |
| Sold listings cover 90 days | Older comparables need a different source |
| ZIP search filters a larger market | Page size 100, offset capped at 50 pages, so a big market can return thin ZIP results |
| `/zipcode/78704` carries no region id | Some scripts want a city or county URL plus a ZIP filter instead |

If you raise the page cap and the row count doesn't move, you've hit the region ceiling, not a pagination bug. An empty dataset isn't a failed run either. One actor records per-URL outcomes in a diagnostics record and reports access failures rather than quietly returning nothing, and another classifies errors as `bad_input`, `no_results`, `search_failed` and `fetch_failed`. Worth wiring those into your scheduler before you trust a zero.

Missing values arrive as null rather than as estimates. A listing without a published price gets `price: null`, and `property_type_name` stays null when the source doesn't map it. Some MLSs suppress days on market, so a null `days_on_market` is a data-supply gap, not a scraper failure. `sold_date` arrives as a millisecond timestamp, and for an active listing it can be the property's previous sale, so filter on `listing_status` before treating it as a comparable.

Search-only tools never open the listing page. That's why they're cheap, and why description, photo and amenity fields are absent from their output.

## What 1,000 homes cost

The one actor here with public per-row pricing charges $0.002 to start a run plus $0.0003 per delivered listing:

| Homes delivered | Per-listing charge | Start fee | Total |
| --- | --- | --- | --- |
| 1,000 | $0.30 | $0.002 | $0.302 |
| 10,000 | $3.00 | $0.002 | $3.002 |
| 20,000 (single-run ceiling) | $6.00 | $0.002 | $6.002 |

The start fee is charged per run regardless of yield, while the per-listing part only applies to rows you actually receive, so a run that comes back thin still costs the start fee. Reaching 20,000 in one run also means submitting enough regions to clear 350 each.

ScrapeHero prices at 10 data credits per record, and its documentation doesn't publish a credit price, so the number can't be converted to dollars from what's available. Its practical difference is scope: it accepts a city page or a single property URL and exports CSV, JSON or Excel across roughly 27 fields, including description, image URLs and the Redfin estimate — the detail-page fields that search-only actors skip.

ScraperAPI bills in credits per request, which maps onto Redfin like this: 1,000 Redfin search pages cost 1,000 credits at the standard rate, or 11,000 if every one of them triggers a bot-protection bypass. Pair that with the Domain Cost Estimator for the real figure.

## Which route fits the job

- **A few hundred active listings, one metro, one-off:** a metered actor at roughly $0.30 per 1,000 is the least work and the least money.
- **You need remarks, photos or amenities:** those live on detail pages, so every listing becomes a second request. Budget accordingly.
- **You need sold comparables:** the window is 90 days, and the sale-date column needs a status check before you trust it.
- **You need rentals:** most ready-made actors skip them outright. The floor-plan route through the rental id is the documented path.
- **You need recurring monitoring across many regions:** per-run start fees and the 350 ceiling bite at scale, and an async structured endpoint with webhook delivery fits better than ad-hoc scripts. ScraperAPI's Redfin Listing Search endpoint accepts a single Redfin search URL or an array of them, with `country_code` geotargeting, a `tld` selector, and either a webhook callback or a `statusUrl` to poll. 👉 [Check the plan that covers your monthly page volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

One vendor in this space argues that scraping public MLS data without logging in is generally lawful in the US, citing hiQ v. LinkedIn (Ninth Circuit, 18 April 2022), while telling users to follow local law and Redfin's terms. That's a tool seller's position, not a legal opinion, and it doesn't bind Redfin's terms of service. Treat the terms and your local rules as the constraints you actually have to satisfy.

## FAQ

**Why doesn't raising maxPages return more homes?**
Redfin serves page one again for later offsets once a region is exhausted. Tools dedupe the repeats, so a bigger page cap changes nothing past 350 homes per region. Add regions instead.

**Why are ZIP results thinner than the neighborhood next door?**
A ZIP search asks Redfin for a larger market response and filters it down. With a page size of 100 and offsets capped at 50 pages, a big market can run out before your ZIP is fully covered. Pull the city or county and filter by ZIP yourself when completeness matters.

**Why is days_on_market null for a listing that clearly just appeared?**
Some MLSs withhold it. The scraper writes null rather than guessing, which is the right behavior but means you can't sort by freshness on those records.

**Why does one property have a sold date when it's currently for sale?**
That field carries the previous sale of the property. Check `listing_status` first.

## Before you buy

Match the tool to the shape of the job. A one-off pull of a few hundred active listings in a single metro is a $0.30 problem. Recurring coverage across dozens of regions is an infrastructure purchase, and the per-row price stops being the deciding factor.

Confirm three things before you pay anyone: whether the plan bills per successful request or per attempt, whether US geotargeting is included at the tier you're considering (region-level on ScraperAPI's lower paid tiers, country-level higher up), and whether your target pages need JavaScript rendering or a bypass that adds credits on top of the base rate. 👉 [Start with ScraperAPI's Redfin endpoint and current plans](https://www.scraperapi.com/pricing?fp_ref=coupons)
