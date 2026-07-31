# Youdo.com AI Orders Monitor

Automated Youdo.com AI order monitoring via mobile API signature replay — fetch, keyword-filter, deduplicate, Telegram notifications with inline cover letter button.

## How it works

1. **Multi-query fetch** — calls Youdo mobile API (`v110/tasks/tasks`) with multiple frozen URL+signature pairs:
   - General IT listing (Category=4194304, 50 tasks per page)
   - Targeted search queries: `q=ии`, `q=AI`, `q=Ии`, `q=aeo`, `q=AEO`
2. **Deduplication** — by task ID across all queries
3. **AI keyword filter** — deterministic regex matching (long + short keywords with Cyrillic word boundaries)
4. **Description fetch** — Youdo listing API returns `Description: ""` (empty). Full description is fetched via Firecrawl rotator (`127.0.0.1:9123`) scraping `youdo.com/t<id>` and extracting the «Нужно» section from rendered markdown.
5. **Telegram notification** — HTML-formatted post with title, description, price, date, client name, and inline «Написать отклик» button
6. **JSONL cache** — all orders saved for cover letter bot

## API signatures

Youdo API uses HMAC-SHA256 signatures (`KeyVersion: 1`). The signature is **per-URL** — computed over the exact URL (path + query string including `SearchRequestId`, `lat`, `lng`, `Category`, `q`, `priceMin`, `page`, `pageSize`). Changing any parameter invalidates the signature.

Each `(url, signature)` pair is **frozen** — works indefinitely for that exact URL. New search queries require a new Proxyman HAR capture from the iOS app.

### Verified search queries (2026-07-31)

| Query | Category | Total | Signature (first 20 chars) |
|---|---|---|---|
| `ии` (all categories) | Cat=1 | 26 | `79Oq4vy7/24HCbr7Xds0...` |
| `AI` | Cat=4194304 IT | 3 | `YfNfdfOLb9uPPY6aZ...` |
| `Ии` | Cat=4194304 IT | 3 | `jV9hZpGAKfVIJiUM...` |
| `aeo` | Cat=1 all | 1 | `ItWH80cBsl/Xkj9h...` |
| `AEO` | Cat=4194304 IT | 1 | `JSb/70Ykz/Zv5e9Q...` |

Queries with 0 results (no active tasks): `CrewAi`, `LangGraph`, `N8n`, `RAG`, `LLM`, `Нейросеть`, `Искусственный интеллект`.

## Headers

```http
Host: api.youdo.com
X-AdvId: 837262DA-8F62-4D19-AF65-31C99119BED7
KeyVersion: 1
X-FeatureSetId: 976
Signature: <per-URL HMAC-SHA256>
X-VisitorId: E38D2FBB-515B-4B6D-AD76-665B0027C70A
X-DeviceId: E38D2FBB-515B-4B6D-AD76-665B0027C70A
User-Agent: 26.5.2,4.226.0.3030102,iosPhoneApp,430x932,3,apple,iPhone16_2,E38D2FBB-515B-4B6D-AD76-665B0027C70A
msid: 6ccec869-4690-46b4-ad77-dcd41861f9cd
```

## Description fetch (Firecrawl)

Youdo listing API returns `Description: ""` for all tasks. Full description is scraped from `youdo.com/t<id>` via Firecrawl rotator:

```python
resp = requests.post("http://127.0.0.1:9123/v2/scrape", json={
    "url": f"https://youdo.com/t{task_id}",
    "formats": ["markdown"],
    "onlyMainContent": True,
    "waitFor": 2000,
}, timeout=15)
# Extract «Нужно» section from markdown
```

The `servicepipe.tech` JS-challenge on youdo.com blocks raw `requests.get()` — Firecrawl bypasses it via headless browser rendering.

## Telegram post format

```
#YouDo
Создать чат бот

Необходимо сделать бот с вопросами (вопросы и ответы подготовлены) в телеграм. Бот должен записывать инфу в базу и уведомлять сотрудника.

💰 до 14000 ₽
🕐 31.07 09:18
👤 Екатерина

🔗 https://youdo.com/t15031340
[📝 Написать отклик] (inline button)
```

## Cron

`*/5 * * * *` — runs `youdo_cron_check.sh` which calls `youdo_check.py`.

## Requirements

- Python 3.12+, `requests`
- Firecrawl rotator at `127.0.0.1:9123` (for description fetch)
- Telegram bot token + chat ID

## Files

- `youdo_check.py` — main script: fetch, filter, dedup, Telegram, JSONL
- `requirements.txt` — Python dependencies

## License

MIT