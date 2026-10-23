<p align="center"><a href="https://apify.com/DevilScrapes/meetup-events-scraper"><img src="assets/banner.svg" alt="Meetup Events Scraper — JavaScript example" width="100%"></a></p>

# Meetup Scraper for Node.js: event search results as JSON for backends and webhooks

<p align="center">
<a href="https://apify.com/DevilScrapes/meetup-events-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/meetup-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/meetup-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to query Meetup.com from Node.js through the **[Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper)**. It is one ES module wrapped around the official `apify-client` package: send keywords and an optional location, and you receive event objects with a title and link, an ISO start time, an online flag, the RSVP total and capacity, any fee with its currency, venue fields and the hosting group's page. The scraping runs on Apify, which means the code you deploy stays tiny, whether it sits in a cron container, an API route or a serverless function. Run from the terminal, it writes `results.json` and `results.csv` next to the script.

> **Actor:** [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [meetup-scraper-python](https://github.com/devilscrapes/meetup-scraper-python)

## 🔥 What you get

- **One search per keyword**: every term in `keywords` runs its own Meetup find query, scoped to `location` when you give one and global when you leave it empty.
- **In-person or online on request**: `eventType` keeps only `physical` or only `online` events, and every row carries both `event_type` and a boolean `is_online`.
- **Turnout and capacity side by side**: `going_count` is the public RSVP total and `max_tickets` the host's cap when one exists, which together give a fill rate.
- **Fees made explicit**: `is_free` is true when the event has no fee settings; paid events add `fee_amount` as a number and `fee_currency`.
- **The organizer behind each event**: `group_name` plus `group_url`, a direct link to the hosting group's Meetup page.
- **Leaner rows on demand**: switch `includeDescription` off and the long event text is returned as `null`.
- **Resilient per keyword**: throttled or failing requests are retried up to five times with growing pauses, and a keyword that still fails is logged and skipped without stopping the others.

## 🎯 Use cases

### Post new community events to a webhook

Keep the `event_id` values from the previous run, compare them with the latest results and send only the new events to Slack, Discord or Teams. The snippet below does exactly this with Node built-ins and the global `fetch`.

### Back a 'what's on this week' page

A scheduled job refreshes a few keyword searches for your city and writes the objects to a database table. Your site lists them by `date_time`, labels online ones from `is_online`, and links each card to `event_url` for the RSVP.

### Feed a devrel dashboard

Developer relations teams can track every event that mentions their product or framework, store `going_count` per run, and chart attendance per hosting group. The `group_url` field is a stable key for each community over time.

### Route sponsorship leads to sales

When a search turns up an in-person event with a large `going_count` and a near-full capacity, create a task in your CRM pointing at `group_url`. Sales gets the organizer's group page; what they do with it stays a human decision.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/meetup-scraper-javascript.git
cd meetup-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Underneath the CLI arguments, the integration is a client, one `call()` that resolves when the run ends, and one `listItems()` request for the dataset.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/meetup-events-scraper').call({"keywords": ["ai"], "location": "New York, NY", "eventType": "any", "maxEventsPerKeyword": 10, "includeDescription": true, "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` reads `input.json` and passes it on as-is. The sample searches for `ai` in New York and caps the result at ten events, so the first run is cheap. Each extra keyword triggers its own search, and `location` accepts a city and region string such as `Austin, TX` or nothing at all for a worldwide search.

```json
{
  "keywords": [
    "ai"
  ],
  "location": "New York, NY",
  "eventType": "any",
  "maxEventsPerKeyword": 10,
  "includeDescription": true,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `keywords` | array | yes | `["ai"]` | — |
| `location` | string | no | `"New York, NY"` | — |
| `eventType` | string | no | `"any"` | `any`, `physical`, `online` |
| `maxEventsPerKeyword` | integer | no | `50` | — |
| `includeDescription` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Objects are flat and JSON-safe. `fee_amount` is a number or `null`, `max_tickets` is `null` when no cap was set, and `description` is `null` when you turned descriptions off, so check for nulls before rendering.

| Field | Type | Always present |
|---|---|---|
| `search_keyword` | string | yes |
| `event_id` | string | yes |
| `title` | string | yes |
| `event_url` | string | yes |
| `date_time` | string | yes |
| `event_type` | string | yes |
| `is_online` | boolean | no |
| `going_count` | integer | no |
| `max_tickets` | integer or null | no |
| `is_free` | boolean | no |
| `fee_amount` | number or null | no |
| `fee_currency` | string or null | no |
| `venue_name` | string or null | no |
| `venue_address` | string or null | no |
| `venue_city` | string or null | no |
| `venue_state` | string or null | no |
| `venue_country` | string or null | no |
| `latitude` | number or null | no |
| `longitude` | number or null | no |
| `group_name` | string or null | no |
| `group_url` | string or null | no |
| `description` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/meetup-events-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This snippet compares today's results with a saved list of known IDs and posts the new events to a webhook set in `WEBHOOK_URL`. Without that variable it simply prints them. Only Node 20 built-ins are used.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const known = new Set(JSON.parse(await readFile('seen.json', 'utf8').catch(() => '[]')));

const fresh = new Map();
for (const row of rows) {
    if (!known.has(row.event_id)) fresh.set(row.event_id, row);
}

const lines = [...fresh.values()]
    .sort((a, b) => Date.parse(a.date_time) - Date.parse(b.date_time))
    .map((e) => {
        const where = e.is_online ? 'online' : [e.venue_name, e.venue_city].filter(Boolean).join(', ');
        const fee = e.is_free ? 'free' : `${e.fee_amount ?? '?'} ${e.fee_currency ?? ''}`.trim();
        return `${e.date_time.slice(0, 16)} | ${e.title} | ${e.group_name ?? 'unknown group'} | ${where} | ${e.going_count} going | ${fee} | ${e.event_url}`;
    });

if (lines.length && process.env.WEBHOOK_URL) {
    await fetch(process.env.WEBHOOK_URL, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ text: lines.join('\n') }),
    });
}
console.log(lines.length ? lines.join('\n') : 'no new events');
await writeFile('seen.json', JSON.stringify([...known, ...fresh.keys()]));
```

## ❓ FAQ

### Do I need access to Meetup's developer API?

No. This example never touches Meetup's developer API and needs no Meetup key or OAuth app. The Actor reads public search results; you authenticate only to Apify.

### How do I keep a serverless function within its timeout?

For a short keyword list, waiting on `call()` is fine. For long keyword lists, start the run with `start()`, return right away and let an Apify webhook on run success trigger the function that reads the dataset.

### Can the same event show up twice?

Yes, when it matches several keywords. Each search writes its own rows, so an event found by `ai` and by `machine learning` is stored and billed twice; `search_keyword` shows which term found it, and keying on `event_id` merges them.

### Does a keyword with no matches fail the run?

No. It adds zero rows and the run carries on. A run where nothing matches at all costs only the start fee.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/meetup-events-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Meetup Events Scraper on Apify Store](https://apify.com/DevilScrapes/meetup-events-scraper)
- [Python version of this example](https://github.com/devilscrapes/meetup-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) — start from specific Meetup groups and pull their upcoming and past events.
- [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) — cover the Luma side of the tech scene with host and ticket fields.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — add ticketed conferences and workshops listed on Eventbrite.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/meetup-events-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Meetup (Meetup.com).
