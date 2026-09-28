# Otto — the Deal Concierge

**Composable AI Hackathon 2026 · Track T6 (Composable commerce orchestration)**
Bloomreach · Google · Shopify · Databricks — plus a fifth system we brought ourselves: **Voucherify**.

> Agentic commerce has solved buying and ignored pricing. Otto is the missing layer:
> a conversational agent that knows who the shopper is, what the customer is entitled to, how
> likely the customer is to leave — and that never invents a single number.

---

## Table of contents

1. [The problem, and why it is worth money](#1-the-problem-and-why-it-is-worth-money)
2. [What we built](#2-what-we-built)
3. [System architecture](#3-system-architecture)
4. [The five systems, and what breaks without each](#4-the-five-systems-and-what-breaks-without-each)
5. [Scenario 1 — the guided purchase](#5-scenario-1--the-guided-purchase)
6. [Scenario 2 — live cart-abandonment rescue](#6-scenario-2--live-cart-abandonment-rescue)
7. [Scenario 3 — weekly opportunity agent to Vincent](#7-scenario-3--weekly-opportunity-agent-to-vincent)
8. [The engineering decisions that make it trustworthy](#8-the-engineering-decisions-that-make-it-trustworthy)
9. [Business impact](#9-business-impact)
10. [Self-check against the submission bar](#10-self-check-against-the-submission-bar)

---

## 1. The problem, and why it is worth money

When a shopper buys through an AI agent today, the agent finds a product and pays **list price**.
Promotions, bundles, stackable codes, loyalty balances and tier economics are invisible to it.

Two parties lose:

| Who | What they lose today |
|---|---|
| **The shopper** | A worse deal through AI than on the merchant's own website. The agent cannot see that adding one more item unlocks 70% off a jacket, or that joining now earns the customer N points on this basket. |
| **The merchant** | The one lever that makes e-commerce profitable: *deciding who gets which incentive, and when to say no.* Their only options are to block agents entirely, or publish blanket codes an agent then hands to everyone. |

The economic buyer is the brand's growth / loyalty lead, who is being asked to open agentic
channels without losing control of discount depth, stacking rules and loyalty liability.

**Our thesis:** the agent should be allowed to negotiate, but only inside guardrails the
merchant already owns — and every amount it quotes must come from the promotion engine,
never from the language model.

The same rules, limits and validation that govern a promotion on the website govern it in
the agent. Agent-ready by design means nothing new for the merchant to build, and nothing
new to leak.

---

## 2. What we built

**Otto** is a Gemini-orchestrated shopping agent, live in Gemini Enterprise, backed by a
vendor-neutral [UCP](https://ucp.dev) commerce server on Cloud Run. It:

- **Recognises** the shopper from the customer's Bloomreach profile and purchase history, and opens the
  conversation with what the customer actually bought last time.
- **Suggests** complementary products, grounded in Bloomreach recommendations and resolved
  against Shopify for price, availability and truth — *Bloomreach ranks, Shopify decides*.
- **Negotiates inside merchant guardrails.** Offers are surfaced as structured cart actions
  ("add the bindings and the 70%-off jacket tier unlocks", "joining now earns N points on this basket").
  Every candidate offer is dry-run-validated by Voucherify against *this* cart and *this*
  customer before Otto is allowed to mention it. Rejections come back structured, with reasons.
- **Reads the risk in the room.** When the shopper hesitates, Otto calls a **Databricks agent**
  that combines the live conversation signals with the customer's real purchase and abandonment history and
  returns a calibrated abandonment probability. That probability is written onto the order as
  metadata, and *Voucherify's own rule* decides whether a rescue discount applies and how deep.
- **Commits atomically.** The checkout link is a Shopify draft order that already carries the
  customer, the address and every line discount. Redemption in Voucherify happens only after
  the order is genuinely **paid**, triggered by webhook or a reconciler — never optimistically.
- **Closes the loop.** The purchase flows back into Bloomreach through the store integration,
  loyalty points accrue in Voucherify, and every abandonment assessment — probability, drivers,
  reasoning, and the discount that was actually applied — is written to a Databricks table:
  real, labeled examples for a future scoring run to learn from.

**Not everything we built waits for a shopper.** The same data that scores one hesitating
customer also feeds a second, scheduled agent that watches the store as a whole — where intent
is holding up but conversion is slipping — and can open a promotion case with Vincent,
Voucherify's promotion agent. Vincent drafts the audience and the incentive, and a marketer
approves, adjusts or rejects the proposal. Once approved, the promotion is created in Voucherify
and the customers who match can be told about it through Bloomreach — an email with their new
discount, sent to exactly the people the two agents described.

Three demo scenarios exercise three different shapes of agency: **conversational** (§5),
**real-time reactive** (§6) and **autonomous / scheduled** (§7).

---

## 3. System architecture

**The request path** — what happens while the shopper is typing:

```
                        ┌─────────────────────────────────────┐
                        │         Gemini Enterprise           │
                        │       the shopper's surface         │
                        └──────────────────┬──────────────────┘
                                           │  A2A JSON-RPC + A2UI widgets
                                           │  (Google-signed ID token)
                                           ▼
                        ┌─────────────────────────────────────┐
                        │            Otto agent               │
                        │   ADK · A2A · A2UI · Gemini 2.5     │
                        │   owns the CONVERSATION only:       │
                        │   phases, consent gates, widgets    │
                        └──────────────────┬──────────────────┘
                                           │  MCP Streamable HTTP, stateless
                                           │  (Google-signed ID token)
                                           ▼
                        ┌─────────────────────────────────────┐
                        │        UCP commerce server          │
                        │   owns the CART, the OFFERS,        │
                        │   EVERY AMOUNT, the checkout link   │
                        │   and post-payment redemption       │
                        └──┬─────────┬────────────┬────────┬──┘
           ┌───────────────┘         │            │        └───────────────┐
           ▼                         ▼            ▼                        ▼
  ┌─────────────────┐    ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐
  │     SHOPIFY     │    │    VOUCHERIFY    │  │   BLOOMREACH   │  │   DATABRICKS   │
  ├─────────────────┤    ├──────────────────┤  ├────────────────┤  ├────────────────┤
  │ catalog         │    │ qualifications   │  │ via Loomi      │  │ Lakehouse      │
  │ variants, stock │    │ validate · lock  │  │ Connect (MCP)  │  │ SQL warehouse  │
  │ draft orders    │    │ redeem           │  │                │  │ hosted agents  │
  │ payment, orders │    │ loyalty hub      │  │ profile        │  │                │
  │                 │    │                  │  │ events         │  │ propensity     │
  │ CATALOG TRUTH   │    │ ELIGIBILITY,     │  │ recommendations│  │ abandonment    │
  │ AND THE MONEY   │    │ DEPTH, STACKING  │  │                │  │ risk           │
  │                 │    │                  │  │ WHO THE        │  │ JUDGEMENT OVER │
  │                 │    │                  │  │ CUSTOMER IS    │  │ HISTORY        │
  │                 │    │                  │  │ AND DID        │  │                │
  └─────────────────┘    └──────────────────┘  └────────────────┘  └────────────────┘

  Both services run on Cloud Run, private, one warm instance each.
  Every hop is authenticated with a Google-signed ID token.
```

**The return path** — what happens after, so the next conversation is better:

```
   Shopper pays on Shopify
        │
        ├──► paid-order event (HMAC webhook, or a 30-second reconciler)
        │          │
        │          └──► UCP server ──► VOUCHERIFY   redeem the locked offers atomically
        │                                           (or record the PAID order, so the
        │                                            loyalty points still accrue)
        │
        └──► Shopify's own integration ──► BLOOMREACH   purchase event on the customer's profile

   VOUCHERIFY ──► BLOOMREACH      customer events: validation, redemption, loyalty

   BLOOMREACH ──► UCP server ──► DATABRICKS    Scenario webhook posts live events;
                                               the server forwards them into the
                                               Lakehouse landing table

   UCP server ──► DATABRICKS      every abandonment assessment, and the discount
                                  that was actually applied

   ══════════════════════════════════════════════════════════════════════════════
   Result: the next greeting, the next qualification and the next score all start
   from what happened in this conversation.
```

### The division of labour

This is the decision the whole design rests on:

| Layer | Owns | Never does |
|---|---|---|
| **Gemini Enterprise** | The surface. Renders the widgets. | Nothing commerce-specific. |
| **Otto agent** | The *conversation*: which phase it is in, which consents were given, which widget to show. Deterministic gates before every tool call. | Never computes a price, never picks a size, never invents an offer id. |
| **UCP commerce server** | The *cart*, the *offers*, **every amount**, the checkout link, and post-payment redemption. Emits ready-made presentation strings. | Never lets a model do arithmetic. |
| **Voucherify** | Eligibility, stacking, discount depth, loyalty liability. | — |
| **Bloomreach** | Who the shopper is, what the customer did, what to recommend. | — |
| **Databricks** | Predictions and judgement over historical behaviour; hosted decision agents. | — |
| **Shopify** | Catalog truth, the order, the money. | — |

The model never sees a percentage it has to multiply. It quotes the server's presentation strings
word for word. **A test in the suite enforces that no vendor name leaks into any tool name,
description, parameter hint or shopper-facing string** — the agent-facing surface is fully
vendor-neutral, and the backends are an implementation detail.

---

## 4. The five systems, and what breaks without each

The hackathon's test is: *if you deleted that system, would your agent still work?*

| System | Load-bearing call, at runtime | Delete it and… |
|---|---|---|
| **Bloomreach** (Loomi Connect MCP) | Customer lookup, customer properties, customer events and recommendations — on every identification. Scenario webhooks feed our live ingest route. | Otto no longer knows who the customer is or what the customer bought. The greeting, the complementary-product suggestions and the whole opening of the conversation disappear. |
| **Google** (Gemini Enterprise, Vertex AI, Cloud Run) | The reasoning model, the A2A surface that renders the widgets, and the hosting for both services. | There is no agent and no surface. |
| **Shopify** (Storefront + Admin GraphQL) | Product search, variants, prices, stock; draft-order creation with line discounts; the paid-order event. | No catalog, no cart truth, no order, no money. |
| **Databricks** (SQL Statement Execution API + agent serving endpoints) | A live purchase-propensity point lookup; a hosted agent invoked mid-conversation for abandonment risk; assessments and Bloomreach events written back. | Scenario 2 disappears entirely; Otto loses the ability to calibrate how hard to sell. |

And the system we brought ourselves:

| System | Load-bearing call, at runtime | Delete it and… |
|---|---|---|
| **Voucherify** | Qualifications per cart, dry-run validation, five-minute locks, atomic redemption after payment, Loyalty Hub. | Otto can only quote list price — i.e. the exact problem we set out to solve. |

All four partner platforms carry weight at runtime. Voucherify is the commercial heart of the
idea: without it there is nothing to negotiate with.

---

## 5. Scenario 1 — the guided purchase

**The story.** The customer bought a snowboard last season. The customer comes back and talks to Otto in
Gemini Enterprise. Otto recognises the customer, suggests the boots and bindings that go with the board,
spots that two items unlock a 70%-off jacket tier, asks once whether the customer wants to join the
loyalty programme, and hands the customer a checkout link that already carries every discount. The customer pays
on Shopify; the redemption and the loyalty points land afterwards.

### The exchange, step by step

```
 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  1. IDENTIFY                                                                      │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"Hi, I'm back" → email──►  Otto  ──identify(email)──►  UCP server
                                                                            │
                     ┌──────────────────────┬─────────────────────────┬─────┘
                     ▼                      ▼                         ▼
                BLOOMREACH             VOUCHERIFY                  SHOPIFY
           profile · purchase      customer profile +      price & availability
           history · recs          loyalty status          for each suggestion
                     │                      │                         │
                     └──────────────────────┴─────────────────────────┘
                                            │
   Otto ◄── name · history · loyalty status · priced suggestions · session ───┘
     │
     └─►  "Welcome back, Rosa — last time you bought The Collection Snowboard."
          + product carousel (boots, bindings)

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  2. CART, WITH A SIZE GATE                                                        │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"I'll take the Summit Pro Boa boots"──►  Otto
                                                        │  no size named → refuses to guess
   Shopper  ◄──────────── size buttons (UK 7–11) ───────┘
   Shopper  ──"UK 9"──►  Otto  ──add_item(product, variant)──►  UCP server
                                                                        │
                                  ┌─────────────────────────────────────┤
                                  ▼                                     ▼
                               SHOPIFY                             VOUCHERIFY
                        variant price + stock            qualifications for this cart
                                  │                          + dry-run validations
                                  └──────────────┬──────────────────────┘
                                                 ▼
                          applicable / unlocked / within-reach offers,
                                 each already PRICED BY THE ENGINE
                                                 │
   Otto ◄── cart + offers + presentation strings ┘
     │
     └─►  cart card  +  offer card: "Racing Jacket £135.00 instead of £450.00"

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  3. LOCK THE OFFER, ASK ABOUT LOYALTY (ONCE)                                      │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"Add the Racing Jacket in M"──►  Otto  ──update → lock(offerIds)──►  UCP
                                                                                    │
                                              VOUCHERIFY ◄── validate & lock (5 min)─┘
   Otto ◄── cart with discount lines, locked ──┘
     │
     ├─►  cart card
     └─►  loyalty card — asked ONCE, by code, never by the model. A "no" is final.

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  4. CHECKOUT AND COMMIT                                                           │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"Checkout"──►  Otto  ──checkout_url(session)──►  UCP server
                                                                   │
                                            SHOPIFY ◄──────────────┘  draft order with
                                                                      customer, address,
                                                                      line discounts
   Shopper  ◄── checkout card + link ──  Otto  ◄── invoice URL + total ──┘
     │
     └─►  pays on Shopify
                 │
                 ├──► paid-order event (HMAC webhook) ──► UCP ──► VOUCHERIFY
                 │    or the 30-second reconciler              redeem the locked offers
                 │                                             (or record the PAID order
                 │                                              so loyalty points accrue)
                 │
                 └──► Shopify's native integration ──► BLOOMREACH purchase event
                                                        THE LOOP CLOSES:
                                                        the next greeting is different
```

---

## 6. Scenario 2 — live cart-abandonment rescue

**The story.** The shopper has a full cart and starts to wobble: *"it's more than I planned"*,
*"I saw it cheaper elsewhere"*. Otto does not argue, and does not reach for a discount. It asks
Databricks. A hosted agent reads the customer's real purchase and abandonment history, weighs it against
what Otto observed in *this* conversation, and returns a probability. That probability is
attached to the order — and **Voucherify decides whether a discount applies and how deep**.
The shopper watches the check as an agent trace. The customer never sees the score.

### Why this is the interesting one

The naive version of this feature is "if the customer hesitates, offer 10% off". That is a
hardcoded if-statement, and it leaks margin to everyone who ever says "hmm".

Ours is a genuine decision with a genuine separation of powers:

| Question | Who answers it | How |
|---|---|---|
| *How likely is she to leave?* | **Databricks agent** (`otto-cart-abandonment-agent`, Agent Bricks) | One tool only: the Unity Catalog function `get_purchase_history`. Live conversation weighs most, history modulates. The server then throws the model's own label away and re-derives the verdict from the number: ≥ 0.7 *likely to leave*, < 0.4 *likely to buy*, otherwise *uncertain*. |
| *Does she get anything at all?* | **Voucherify** | Campaign *Cart Abandonment Prevention*, tier *Activate Your Purchase*. The rule: order metadata `abandonment_probability` must be above 0.7. Below that the tier is inapplicable, it never enters the offer registry, and the cart is unchanged. |
| *How much?* | **Voucherify** | `ORDER_METADATA("abandonment_probability") × 20`, a 16–20% band on the order. The merchant owns the ceiling. The server only formats the amount the engine returned. |
| *Does it stack?* | **Voucherify** | A dry-run first. If it applies, the tier is registered like any other offer and locked **last**, so the engine prices it on the subtotal net of the discounts already there. If it does not stack, it does not apply. |
| *What does the shopper hear?* | **Otto** | A progress line, then a Databricks trace on the same surface. If a discount was approved, one sentence supplied by code, then `presentation.totalLine`. Never a score, never a probability, never the word "risk". |

The tool exists only when the serving endpoint is configured. It runs once per session, and
again if the shopper's hesitation genuinely escalates; an unchanged repeat returns the stored
result instead of calling Databricks again. If the agent call fails or times out (60 seconds),
the verdict comes back *unknown* and the conversation carries on.

### The exchange, step by step

```
 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  1. THE CUSTOMER WOBBLES                                                          │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"It's more than I wanted to spend…"──►  Otto
                                                        │  reads the turn into a fixed schema:
                                                        │  hesitation_count · price_objection
                                                        │  comparing_alternatives
                                                        │  shipping_or_returns_concern
                                                        │  stated_intent · a one-line summary
   Shopper  ◄── "One moment, our Databricks agent is reviewing your cart
                 and purchase history to see what we can do for you."
              + a chip on its own surface: "Contacting Databricks…"

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  2. THE AGENT DECIDES                                                             │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Otto  ──assess(session, signals)──►  UCP server
                                           │  cart (items, subtotal, discounts already
                                           │  on it, age) + the live signals, as JSON
                                           ▼
                                    DATABRICKS AGENT
                                           │
                                           └── get_purchase_history(email) ──► UNITY CATALOG
                                                    │
                                                    ◄── purchases_90d · abandoned_checkouts
                                                        cart_to_checkout_rate · AOV · recency
                                           │
                                           │  { abandonment_probability: 0.86,
                                           │    drivers[], reasoning }
                                           │  one tool, JSON only, no write
                                           ▼
                                    UCP SERVER
                                      stores it on the session
                                      verdict = f(the number)

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  3. THE ENGINE DECIDES                                                            │
 └───────────────────────────────────────────────────────────────────────────────────┘
   UCP  ──dry-run validate "Activate Your Purchase"──►  VOUCHERIFY
            order.metadata.abandonment_probability = 0.86
                 │
                 ├── not applicable (≤ 0.7, or it will not stack)
                 │     cart untouched · rescue id never registered
                 │     the row is still logged, with a null discount
                 │
                 └── applicable
                       depth from the merchant's formula (0.86 × 20 → 17.2% of the net)
                       register the tier · lock it LAST, after the offers already held
                       a later quantity change re-locks it with the rest of the stack

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  4. THE CUSTOMER SEES THE TRACE, NOT THE SCORE                                    │
 └───────────────────────────────────────────────────────────────────────────────────┘
   The finished card reuses the chip's surface, so the wait becomes the trace:

        Databricks agent          Completed in 19.4 s · 3 steps
        ─────────────────────────────────────────────────────────
        ✓  Reviewed your cart              3 items · £289.95
        ✓  Checked your purchase history   Unity Catalog · get_purchase_history
        ✓  Matched eligible promotions     Promotion rules for this order
        ─────────────────────────────────────────────────────────
        ★  Additional 17.2% discount approved
           Applied to this order: you save a further £xx.xx
           New total, from presentation.totalLine

   Otto ◄── the sentence, from code ── "Good news: our Databricks agent reviewed your
            cart and purchase history and approved an additional 17.2% discount
            (£xx.xx), which is already applied to your order."
            Then the total, word for word.
     │
     └─►  the cart card redraws underneath. No loyalty question is stacked on this turn.

   If the engine said no, the same three steps end on
   "No additional discount available for this order right now."
   One short sentence, and the cart the customer already had.

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  5. IT SURVIVES PAYMENT, AND THE ROW IS WRITTEN                                   │
 └───────────────────────────────────────────────────────────────────────────────────┘
   Shopper  ──"Checkout"──►  the same Shopify draft order as Scenario 1,
                             customer, address, and the rescue already on the lines
     │
     └──► pays
            │
            ├──► paid-order event ──► UCP ──► VOUCHERIFY
            │    the probability rides on the order again: the rule is checked
            │    at redemption, not only at the dry-run
            │
            └──► DATABRICKS (fire-and-forget, from the server, not from the agent)
                   MERGE into cart_abandonment_assessments:
                   probability · verdict · drivers · reasoning · signals
                   · cart value · AND the discount cents actually given (or null)
                   ─────────────────────────────────────────────────────────────
                   real, labeled examples for a future scoring run to learn from
```

### Signal → reason → act → close the loop

- **Signal.** Bloomreach Scenario webhooks post live events to the commerce server, which forwards
  them into a Databricks landing table — fire-and-forget, because telemetry must never block a
  shopper's turn. Batch scoring turns those into `customer_predictions_live`. A separate, fast
  point lookup (`ucp_get_purchase_probability`) lets Otto calibrate *how hard to sell* before
  anyone hesitates. That lookup is not the rescue. The rescue is the hosted agent, called only
  when the cart is full and the customer pushes back.
- **Reason.** The hosted agent decides. Per customer, a different number. The server keeps the
  number and discards the label.
- **Act.** A real tier is locked on a real cart, inside the merchant's rule and formula, and it
  survives a cart edit, the Shopify draft order, and the redemption after payment.
- **Close the loop.** The assessment row records the probability, the drivers, the reasoning,
  the signals, the cart value **and the discount that was actually applied** — null when the
  engine said no — real, labeled examples for a future scoring run to learn from.

---

## 7. Scenario 3 — weekly opportunity agent to Vincent

**The story.** Nobody is in the conversation. A Databricks-hosted agent wakes on a weekly
schedule, reads the store's aggregate behaviour through its own tools, and decides — on its
own, with no human in the loop — whether there is money on the table. If there is, it writes a
brief: *who* should get a promotion and *why*. Vincent, Voucherify's AI promotion agent, picks
that brief up, drafts the cohorts, and leaves a thread waiting for a human to read. The marketer
arrives to a decision already made and two agents' reasoning already on the page — and can keep
talking to it.

This is the scenario that answers the hackathon's "does it start without you?" question in the
strongest way: **no human, no conversation, no trigger from the product surface.**

### 7.1 The pipeline

```
   ┌──────────────────────────────┐
   │  DAILY REPORT JOB            │   aggregates the raw event data
   │  (Databricks Workflow)       │   → one identifier-free row per day
   │  → daily store statistics    │   → behavioral_report_daily
   └──────────────┬───────────────┘
                  │
                  ▼
   ┌──────────────────────────────┐
   │  OPPORTUNITY AGENT           │   weekly schedule, or "Run now"
   │  (Agent Bricks Supervisor)   │
   │                              │   TWO Unity Catalog tools, both mandatory:
   │  System prompt, one job:     │   · snapshot — current 7/30/90/180-day windows
   │  read the evidence, decide   │   · trend    — six months of dated checkpoints
   │  whether a promotion would   │
   │  make money, and if so       │
   │  say WHO and WHY             │
   └──────┬────────────────┬──────┘
          │                │
      NO_ALERT        NOTIFY_VINCENT
          │                │
          ▼                ▼
   ┌─────────────┐   ┌───────────────────────────────┐
   │  STOPS.     │   │  BRIEF                        │
   │  Nothing    │   │  who to promote to, and why   │
   │  is sent.   │   │  aggregates only — no PII     │
   │             │   └──────────────┬────────────────┘
   │  ← this is  │                  │  POST /proactive/signals
   │  a REAL     │                  ▼
   │  decision   │   ┌───────────────────────────────┐
   └─────────────┘   │  VINCENT                      │
                     │  Voucherify's AI promotion    │
                     │  agent — marketing node runs  │
                     │  forced into exploratory mode │
                     │  → drafts COHORT PROPOSALS    │
                     │  → sizes them on Bloomreach   │
                     └──────────────┬────────────────┘
                                    │
                                    ▼
                     ┌───────────────────────────────┐
                     │  UNCLAIMED THREAD             │
                     │  · message 1: Databricks agent│
                     │  · message 2: Vincent         │
                     │  stopped at cohort_selection  │
                     └──────────────┬────────────────┘
                                    │
                                    ▼
                     ┌───────────────────────────────┐
                     │  A HUMAN claims it, picks a   │
                     │  cohort, continues the thread │
                     └───────────────────────────────┘
```

### 7.2 The Databricks side

**The daily report job.** It builds an aggregate commercial snapshot from the raw customer
events — sessions, product views, cart updates, checkouts, purchases. For each of the 7-, 30-,
90- and 180-day windows it computes viewers and views, high-intent customers (5+ product views,
or any cart update, or any checkout), how many of those bought nothing, the conversion rate,
cart and checkout abandoners, purchase revenue, abandoned cart and checkout value in GBP, the
desktop/mobile split, the products high-intent nonbuyers look at most, and aggregate K-means
cluster profiles. It writes **one row per day** into `behavioral_report_daily` — no emails, no
customer ids, no individual events.

It is meant to run daily, but it can be started at any time with **Run now**.

**The opportunity agent.** Built in Databricks **Agent Bricks** as a Supervisor Agent called
*Behavioral Opportunity Analyst*: natural-language instructions plus two governed Unity Catalog
functions as tools. Agent Bricks hosts it, traces it, and exposes a serving endpoint the weekly
job can call. The tools are SQL functions over the daily report table — a current snapshot across
the 7/30/90/180-day windows, and a six-month trend of dated checkpoints with the change versus 30,
90 and 180 days ago. The instructions require both to be called before a decision.

What it looks for is the case where purchase intent holds up or grows while conversion falls,
and whether that is persistent rather than a one-week spike. It is told not to invent causes it
has no evidence for, not to treat value sitting in carts as guaranteed revenue, and not to sum
overlapping metrics. It never sees personal data: only thresholds, counts, rates, products,
devices and amounts. For targeting it may use five fixed segment labels synchronised with the
Bloomreach profiles, described in plain marketing language — never column names, EQL or cluster
numbers.

**The weekly job.** Runs Mondays at 09:00, and on demand. It performs no analysis of its own: it
wakes the agent, and then carries whatever the agent decided. If the agent concludes there is
nothing worth acting on, nothing leaves Databricks. If it finds an opportunity, it writes a brief
the way one specialist writes to another — what changed, who it describes, why a campaign could
pay — and hands it to Vincent. Vincent reads it, decides the case is worth acting on, and drafts
a proposal: several concrete cohorts, each sized against real customer data, and the incentives
that could bring the money in — a discount, a promotion, a loyalty reward — ready for a
marketer to approve, adjust or reject. Two agents have read, reasoned and proposed. Nothing has
been created or discounted — that happens only after a human says yes. Every run, whether
it ended in a brief or in silence, is kept as an audit record.

The notebooks, the Supervisor instructions, and the two SQL tools are in
[`databricks/behavioral-opportunity/`](databricks/behavioral-opportunity/README.md).

**The decision is real.** The agent is allowed — and expected — to conclude *no* and produce
nothing. A run that outputs a brief every time would be a report generator, not an agent.

### 7.3 The Vincent side

**Vincent** is Voucherify's AI promotion agent: it turns a commercial opportunity into
concrete, reviewable promotion work inside Voucherify.

**What arrives.** A message from another agent: the Databricks analyst explaining what changed,
who it describes, why a campaign could pay, and asking Vincent to propose several concrete
cohorts — without picking the audience itself. That message is the **first visible turn** of a
new organisation-wide thread, attributed to the Databricks agent.

**The marketing node.** It runs in **exploratory mode**: it turns the described characteristics
into concrete cohort candidates, validates each one against Bloomreach so the sizes are real,
replies in the thread, and **stops at the `cohort_selection` gate**. It picks no audience and
creates nothing in Voucherify at this stage.

**What is left behind.** A **thread with two messages from two agents** — the Databricks
analyst's request and Vincent's drafted cohorts — sitting unclaimed. A marketer picks it up and
can **continue the conversation** with Vincent from there.

---

## 8. The engineering decisions that make it trustworthy

These are the parts we would defend in a code review, not just in a pitch.

**1. The model never touches money.**
Every amount the shopper hears is a presentation string built by the server from an engine
response. Money is carried in minor units alongside its formatted string, and formatted in
exactly one place. The agent's instructions say *quote it word for word*. No percentage is ever
multiplied model-side.

**2. Offers are validated before they are spoken.**
The offers engine turns Voucherify qualifications into three buckets — *applicable* (already
pricing this cart), *unlocked* (conditions met, product not in the cart yet) and *within reach*
(one numeric condition short) — and every amount comes from the engine or from a dry-run
validation. Locking accepts **only** offer ids the session itself returned, so an invented offer
cannot be locked.

**3. The hard rules are code, not prompt.**
The agent's gates run before every tool call, are deterministic, and each one has a negative
test: identify before anything touches the cart; never pick a size the shopper did not name or
press; no checkout link without a checkout request in the current turn; no loyalty enrolment
without a yes in a *later* turn than the question; no server-side order placement at all — the
tool that would do it is filtered out of the agent's toolset entirely.

**4. Redemption happens after payment, once.**
The checkout link is a Shopify draft order tagged with the session id. Redemption is triggered by
the paid-order webhook, HMAC-verified — or, where Shopify cannot reach a private service, by a
30-second reconciler. Both paths are idempotent per session. A cart with no offers still gets its
paid order recorded in Voucherify so loyalty points accrue.

**5. Two sides, one schema.**
Response contracts are written down as JSON Schema. The Python agent's test fixtures are *real
captured server responses*, validated against those same schemas — so the TypeScript server and
the Python agent cannot drift apart silently.

**6. Degradation is designed, not accidental.**
Bloomreach token expired? Otto still identifies the shopper from Shopify and Voucherify, minus
history. No loyalty programme configured? The tools report it as unavailable and loyalty is
never mentioned. Databricks agent times out? The verdict comes back *unknown*, and the
conversation carries on with normal objection handling. No path fails loudly at the shopper.

---

## 9. Business impact

| Lever | Mechanism | Who benefits |
|---|---|---|
| **Basket growth** | Unlocked and within-reach offers are surfaced conversationally: *"add the bindings and the jacket drops to £135"*. The shopper is told what the customer is one step away from. | Both |
| **Margin governance in the agentic channel** | Discount depth, eligibility and stacking stay in Voucherify. The agent can only offer what the engine validated for this shopper and this cart. There is no blanket code for an agent to leak. | Merchant |
| **Discount efficiency** | The rescue discount is gated on a probability above 0.7 and scales with it. Shoppers who were going to buy anyway are not paid to buy. | Merchant |
| **Cart recovery in the moment** | Recovery happens *during* the session, not in an email 24 hours later when the shopper has already bought elsewhere. | Both |
| **Loyalty as a conversation** | Enrolment happens mid-purchase, once, with consent, with the earning preview shown from the real programme rules. Points are the means; the repeat purchase is the outcome. Joining with a real earning figure in hand, in the middle of buying, is the moment a programme changes the next purchase — not the inbox. | Both |
| **Compounding data** | Assessments, outcomes, purchases and redemptions flow back to the Lakehouse and to the Bloomreach profile. The next conversation opens differently because of this one. | Merchant |

---

## 10. Self-check against the submission bar

| The five questions | Our answer |
|---|---|
| **Name your three systems. Would removing any single one break the workflow?** | Five: Bloomreach, Google, Shopify, Databricks, Voucherify. §4 gives the load-bearing runtime call for each and what breaks without it. None is decorative. |
| **Does it start without you?** | Scenario 2 is triggered by the shopper's own hesitation, mid-conversation, with no human deciding to run anything. Scenario 3 is a scheduled Databricks job — no human, no conversation, no product surface. Scenario 1 is user-initiated by nature, and hands off to webhook- and reconciler-driven completion afterwards. |
| **Does it make a decision?** | The Databricks agent produces a *different probability per customer* from history plus live conversation, and Voucherify then decides eligibility and depth from merchant rules. The opportunity agent is free to decide there is no opportunity, and write nothing. Neither is an if-statement. |
| **Does something change?** | Shopify draft orders and paid orders; Voucherify validations, locks, redemptions and loyalty enrolments; Bloomreach customer events; Databricks rows for both ingested events and scored assessments; Voucherify cohort drafts via Vincent. |
| **Does the result feed back?** | Purchases reach the Bloomreach profile, so the next greeting differs. Redemptions and points land in Voucherify, so the next qualification differs. Every assessment and the discount it produced land in the Lakehouse — real, labeled examples for a future scoring run to learn from; no batch job reads them back yet. |
---