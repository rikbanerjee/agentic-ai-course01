# Module 2 — Domain & Data Preparation

> **Time**: 3–4 hours
> **Goal**: Have a realistic retail corpus loaded into your repo and a script that explores it.
> **Deliverable**: A `data/raw/` folder with the synthetic retail dataset and a notebook that profiles it.

The course's running domain is **retail / e-commerce search**. Everything you build — agents, RAG, evals — will operate over the dataset you assemble in this module.

---

## 2.1 Why a synthetic dataset?

Real retail data is rarely shareable and rarely small enough to start with. Synthetic data lets us:

- **Run every lab end-to-end** without legal or scale headaches.
- **Inject known failure modes** (ambiguous descriptions, contradictory policies) so evals have real signal.
- **Reproduce results** exactly — your eval scores will match across machines.

When you graduate to your own data, the *shape* of this dataset is the template: products, policies, FAQs, reviews, and orders.

---

## 2.2 The dataset shape

We'll create five data sources, each saved to `data/raw/`:

| File | Format | Rows | What's in it |
|------|--------|------|--------------|
| `products.csv` | CSV | 200 | SKU, name, category, brand, price, attributes (JSON), description, stock |
| `reviews.csv` | CSV | ~1500 | review_id, sku, rating, title, body, verified |
| `policies/*.md` | Markdown | ~10 | Returns, shipping, warranty, price-match — one doc each |
| `faqs.json` | JSON | ~40 | question, answer, tags |
| `orders.csv` | CSV | 50 | order_id, customer_id, sku, qty, ordered_at, status |

This roughly matches what a small-to-mid e-commerce stack would expose to an internal copilot.

---

## 2.3 Lab 2.1 — Generate the synthetic data

We'll use an LLM to *generate* the data, but anchor it with deterministic seeds so re-runs are stable. The script below produces all files at once.

### Python — `python/scripts/build_dataset.py`

```python
"""Generate a synthetic retail dataset using deterministic Python + a sprinkle of LLM."""
from __future__ import annotations
import csv
import json
import random
from pathlib import Path
from datetime import datetime, timedelta

random.seed(42)
ROOT = Path(__file__).resolve().parents[2]
RAW = ROOT / "data" / "raw"
RAW.mkdir(parents=True, exist_ok=True)
(RAW / "policies").mkdir(exist_ok=True)

CATEGORIES = {
    "outerwear": ["jacket", "parka", "rain shell", "fleece"],
    "footwear": ["hiking boot", "trail runner", "sneaker", "sandal"],
    "accessories": ["backpack", "hat", "gloves", "socks"],
    "camping": ["tent", "sleeping bag", "stove", "headlamp"],
}
BRANDS = ["Northcrest", "Ridgeline", "Apex Trail", "Cedar & Pine", "Summit Co"]
ATTRIBUTES = {
    "outerwear": ["waterproof", "windproof", "insulated", "lightweight", "packable"],
    "footwear": ["waterproof", "vegan", "wide-fit", "grippy-sole", "lightweight"],
    "accessories": ["water-resistant", "reflective", "thermal", "merino-wool"],
    "camping": ["3-season", "4-season", "ultralight", "family-size", "quick-pitch"],
}
GENDERS = ["men", "women", "unisex", "kids"]
SIZES_CLOTH = ["XS", "S", "M", "L", "XL", "XXL"]
SIZES_SHOE = ["6", "7", "8", "9", "10", "11", "12"]


def gen_products(n: int = 200) -> list[dict]:
    products = []
    for i in range(1, n + 1):
        category = random.choice(list(CATEGORIES.keys()))
        subtype = random.choice(CATEGORIES[category])
        brand = random.choice(BRANDS)
        gender = random.choice(GENDERS)
        sizes = SIZES_SHOE if category == "footwear" else SIZES_CLOTH
        attrs = random.sample(ATTRIBUTES[category], k=random.randint(1, 3))
        price = round(random.uniform(25, 450), 2)
        sku = f"SKU-{1000 + i}"
        name = f"{brand} {gender.title()} {subtype.title()}"
        stock = random.choice([0, 0, 5, 12, 30, 100])
        description = (
            f"The {name} is a {', '.join(attrs)} {subtype} built for everyday adventures. "
            f"Available in sizes {', '.join(sizes)}."
        )
        products.append(
            {
                "sku": sku,
                "name": name,
                "category": category,
                "subtype": subtype,
                "brand": brand,
                "gender": gender,
                "price": price,
                "attributes": json.dumps(attrs),
                "sizes": json.dumps(sizes),
                "stock": stock,
                "description": description,
            }
        )
    return products


def gen_reviews(products: list[dict], avg_per_product: int = 8) -> list[dict]:
    reviews = []
    rid = 0
    sample_bodies = [
        "Fits true to size and held up in two weeks of rain.",
        "Color is slightly different than the photos, otherwise great.",
        "Wore them on a 12-mile hike, no blisters.",
        "Stitching came apart after a month — disappointed.",
        "Great value for the price, would buy again.",
        "Runs small — order one size up.",
        "Perfect for cold mornings, breathable enough for summer evenings.",
        "Heavier than I expected, otherwise comfortable.",
        "My kid loves it. Easy to clean.",
        "Returned — zipper got stuck on day one.",
    ]
    for p in products:
        n = max(1, int(random.gauss(avg_per_product, 3)))
        for _ in range(n):
            rid += 1
            reviews.append(
                {
                    "review_id": f"R-{rid:05d}",
                    "sku": p["sku"],
                    "rating": random.choices([1, 2, 3, 4, 5], weights=[3, 5, 12, 35, 45])[0],
                    "title": random.choice(["Love it", "Solid", "Meh", "Disappointed", "Worth it"]),
                    "body": random.choice(sample_bodies),
                    "verified": random.random() > 0.15,
                }
            )
    return reviews


def gen_orders(products: list[dict], n: int = 50) -> list[dict]:
    orders = []
    today = datetime(2026, 5, 1)
    for i in range(1, n + 1):
        p = random.choice(products)
        orders.append(
            {
                "order_id": f"ORD-{4000 + i}",
                "customer_id": f"C-{random.randint(100, 999)}",
                "sku": p["sku"],
                "qty": random.randint(1, 3),
                "ordered_at": (today - timedelta(days=random.randint(0, 60))).isoformat(),
                "status": random.choices(
                    ["shipped", "delivered", "processing", "returned", "cancelled"],
                    weights=[35, 40, 15, 7, 3],
                )[0],
            }
        )
    return orders


POLICIES = {
    "returns.md": """# Returns Policy

We accept returns within **30 days** of delivery for unworn, unwashed items with original tags.

- Final-sale items (marked SALE-FINAL) are **not returnable**.
- Footwear must be returned in original box; outdoor wear is allowed once.
- Refunds issue to the original payment method within 5–7 business days.
- Return shipping is free for domestic orders; international customers pay return shipping.

Exchanges: ship the new item out as soon as the return is in transit.
""",
    "shipping.md": """# Shipping Policy

- **Standard** (5–7 business days): free over $50, otherwise $6.
- **Express** (2 business days): $14 flat.
- **Overnight**: $29, order must be placed before 1pm ET.
- We ship to all 50 US states plus Canada. International via DHL on request.

Orders to AK/HI add 2 business days. P.O. boxes accepted for Standard only.
""",
    "warranty.md": """# Warranty Policy

All footwear and outerwear carry a **1-year manufacturer warranty** against defects.

- Defects covered: stitching failure, zipper failure, sole separation.
- Not covered: normal wear, accidental damage, alterations by third parties.
- Submit a warranty claim with order ID and photos.
- Resolution: repair, replace, or refund at our discretion within 14 days of approval.
""",
    "price-match.md": """# Price Match Policy

We match prices on identical items from authorized retailers (REI, Backcountry, Patagonia.com).

- Match request must be made within **14 days** of purchase.
- Item must be identical SKU, color, size, and in-stock at the competitor.
- Marketplaces (Amazon, eBay) and member-only prices are excluded.
""",
    "loyalty.md": """# Loyalty Program

Members earn **1 point per $1 spent**; 500 points = $25 credit.

- Free shipping on all orders for members.
- Early access to sales 48 hours before the public.
- Birthday bonus: 100 points in your birthday month.
- Points expire 12 months after issue.
""",
}

FAQS = [
    {"q": "Do you offer student discounts?", "a": "Yes — verify with SheerID for 15% off.", "tags": ["pricing", "discount"]},
    {"q": "Can I change my shipping address after ordering?", "a": "Only if the order has not yet shipped. Contact support@retail.example within 1 hour.", "tags": ["shipping", "orders"]},
    {"q": "Are your products vegan?", "a": "Products tagged 'vegan' contain no animal materials. Check the product page badges.", "tags": ["product", "materials"]},
    {"q": "What size should I order for a 6-year-old?", "a": "Kids' XS fits ages 4–6; refer to the size chart on each product page.", "tags": ["sizing", "kids"]},
    {"q": "Do you price match Amazon?", "a": "No, marketplaces are excluded from our price-match policy.", "tags": ["pricing", "price-match"]},
    {"q": "How long does a return take to process?", "a": "5–7 business days after we receive the package.", "tags": ["returns"]},
    {"q": "Can I use multiple promo codes?", "a": "Only one promo code per order.", "tags": ["pricing", "discount"]},
    {"q": "Are gift cards refundable?", "a": "Gift cards are non-refundable but never expire.", "tags": ["gift-cards"]},
    {"q": "Do you ship to APO/FPO?", "a": "Yes, via USPS Priority — allow 10–14 business days.", "tags": ["shipping", "military"]},
    {"q": "What's the difference between waterproof and water-resistant?", "a": "Waterproof items keep water out under pressure; water-resistant items repel light rain only.", "tags": ["product", "materials"]},
]


def main():
    products = gen_products()
    reviews = gen_reviews(products)
    orders = gen_orders(products)

    with (RAW / "products.csv").open("w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(products[0].keys()))
        writer.writeheader()
        writer.writerows(products)

    with (RAW / "reviews.csv").open("w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(reviews[0].keys()))
        writer.writeheader()
        writer.writerows(reviews)

    with (RAW / "orders.csv").open("w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(orders[0].keys()))
        writer.writeheader()
        writer.writerows(orders)

    for name, content in POLICIES.items():
        (RAW / "policies" / name).write_text(content)

    (RAW / "faqs.json").write_text(json.dumps(FAQS, indent=2))

    print(f"✅ Wrote {len(products)} products, {len(reviews)} reviews, "
          f"{len(orders)} orders, {len(POLICIES)} policies, {len(FAQS)} FAQs to {RAW}")


if __name__ == "__main__":
    main()
```

Run it:

```bash
cd python && uv run python scripts/build_dataset.py
```

### TypeScript — `typescript/scripts/buildDataset.ts`

If you prefer to generate from TS, use the same logic. (Many course participants generate once in Python and use the resulting files from both languages — recommended.)

```typescript
// typescript/scripts/buildDataset.ts
import { mkdir, writeFile } from 'node:fs/promises';
import { join } from 'node:path';

const ROOT = join(import.meta.dirname, '..', '..');
const RAW = join(ROOT, 'data', 'raw');

function seedRand(seed: number) {
  let s = seed;
  return () => {
    s = (s * 9301 + 49297) % 233280;
    return s / 233280;
  };
}
const rand = seedRand(42);
const pick = <T,>(arr: T[]) => arr[Math.floor(rand() * arr.length)];
const sample = <T,>(arr: T[], k: number) =>
  [...arr].sort(() => rand() - 0.5).slice(0, k);

// (Truncated — port the Python categories/brands/attributes arrays and gen functions.
// See the Python version for the source of truth.)

async function main() {
  await mkdir(join(RAW, 'policies'), { recursive: true });
  // ... call gen functions, write CSVs/JSON/MD
  console.log('✅ Dataset generated');
}
main();
```

> 💡 **Tip**
> Generate the dataset in one language and read it from both. This module isn't about teaching CSV writing — it's about getting a corpus you can build on.

---

## 2.4 Lab 2.2 — Profile the corpus

Open a notebook (`notebooks/00_exploration.ipynb`) and run a quick profile.

```python
import pandas as pd, json
from pathlib import Path

RAW = Path("../data/raw")
products = pd.read_csv(RAW / "products.csv")
reviews = pd.read_csv(RAW / "reviews.csv")
orders = pd.read_csv(RAW / "orders.csv")

print("Products:", len(products), "| Categories:", products["category"].unique())
print(products["price"].describe())
print("\nStock distribution:\n", products["stock"].value_counts().sort_index())
print("\nReviews per SKU (top 5):\n", reviews.groupby("sku").size().sort_values(ascending=False).head())
print("\nOrder status mix:\n", orders["status"].value_counts())
```

You should see something like 200 products, $25–450 price range, ~1500 reviews, mixed stock and order statuses.

---

## 2.5 Why this matters for agentic AI

When you start building tools in Module 5, every tool you write will be operating over this data:

- `search_products(query, max_price?, in_stock_only?)` → reads `products.csv`
- `get_product(sku)` → reads `products.csv`
- `get_order(order_id)` → reads `orders.csv`
- `get_policy(topic)` → RAG over `data/raw/policies/*.md`
- `get_faq(question)` → RAG over `faqs.json`

> 💡 **Why this matters**
> "Data fluency" is underrated in agent design. If you can't describe your corpus' fields, distributions, and edge cases in 60 seconds, your agent's tool descriptions will be vague — and the LLM will make bad tool choices.

---

## 2.6 The "data card" — your reference doc

Create `docs/data_card.md`:

```markdown
# Data Card — Retail Copilot Corpus

## Sources
- `data/raw/products.csv` — 200 synthetic products, 4 categories, 5 brands
- `data/raw/reviews.csv` — ~1500 reviews, ratings 1–5
- `data/raw/orders.csv` — 50 orders, statuses [shipped, delivered, processing, returned, cancelled]
- `data/raw/policies/*.md` — 5 policy docs (returns, shipping, warranty, price-match, loyalty)
- `data/raw/faqs.json` — 10 FAQs

## Known properties
- Generated with seed=42 → fully reproducible
- ~10% of products are out of stock (intentional)
- ~7% of orders are "returned" (so the agent has to handle that case)
- Reviews skew positive (4–5 stars)
- Policy docs intentionally contain a contradiction edge case (price-match excludes Amazon, but the FAQ duplicates it — use this for eval cases)

## Known gaps (intentional, for evals)
- No images
- No size charts (mentioned but not provided)
- No multilingual content
- No PII
```

This card is the single source of truth when an eval question comes up like "should the agent admit it doesn't know X?"

---

## 2.7 Checklist before moving on

- [ ] `data/raw/products.csv`, `reviews.csv`, `orders.csv` exist with the expected row counts.
- [ ] `data/raw/policies/` contains 5 markdown files.
- [ ] `data/raw/faqs.json` is valid JSON.
- [ ] You profiled the corpus in a notebook and wrote down at least 3 surprises.
- [ ] `docs/data_card.md` is written.

Head to [Module 1 — GenAI & Agentic Apps](03-module-1-genai-foundations.md).

---

## Decision-log prompts

- What's the smallest version of this dataset that still tests your agent meaningfully?
- Which fields would change if you swapped synthetic data for a real catalog?
- What's one edge case in your data that you'd want every eval to cover?
