# H200 rental: hourly vs monthly real costs, how many GPUs your model needs, and when a $3.50/GPU-hour B200 changes the math (Vultr GPU pricing and $300 credit terms)

On-demand H200 capacity lists at roughly **$3.59 to $5.29 per GPU-hour**. A dedicated single card on a monthly plan runs $2,772 to $3,090, and a full 8-GPU HGX H200 node sits near $23,900 a month. Whether any of those is a good deal comes down to two numbers most renters never calculate: how many hours a month the card will actually run, and how much of the 141GB your KV cache swallows.

Both are answerable before you pay anyone. What follows is the per-hour and per-month math done out, a table for deciding how many cards a given model needs, the dense-versus-sparse spec trap that catches people comparing spec sheets, and where Vultr's GPU lineup fits if what you want is 180GB per GPU rather than 141GB.

> **Quick pick.** One H200 serving a 70B model in FP8, or running QLoRA: rent hourly if you'll use under roughly 580 hours a month, take the monthly single-card plan above that. Need 180GB per GPU, FP4, or the lowest per-GPU-hour currently listed: Vultr's 8× HGX B200 comes in at **$3.50 per GPU-hour**, which is $28.00 an hour for the node and about $20,440 a month if it never idles. New accounts also get a **$300 promotional credit** that expires 30 days after it's issued.
> 👉 [See Vultr's current GPU configurations and credit terms](https://bit.ly/VuLtr)

## What Vultr actually lists for GPU work

Vultr's published GPU lineup is built around whole-node Blackwell capacity and single-GPU cloud instances. An H200 configuration does not appear in it, so if H200 specifically is a hard requirement, the market table further down is where you shop. If the real requirement is "more than 141GB per GPU at the lowest hourly rate", the B200 node is the more interesting product.

| Configuration | What you get | Listed price | Billing | Entry point |
| --- | --- | --- | --- | --- |
| Cloud GPU (entry tier) | Single-GPU cloud instances, model varies by region | from **$1.67/hr** | Hourly | [Check availability](https://bit.ly/VuLtr) |
| 8× HGX B200 | 8 × 180GB HBM3e, sold as one node | **$3.50/GPU-hr** ($28.00/hr for the node) | Hourly | [Check current node pricing](https://bit.ly/VuLtr) |

Two things to know before you budget against that table. GPU catalogs at every provider rotate by region and month, so confirm the model and the rate on the live pricing page rather than on a comparison site. And there is no free tier at Vultr; the $300 credit is the trial mechanism, not a permanent free plan.

**The credit terms matter more than the number.** It applies to new accounts, requires a verified payment method, and runs out 30 days after issue. Put against a B200 node at $28.00 an hour, $300 buys about **10.7 hours** of an 8-GPU machine. That's enough for a benchmark run or a short fine-tune, not enough to be a free month of GPU work. Whether GPU instances draw on the credit is also not spelled out in Vultr's public materials, so confirm it with support before you plan a test around it.

## How many H200s does your model need?

The 141GB figure is the whole decision. Weights scale at roughly 2 bytes per parameter in BF16, 1 byte in FP8, and 0.5 bytes at 4-bit, with KV cache and 10–30% headroom on top.

| Model | Precision | H200s needed | Why |
| --- | --- | --- | --- |
| 70B dense | FP8 | **1** | ~70GB of weights leaves 50–60GB for KV cache |
| 70B dense | BF16 | **2** | ~140GB of weights alone, too tight for one card |
| 405B dense | FP8 | **4** | ~405GB of weights; an 8-card node leaves ~700GB spare |
| 671B MoE (DeepSeek-V3/R1 class) | FP8 | **8** | ~671GB of weights, ~450GB left over |
| 671B MoE | 4-bit | **4** | ~335GB of weights |

Memory bandwidth explains why this card exists at all. Token generation is limited by how fast weights and KV cache move, not by FLOPs, and 4.8TB/s against the H100's 3.35TB/s is a 1.4× advantage on exactly that path. Capacity sets your concurrency ceiling: a Llama-style 70B with grouped-query attention needs about 0.33MB of KV cache per token in BF16, so a single 128k-token context costs roughly 43GB. That fits alongside FP8 weights on an H200 and does not fit on an 80GB card.

Fine-tuning splits the same way. QLoRA on 70B, at about 35GB of 4-bit weights, is comfortable on one card. LoRA in BF16 is not, because the frozen weights alone are ~140GB. Full fine-tuning with Adam needs 16–20 bytes per parameter before activations, which is 512–640GB for a 32B model and 1.1–1.4TB for a 70B model. The 8-GPU node holds 1.1TB of HBM3e, so 70B full fine-tuning spills optimizer state into the 2TB of system RAM or doesn't run.

## Hourly or monthly: the break-even calculation

Monthly plans are quoted on the assumption the card runs all 730 hours in a month. Divide the monthly price by a competitor's hourly rate and you get the number of hours you need to use before the monthly plan wins.

- $3,090/month against a $5.29/hr secure-cloud rate: **584 hours** to break even. Run more than that and monthly is cheaper.
- $3,090/month against a $3.59/hr community-cloud rate: **861 hours**. There aren't 861 hours in a month, so hourly wins at any utilization.
- 220 business hours on the $3,090 plan works out to **$14.05 per hour actually used**. A team serving a 9-to-5 internal endpoint is paying quadruple the advertised rate.

That last figure is where most rental budgets go wrong. A 7×24 production endpoint justifies a monthly commit. A model you benchmark on weekday afternoons does not.

There's a second effect worth knowing. Eight single-GPU plans at $3,090 each total $24,720, which is **more** than the $23,900 8-GPU HGX server, and the singles have no NVLink between them. If you need tensor parallelism across cards, the node is both cheaper and the only option that keeps the 900GB/s interconnect. Split the model across separate single-GPU rentals and you're limited to pipeline passing over the network.

Prices are also moving up. A survey of public H200 price lists in October 2026 put the median on-demand rate roughly **27% higher per GPU-hour** than a year earlier, which is the argument for locking a monthly plan if you already know you're staying.

## H200 market rates, side by side

| Provider | Listed rate | Basis and date |
| --- | --- | --- |
| OverGPU | $3,090/mo single card; $23,900/mo 8-GPU HGX | Monthly, works out to $4.23 and $4.09 per GPU-hour at 730 hours; 3 Nov 2026 |
| Akash | $3.85/GPU-hr on demand; $2,772/mo for 1×; $22,176/mo for 8× | Per GPU-hour, 720-hour month; 22 Oct 2026 |
| RunPod | $3.59/hr Community Cloud; $5.29/hr Secure Cloud | Per GPU-hour on demand; 27 Aug 2026 |
| Runcrate | from $5.25/hr, range $5.25–7.70 | Per-second billing, three regions; 8 Nov 2026 |
| Vast.ai | No published unit price; claims 5–6× cheaper than traditional cloud | On-demand, interruptible and reserved tiers |

Vultr's B200 node at $3.50 per GPU-hour undercuts every H200 rate in that table, and 8 × 180GB is 1.44TB of HBM3e against 1.1TB on an 8× H200. That comparison is not free, though. Most providers selling B200 at that price require the full 8-GPU node, so the entry ticket is $28.00 an hour whether your workload uses one card or eight.

## Five things that go wrong when comparing H200 prices

**FP16 specs get quoted sparse.** You'll see the H200 listed at 1,979 TFLOPS FP16 and at 989 TFLOPS FP16 on different pages. Both are correct: 989.5 is dense, 1,979 is sparse, and NVIDIA's own dense figure is the sparse number halved. The H200 has the same compute as the H100 — 1,979 TFLOPS FP8 dense for both — so any comparison that shows the H200 ahead on raw compute is comparing two different columns.

**Single cards have no NVLink.** NVLink 4 runs at 900GB/s per GPU inside an HGX node. Two separate single-GPU rentals talk over Ethernet. If your plan involves tensor parallelism, budget for the node.

**Monthly plans punish part-time use.** Covered above: $14.05 per real hour at 220 hours. Hourly billing, per-second at some providers, is the correct choice for anything below the break-even point.

**70B in BF16 doesn't fit one card.** 140GB of weights against 141GB of memory leaves nothing for KV cache. FP8, or two cards.

**Sizing tables assume inference.** Training has an optimizer-state multiplier on top of weights, so a model that serves comfortably on one card can need eight to fine-tune. The 8-card node covering a 32B full fine-tune at 512–640GB is the useful reference point.

## Which one to rent

Serving a 70B model in FP8 with long context, or running QLoRA: **one H200**, hourly until you cross roughly 580 hours a month, then a monthly single-card plan. Add `--kv-cache-dtype fp8` to the vLLM launch and the FP8 KV cache roughly doubles your concurrency on the same 141GB.

Serving 405B or a 671B MoE, or training across cards: **the 8-GPU node**, both for the 1.1TB of pooled memory and because tensor parallelism needs NVLink. At that size, Vultr's 8× B200 is worth pricing first, since $3.50 per GPU-hour and 180GB per card come out ahead of every H200 node rate listed above.

Running a few dozen GPU-hours to benchmark: **hourly, per-second billing if available**, and skip the monthly commitment entirely.

Two things to confirm at checkout, regardless of where you land. Whether the $300 credit applies to GPU instances, and whether billing is per-second or rounded to the hour. A 10-hour benchmark run billed by the hour costs the same as a 10-hour run billed by the second; a 40-minute one does not.

👉 [Start with Vultr's GPU pricing page and check the $300 credit terms](https://bit.ly/VuLtr)
