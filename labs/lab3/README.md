# Practice 3

**Student:** Iryna Velmakina · **Platform:** AWS Academy guided lab · **Account:** AWS Academy sandbox · **Region:** us-east-1 · **Date:** 2026-11-22
**Bucket:** `inventory-123123`

An inventory pipeline with no servers: a CSV dropped into S3 triggers a Lambda that loads
it into DynamoDB; a second Lambda listens to the table's stream and emails through SNS
when an item hits zero. Built by hand in the console, so the evidence is screenshots —
cropped to the console pane, with account ID, ARNs and email redacted.

---

## 0. Architecture

![Architecture](img/00-architecture.png)

| step | trigger | action |
|---|---|---|
| 1 | store uploads `inventory-<city>.csv` | object lands in `inventory-123123` |
| 2 | S3 `ObjectCreated` event | invokes **Load-Inventory** |
| 3 | Load-Inventory | `put_item` per CSV row into **Inventory** (key `Store` + `Item`) |
| 4 | DynamoDB Stream on Inventory | invokes **Check-Stock**; if `Count == 0` → publish to **NoStock** |
| 5 | SNS | emails the confirmed subscriber |
| — | dashboard (static site on S3) | reads Inventory directly with an anonymous Cognito identity |

---

## Task 1 — Load-Inventory Lambda

### Setup

| setting | value |
|---|---|
| function | `Load-Inventory` |
| runtime | Python |
| execution role | `Lambda-Load-Inventory-Role` (S3 read, DynamoDB write) |
| code | [`code/load_inventory.py`](code/load_inventory.py) |

**File → Save** does not update the function — only **Deploy** does. Until then the
status bar says *Undeployed Changes* and S3 would invoke the old "Hello from Lambda" stub.

### Result

![Load-Inventory deployed](img/01-load-inventory-deployed.png)

---

## Task 2 — S3 bucket and event notification

### Setup

The first `Create bucket` failed:

> A conflicting conditional operation is currently in progress against this resource.

This is **not** the "name taken" error (that one reads *The requested bucket name is not
available*). It means another operation on a bucket of the same name was still in
flight — typically a double-submitted Create, or a same-named bucket deleted moments
before. Retrying succeeded.

![Bucket creation conflict](img/02-bucket-name-conflict.png)

![Bucket created](img/03-bucket-created.png)

### Event notification

| field | value |
|---|---|
| name | `Load-Inventory` |
| event types | All object create events (`s3:ObjectCreated:*`) |
| destination | Lambda → `Load-Inventory` |

![Event notification](img/04-s3-event-notification.png)

The console also adds a resource-based policy on the function allowing
`s3.amazonaws.com` to invoke it — without that, S3 would have the event but no right to
deliver it.

---

## Task 3 — Testing the load

### Input — `samples/inventory-berlin.csv`

```csv
store,item,count
Berlin,Echo Dot,12
Berlin,Echo (2nd Gen),19
Berlin,Echo Show,18
Berlin,Echo Plus,0
Berlin,Echo Look,10
Berlin,Amazon Tap,15
```

![Upload Berlin](img/05-upload-berlin.png)

### Result

Dashboard reads from DynamoDB, so data here proves the whole S3 → Lambda → DynamoDB chain:

![Berlin dashboard](img/06-dashboard-berlin.png)

Same six items in the table:

![DynamoDB items](img/07-dynamodb-items.png)

`Echo Plus = 0` is highlighted. **No alert was ever sent for it** — see Task 6.

---

## Task 4 — SNS topic and subscription

### Setup

| resource | value |
|---|---|
| topic | `NoStock`, type Standard |
| subscription | protocol Email → personal address |

![SNS topic](img/08-sns-topic.png)

![SNS subscription](img/09-sns-subscription.png)

The subscription starts as *Pending confirmation*. SNS delivers nothing until the
**Confirm subscription** link in AWS's email is clicked — opt-in is mandatory so a topic
can't be used to spam arbitrary addresses. The screenshot predates the click; the alert
emails in Task 6 are the proof it was confirmed.

---

## Task 5 — Check-Stock Lambda

### Setup

| setting | value |
|---|---|
| function | `Check-Stock` |
| execution role | `Lambda-Check-Stock-Role` (`sns:Publish`, stream read) |
| code | [`code/check_stock.py`](code/check_stock.py) |
| trigger | DynamoDB → `Inventory` (stream) |

![Check-Stock trigger](img/10-check-stock-trigger.png)

`Triggers (0)` in the lower panel is stale UI — the banner above confirms the trigger
was added.

### Why a second function instead of extending Load-Inventory

Load-Inventory loads; Check-Stock judges. The stream decouples them: a "low stock"
rule at `count < 5`, an audit log, a reorder bot — each is one more stream consumer, and
none of them touches the loader.

---

## Task 6 — End-to-end test

### Single file — Calcutta

![Two stores](img/11-dashboard-two-stores.png)

![Calcutta email](img/12-email-calcutta.png)

### Three files at once — Pusan, Shanghai, Springfield

![Three files uploaded](img/13-upload-three-files.png)

![Five stores](img/14-dashboard-five-stores.png)

![Three alerts](img/15-emails-parallel.png)

### Results

| upload | stores on dashboard | alert received |
|---|---|---|
| Berlin (before Check-Stock existed) | 1 | **none** — `Echo Plus = 0` |
| Calcutta | 2 | Calcutta / Echo (2nd Gen) |
| Pusan + Shanghai + Springfield (one upload) | 5 | Pusan / Echo Show · Shanghai / Amazon Tap · Springfield / Echo Show |

### Question — what happens when several files are uploaded at the same time?

S3 emits **one event per object**, so three files mean three independent Lambda
invocations running concurrently. All three loaded, the dashboard went from 2 to 5
stores, and three alerts arrived within the same minute.

### Question — why was there no alert for Berlin?

Berlin was loaded before Check-Stock and its trigger existed. A DynamoDB Stream trigger
reads from the point it's attached; it does not replay earlier writes. The zero was
already in the table and nothing ever looked at it. **Order of setup is part of the
system's behaviour.**

---

## Files

```
lab3/
├── README.md
├── code/
│   ├── load_inventory.py     S3 event  → parse CSV → DynamoDB put_item
│   └── check_stock.py        DDB stream → Count == 0 → SNS publish
├── samples/
│   └── inventory-berlin.csv  sample input
└── img/                      console screenshots, redacted
```
