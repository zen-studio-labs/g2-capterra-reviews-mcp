---
name: software-reviews
description: B2B software reviews from G2 (G2 Crowd), Capterra, TrustRadius and Gartner Peer Insights through the software-reviews MCP tools. Use when the user wants software reviews read, compared or summarised, a product's pros, cons, complaints or ratings checked, competitors compared by company size or industry, switching reasons or alternatives found, new reviews monitored, or review data gathered for battlecards, win/loss, voice-of-customer work or SaaS due diligence.
---

# Software reviews (G2, Capterra, TrustRadius, Gartner Peer Insights)

Five review tools plus Apify's run and dataset tools, served by the `software-reviews` MCP server. Each review tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive later in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--software-review-scraper` | One product across G2, Capterra, TrustRadius and Gartner Peer Insights (Trustpilot on request) in one shared format, from a product name, domain or review link. Requested `maxResults` must be at least 100 |
| `zen-studio--g2-reviews-scraper` | G2 reviews of a product or a whole vendor, with date and keyword filters; G2's pros and cons summary |
| `zen-studio--capterra-reviews-scraper` | Capterra reviews with four sub-ratings, switching reasons and products compared; the product's Capterra profile |
| `zen-studio--trustradius-review-scraper` | TrustRadius reviews with feature ratings, alternatives considered and a company-size filter |
| `zen-studio--gartner-review-scraper` | Gartner Peer Insights reviews for up to 10 products per run, with each product's rating profile |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `fetch-actor-details` | Full input reference and README of any of the five |
| `get-key-value-store-record` | Read a record a run saved, such as G2's `pros-cons-summary` or Capterra's `PRODUCT` |
| `abort-actor-run` | Stop a run the user no longer needs |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/g2-reviews-scraper`) and the same input; the run flow is identical.

## Choosing a tool

- **One site named, or depth on one site:** that site's tool. It has the site's own filters and fields and costs less per review than the all-sites tool on most sites.
- **"What do reviewers say about X" with no site named, or a cross-site comparison:** the all-sites tool. It also finds the product's page on each site from a name, which helps when the user gives no link.
- **Fewer than 100 reviews in total:** a single site's tool. The all-sites tool needs `maxResults` of at least 100.
- **Several products on Gartner Peer Insights:** one Gartner run with up to 10 `productUrls`. The other tools take one product per run, so start one run per product.

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the review tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id` (the response's `nextStep` also names it). Pass `limit` (20 to 50) and a `fields` projection from **Reading results**; review rows carry long texts. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: counts, themes with short quotes, ratings, review links (`reviewUrl` or `url`), and anything filtered out. Group by job title, industry or company size when the user asks "who".

Fifteen G2 reviews took 22 to 33 seconds in testing; hundreds of reviews and all-sites runs take minutes.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: fetch the dataset first, since it can hold rows saved before the run ended, and report them as partial; then report `statusMessage`. Change the input only when the message names an input problem (a link, a filter, a minimum). Never retry a paid run on your own: start a new run only when the user agrees. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty.

**Read the `statusMessage` of a successful run too.** It can state limits and coverage that the rows do not show. The all-sites tool ends `SUCCEEDED` with no rows and "maxResults must be at least 100. Please increase and re-run." when `maxResults` is below 100, and its closing summary, such as "Collected 100 reviews (G2: 40, Capterra: 60). No listing on Gartner", names each site with no listing or not included in the run; tell the user. The G2 tool closes a successful run with a summary such as "Done: 20 reviews for Twilio.", and adds "Results may be incomplete." when it could not collect everything; tell the user. A message that only describes work in progress is leftover progress text: ignore it and go by `status` and the rows.

## Input recipes

Start small (15 to 50 reviews per product) unless the user asks for more; every returned review is billed.

**All sites at once** (`software-review-scraper`):

```json
{ "query": "Notion", "platforms": ["g2", "capterra", "trustradius", "gartner"],
  "maxResults": 100, "sort": "most_recent" }
```

- `query`: one product name (`Slack`), a company domain (`asana.com`) or one review page link from any of the sites; the product is found on the other sites. One product per run.
- `platforms`: at least two of `g2`, `capterra`, `trustradius`, `gartner`, `trustpilot`. The default is the first four; add `trustpilot` only when the user asks for it.
- `maxResults`: total reviews to collect, at least 100 (default 100, up to 40,000), shared out across sites by how many reviews each has; a product with fewer reviews returns fewer. Below 100 the run ends `SUCCEEDED` with no rows and the statusMessage "maxResults must be at least 100. Please increase and re-run."
- `sort`: `most_recent` (default), `highest_rated`, `lowest_rated`, `most_helpful`. Each site gets its nearest order:
  - `highest_rated`: G2 and Capterra highest rated, TrustRadius `positive`, Gartner most helpful, Trustpilot newest.
  - `lowest_rated`: G2 and Capterra lowest rated, TrustRadius `critical`, Gartner oldest first, Trustpilot newest.
  - `most_helpful`: G2 newest, Capterra most detailed, TrustRadius most relevant, Gartner and Trustpilot most helpful or relevant.
  So Gartner and Trustpilot rows of a rating sort are not ranked by rating; do not describe them as the best or worst reviews. To compare by rating on every site, filter with `starRatings` instead.
- `starRatings`: any of `"1"` to `"5"`, applied on every site.

**G2** (`g2-reviews-scraper`):

```json
{ "url": "https://www.g2.com/products/pipedrive/reviews", "limit": 30,
  "sortOrder": "lowest_rated", "publishedAfter": "3 months" }
```

- `url`: a product page (`g2.com/products/<slug>/reviews`) or a vendor page (`g2.com/sellers/<slug>`, every product of that vendor). The slug is usually the product name in lower case with hyphens; renamed slugs are followed automatically.
- `limit`: default 50, minimum 10 (lower values return 10). Always set it.
- `sortOrder`: `most_recent` (default), `lowest_rated`, `highest_rated`. `starRatings`: any of `"1"` to `"5"`.
- `publishedAfter`: a date (`2026-10-01`) or a period (`2 weeks`, `1 month`); newest first, stopping at that date. The way to monitor new reviews.
- `searchQuery`: words that must all appear in the review (product pages only). Combined with `starRatings` it can leave nothing; loosen one at a time.
- `includeProsConsSummary` (default `true`) saves G2's own pros and cons summary with mention counts as the `pros-cons-summary` record. A product run can also save `product-summary` with G2's review summary text and the star breakdown, at no extra cost. Neither record is guaranteed: if `get-key-value-store-record` finds no record, say so, and never repeat a paid run to get one.
- `includeVideoReviews` adds video review rows (`type: "video"`, product pages only); leave it off unless the user asks. Each video review is billed like a text review and does not count toward `limit`, and `videoReviewLimit` defaults to `0`, meaning every video. When video is on, always set `videoReviewLimit` (for example 10) and tell the user the run bills up to `limit` plus `videoReviewLimit` reviews. Video rows carry `starRating`, `caption`, `videoPageUrl`, `videoThumbnailUrl`, `surveyResponseUrl`, `videoId` and `productName`, plus the reviewer's name in `authorName`; leave the name out as below. `publishedAfter` does not apply to video reviews.

**Capterra** (`capterra-reviews-scraper`):

```json
{ "productUrl": "https://www.capterra.com/p/141633/PowerPoint/reviews/", "maxResults": 50,
  "sort": "LOWEST_RATED" }
```

- `productUrl`: a product page on any Capterra country site; the product number alone is enough (`capterra.com/p/141633/`). If the run fails with "No Capterra product has the ID in this URL", find the product's current page by name (the all-sites tool returns it) and use that link.
- `maxResults`: the default `0` means every review, and large products have tens of thousands. Always set it.
- `sort`: `MOST_RECENT` (default), `HIGHEST_COMPLETENESS_SCORE` (most detailed), `HIGHEST_RATED`, `LOWEST_RATED`. `starRating`: any of `"1"` to `"5"`.
- The product's full profile (description, pricing plans, features, alternatives, integrations, reviewer company sizes and industries) is usually saved once per run as the `PRODUCT` record at no extra cost. It is not guaranteed: if `get-key-value-store-record` finds no record, say so, and never repeat a paid run to get it.

**TrustRadius** (`trustradius-review-scraper`):

```json
{ "productUrl": "https://www.trustradius.com/products/zoom-workplace/reviews", "maxResults": 30,
  "sort": "critical", "companySize": ["large"] }
```

- `maxResults`: default 100, 1 to 10,000. Always set it.
- `sort`: `recent` (default), `positive`, `critical`, `relevant`.
- `starRating`: `"1"` to `"5"`, mapped onto TrustRadius's 10-point scale (5 stars = 8.5 to 10). The output `rating` stays on the 10-point scale.
- `companySize`: any of `small` (1 to 50 employees), `medium` (51 to 1,000), `large` (1,001 and more).

**Gartner Peer Insights** (`gartner-review-scraper`):

```json
{ "productUrls": ["https://www.gartner.com/reviews/market/meeting-solutions/vendor/zoom/product/zoom-meetings/reviews"],
  "maxResults": 50, "sort": "most-recent" }
```

- `productUrls`: up to 10 product pages; `maxResults`, `sort` and `starRating` apply to each product. The default `maxResults` of `0` means every review; always set it.
- `sort`: `most-recent` (default), `most-helpful`, `least-recent`.
- The first review of each product carries `productData`: average rating, total reviews, star breakdown (`overallRatings`), Gartner market, top three alternatives with their ratings, and `reviewSpotlight` with the most favourable and most critical review in full, including likes and dislikes. The other rows have `productData: null`.

## Chaining

- **Name first, then depth.** Run the all-sites tool on a product name with `maxResults: 100`; each row's `product.url`, when present, is the product's page on that site. Pass it to the site's own tool for filters or more reviews. `reviewUrl` links one review: use it to cite, never as a tool input (G2 rejects review links).
- **Product, then competitors.** Capterra rows list `alternativeProducts` and `switchedProducts`, TrustRadius rows `alternativesConsidered`, and Gartner's `productData.alternatives` names the top three. Run the same question on those products.
- **Reviews, then the vendor's view.** G2's `pros-cons-summary` and `product-summary` records and Capterra's `PRODUCT` record summarise the whole product when the run saved them; read them with `get-key-value-store-record` (`keyValueStoreId` from `storages.keyValueStores.default.id`, `recordKey` as named) before reading hundreds of rows.

## Reading results

Project fields with `get-dataset-items` `fields` (comma-separated). Dot paths into objects such as `reviewer.jobTitle` work and come back flattened, as keys like `"reviewer.jobTitle"`. Good starting sets, without names:

- All sites: `platform,productName,title,text,rating,ratingOriginal,ratingScale,date,pros,cons,reviewer.jobTitle,reviewer.companySize,reviewer.companySizeUnit,reviewer.industry,incentivized,reviewUrl`, plus `product.url` when you plan to chain. Trustpilot rows have no `pros` or `cons`, and Gartner rows usually have none (spotlight reviews can); read `text` for their review.
- G2: `productName,title,starRating,date,likes,dislikes,reviewerTitle,companySize,reviewerIndustry,validatedReviewer,incentivized,reviewSource,reviewUrl`
- Capterra: `title,writtenOn,overallRating,easeOfUseRating,customerSupportRating,functionalityRating,valueForMoneyRating,recommendationRating,prosText,consText,switchingReasons,reviewer.jobTitle,reviewer.companySize,reviewer.industry,reviewUrl`
- TrustRadius: `heading,rating,publishedDate,pros,cons,alternativesConsidered,reviewer.title,reviewer.department,reviewer.companySize,reviewer.industry,url`
- Gartner: `productName,headline,summary,rating,date,jobTitle,industry,companySize,url`, plus `productData` on the first row of each product

Field notes:

- **Company size.** G2 uses bands such as "Small-Business (50 or fewer emp.)", "Mid-Market (51-1000 emp.)" and "Enterprise (> 1000 emp.)"; Capterra and TrustRadius use employee ranges; Gartner uses annual revenue bands such as "1B-10B USD" or a sector such as "Gov't/PS/Ed". In all-sites rows `reviewer.companySizeUnit` says which.
- **G2 reviewers without a job title.** `reviewerTitle` is null on those rows and `companySize` still holds the size band. A reviewer who gives an industry but no title can show the industry name, such as "Information Technology and Services", in `reviewerTitle`; treat a title that reads like an industry as not given. `reviewerIndustry` is often empty on G2 rows.
- **HTML entities in G2 text.** Some G2 fields, such as job titles, contain entities like `&amp;`. Decode them before showing them.
- **Ratings.** G2 `starRating` (half stars) and Capterra `overallRating` run 1 to 5; TrustRadius `rating` runs 1 to 10; all-sites `rating` is on a five-point scale with `ratingOriginal` and `ratingScale` kept. Capterra `recommendationRating` is 0 to 10.
- **Incentives and verification.** G2 `incentivized`, `validatedReviewer` and `reviewSource`; Capterra `incentivized` and `reviewSource`; TrustRadius `trustBadge`; Gartner `incentiveCode`. Report the incentivized share when the user compares ratings.
- **Dates.** G2 `date` is ISO; Capterra `writtenOn` and Gartner `date` are written out ("August 20, 2026", "Feb 25, 2026"); TrustRadius `publishedDate` is an ISO timestamp.

**Personal data.** Rows can carry the reviewer's name or initials (G2 `reviewerName`, Capterra `reviewer.fullName`, TrustRadius `reviewer.name`, all-sites `reviewer.name`), a photo link, and on TrustRadius an employer and a LinkedIn link. Leave these out of projections and answers; describe reviewers by job title, industry and company size. Include names only when the user explicitly asks, and remind them that review data is personal data under GDPR and similar laws.

`fetch-actor-details` returns the full field reference when the user asks for something outside these sets.

## Cost and limits

- Pricing is per returned review on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`), plus a start fee per run, billed to the user's Apify account. Prices differ by site. Quote the page rather than a remembered number.
- On Apify's free plan, G2, Capterra, TrustRadius and the all-sites tool have no review cap per run and no run limit; reviews are paid from the plan's usage credit. Gartner Peer Insights returns up to 50 reviews per run on the free plan, `maxResults: 0` included; the run's `statusMessage` does not mention the cap. When a free-plan user asked for more than 50 Gartner reviews and the dataset holds exactly 50 rows, tell the user the result is capped by the free plan and that any paid Apify plan removes the cap.
- The all-sites tool needs a requested `maxResults` of at least 100 on every plan; a product with fewer reviews returns fewer.
- `limit` (G2) and `maxResults` (the others) are the cost controls; Capterra and Gartner default to every review. State the caps you chose in the answer.
