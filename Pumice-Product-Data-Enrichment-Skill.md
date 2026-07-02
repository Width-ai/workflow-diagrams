# Pumice API Reference

Generate, enrich, and standardize product content at scale. With a single REST API you can
create titles, descriptions, and bullet points, extract and standardize attributes,
categorize and match products, describe product images, and research products from the web —
all tuned to your brand's voice and rules.

## Base URL

```
https://app.pumice.ai/api
```

All endpoints are relative to this base URL. Every request is a `POST` with a JSON body
(unless noted otherwise).

## Authentication

Every request requires an API key. Generate one in the **Pumice dashboard → Settings → API
Keys**, then include it in the `x-api-key` header:

```
x-api-key: YOUR_API_KEY
Content-Type: application/json
```

Keep your API key secret — treat it like a password, and don't embed it in client-side code.
You can create, rotate, and revoke keys at any time from the dashboard.

## How it works

Pumice enriches product data in two phases. Use either on its own, or chain them together —
**research** to gather raw source information, then **generation** to turn it into polished,
customer-ready content.

### Research phase

When you start with only sparse data — a name, an MPN, a manufacturer domain — the research
endpoints find and extract richer product information from the web.

- **Universal Search (`/scraper/universal`)** — give it a query (a product name, part number,
  or targeted `site:` search) and it searches the web to surface the most relevant product
  pages and source URLs. This is how you locate authoritative pages — manufacturer sites,
  distributor listings, spec sheets — worth enriching from. It takes `{ "query": "..." }`
  and returns a ranked list of matching pages.

- **Smart Scraper (`/scraper/smart-scrape`)** — point it at a page with a plain-language
  prompt describing what you want (e.g. *"extract the title, description, specs, SKU, and
  image URLs"*) and it reads the page like a person would, returning exactly those fields as
  structured JSON. It takes `{ "prompt": "...", "source_url": "..." }`.

A typical research flow: run Universal Search to find candidate pages, then run Smart Scraper
on the best matches to pull full product details. The results feed directly into the
generation endpoints below.

### Generation endpoints

The generation endpoints take product data — whatever you already have, plus anything
gathered during research — and produce optimized, on-brand output. Every one is guided by your
`customer_rules` and optional `examples`, so results match your voice, length limits, and
formatting standards. At a high level they let you:

- **Write customer-facing copy** — titles (`/generate_title`), descriptions
  (`/generate_description`), and feature/benefit bullets (`/generate_bullet`).
- **Structure product data** — generate attributes from a target list
  (`/generate_attribute`), extract attributes already present in the content
  (`/attribute_extraction`), and standardize variant values
  (`/product_variant_field_standardization`).
- **Organize and reconcile catalogs** — categorize products against your taxonomy
  (`/categorization/{model_id}/predict`) and detect duplicates or matches
  (`/product_matching`).
- **Work from images** — generate a description straight from a product photo
  (`/generate_image_description`).
- **Clean up and check content** — rewrite messy descriptions (`/rewrite_product`) and
  validate content against your rules (`/run_llm_validator`).

Each generation endpoint returns a focused JSON result — see the reference below.

## The request format

Most content endpoints share a common request shape:

```json
{
  "product": {
    "title": "Widget",
    "description": "A compact blue widget",
    "image_url": "https://example.com/widget.jpg"
  },
  "customer_rules": ["Keep it under 80 characters", "Use a friendly, professional tone"],
  "examples": []
}
```

- **`product`** — the product information you already have (title, description, image URL).
  Provide as much as you can; more context yields better output.
- **`customer_rules`** — plain-language guidelines the model should follow, such as brand
  voice, length limits, formatting requirements, or words to avoid.
- **`examples`** — optional sample outputs that steer the model toward your preferred style.

## Endpoints

| Endpoint (`POST`)                          | What it does                                             | Returns                 |
|--------------------------------------------|----------------------------------------------------------|-------------------------|
| `/generate_title`                          | Generate an optimized product title                      | `product_title`         |
| `/generate_description`                    | Generate a product description                           | `product_description`   |
| `/generate_bullet`                         | Generate feature/benefit bullet points                   | `product_bullets`       |
| `/generate_attribute`                      | Generate product attributes from a target list           | attributes              |
| `/attribute_extraction`                    | Extract existing attributes from product content         | attributes              |
| `/categorization/{model_id}/predict`       | Categorize products against your taxonomy model          | predictions             |
| `/product_matching`                        | Decide whether two products are the same                 | `answer` (`yes`/`no`)   |
| `/rewrite_product`                         | Rewrite/clean up a product description                   | `product_description`   |
| `/run_llm_validator`                       | Validate content against your rules                      | validation result       |
| `/product_variant_field_standardization`   | Standardize variant attribute values                     | `result`                |
| `/generate_image_description`              | Describe a product from its image (multipart upload)      | `image_description`     |
| `/scraper/universal`                       | Search the web for product sources                       | search results          |
| `/scraper/smart-scrape`                    | Extract structured data from a product page              | scraped data            |

### Generate a title

```bash
curl -X POST https://app.pumice.ai/api/generate_title \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "product": {
          "title": "Widget",
          "description": "A compact blue widget",
          "image_url": "https://example.com/widget.jpg"
        },
        "customer_rules": ["Keep it under 80 characters"],
        "examples": []
      }'
```

```json
{ "product_title": "Compact Blue Widget — Durable, Everyday-Ready Design" }
```

`/generate_description` and `/generate_bullet` accept the same request shape and return
`product_description` and `product_bullets` respectively.

### Categorize products

Categorization runs a batch of products against one of your taxonomy models. Replace
`{model_id}` with your categorization model's identifier from the dashboard.

```bash
curl -X POST https://app.pumice.ai/api/categorization/taxonomy-v2/predict \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "products": [
          {
            "title": "Widget",
            "description": "A compact blue widget",
            "image_url": "https://example.com/widget.jpg"
          }
        ]
      }'
```

You can send multiple products in a single request; each is billed as one prediction.

### Match two products

```bash
curl -X POST https://app.pumice.ai/api/product_matching \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "product_1": "Compact Blue Widget, MPN WB-100",
        "product_2": "Blue Widget (Compact) — item WB100",
        "customer_rules": [],
        "fields": "",
        "examples": null
      }'
```

```json
{ "answer": "yes" }
```

### Describe a product image

This endpoint accepts a multipart upload (an image file plus optional rules/examples), rather
than JSON:

```bash
curl -X POST https://app.pumice.ai/api/generate_image_description \
  -H "x-api-key: YOUR_API_KEY" \
  -F "image=@/path/to/product.jpg;type=image/jpeg" \
  -F 'customer_rules=["Focus on material and color"]' \
  -F 'examples=[]'
```

```json
{ "image_description": "A compact blue widget with a matte finish and rounded edges..." }
```

## Usage & credits

Each successful call consumes credits from your plan. Batch endpoints such as
`/categorization/{model_id}/predict` consume one credit per product in the request. Failed
requests are not billed. Track your usage and remaining credits in the dashboard.

## Errors

Pumice uses standard HTTP status codes:

| Status | Meaning                                                                 |
|--------|-------------------------------------------------------------------------|
| `200`  | Success                                                                 |
| `401`  | Missing or invalid API key                                              |
| `403`  | Your plan or key doesn't have access to this resource                   |
| `422`  | The request body failed validation (check required fields and format)   |
| `429`  | Rate limit exceeded — slow down and retry                               |
| `5xx`  | A temporary server error — retry after a short delay                    |

Error responses include a JSON body with an `error` message describing what went wrong.

Generation and research calls can take time on large or complex products — allow a generous
client-side timeout for these requests.

## Support

Manage API keys, monitor usage, and find your taxonomy model IDs in the
[Pumice dashboard](https://app.pumice.ai). For help, contact your Pumice account team.
