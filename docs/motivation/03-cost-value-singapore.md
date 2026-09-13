# Cost, Value, and the Reality of Compute in Singapore

*Foundational document. The money argument, what the cluster cost to build, what
it costs to run, what it's worth, and how that sits against the actual state of
compute in Singapore in 2026.*

All figures are current as of **July 2026** and, where they change quarterly (the
electricity tariff especially), I've dated them so a future reader knows to
re-check.

## 1. Capital cost: what it cost to build

The defining feature of my build is that most of the hardware I already owned or
acquired cheaply, so my *marginal* capital outlay was low. But to make the value
argument honest, it's worth pricing what this cluster would cost someone building
it from scratch in Singapore today.

### What I actually spent

I wouldn't say this is a cheap project to make. But honestly, I believe this is one
of the cheaper hobbies that you can have, and you at least end up with actualy tangible
compute power.

| Item | Notes | Approx. cost |
|---|---|---|
| deus M910q Lenovo Thinkcentre Mini-PC | Bought online, i7-6700T + 16GB Ram | $280 |
| nebula Used custom gaming pc | Bought from a seller in Carousell, came with i7-6700K, 24GB Ram and a GTX 1080| $270 |
| legion Lenovo Legion gaming laptop | Used to belong to Red Hat, sold off for free in their company marketplace | $0 |
| Longhorn NVMe drives (3 drives, 256GB) | **Free** used, from company IT (pretty good quality) | S$0 |
| legion's WD Red SA500 500GB SATA | NAS-rated, for the Longhorn partition | ~S$103 |
| ZFS NAS drives (nebula) | 2 x 3.6TB in a mirror | $400 |
| Networking (switch, cables, MoCA) | Omada switch, Cat6, MoCA over coax | ~S200 |
| Misc (thermal pads, USB sticks, etc.) | | ~S$30 |

The total cost is about **S\$1283**, not too bad at this day and age with ballooning 
memory and storage prices, and the single most important storage component 
(the Longhorn drives) was free.

> **Note (September 2026):** these figures cover the original three nodes. Two more
> control-plane nodes, sol (Intel NUC) and opus (HP EliteDesk 800 G4 Mini), joined
> in August. Their purchase price and power draw are not in this document yet.

### What it would cost to build fresh

If someone replicated this from nothing, at Singapore retail (July 2026):

| Item | Qty | Unit (S$) | Subtotal (S$) |
|---|---|---|---|
| N100 mini-PC (16GB / 500GB, new) | 3 | ~400 | 1200 |
| *or* refurb USFF (EliteDesk / ThinkCentre, Carousell) | 3 | ~300 | 900 |
| Managed switch (TP-Link SG108E class) | 1 | ~40 | 40 |
| NAS-rated SSD / drives for storage | N/A | N/A | ~600 |
| Cables, USB sticks, surge strip, misc | N/A | N/A | ~80 |
| **Total (new mini-PCs)** | | | **~S$1920** |
| **Total (refurb path)** | | | **~S$1620** |

So a from-scratch equivalent lands right around the original **S$1,000 budget**
under it on the refurb path. That's the capital number to hold in your head:
**a complete, capable multi-node K3s platform for roughly one thousand Singapore
dollars, once.** 

## 2. Running cost: electricity is the only recurring line

Once built, the cluster's only ongoing cost is power. And Singapore's electricity
price matters a lot here, because it's currently at a **record high.**

### The tariff (dated, re-check quarterly)

For **1 July - 30 September 2026**, SP Group's regulated household tariff is
**31.91 cents/kWh before GST, or 34.78 cents/kWh with 9% GST**: the highest in
Singapore's history, up 17% from the previous quarter, driven by higher natural
gas costs. It's reviewed every quarter, and EMA has flagged Q4 could ease *or*
rise further depending on global fuel prices. For this analysis I use the all-in
**S$0.348/kWh.**

### The draw

The recurring cost is the **always-on** draw. The original three nodes all run
24/7 (sol and opus are not measured yet):

- **deus** (M910q): idles low, ~15W.
- **nebula** (i7-6700K + GTX 1080): the heavy one; higher idle ~50W, more under load.
- **legion** (laptop, battery removed): modest, ~16W.

Realistic always-on total: **~80W idle, ~130W with moderate workload.**
Do note that the high idle is due to overhead power consumption. The overhead is due to
motherboard and other components (GPU is your worst nightware here), not the CPU 
(especially the case with nebula, with only 5W CPU idle).

### The monthly bill

- **At idle (~80W):** 80W x 24h x 30d = 57.6 kWh, so **~S$20/month**
- **With moderate load (~130W):** ~94 kWh, so **~S$33/month**

So realistically **S$25-33/month**, call it **~S$300-400/year.** deus alone is
almost free (~S$4/month); nebula with the 1080 is the main contributor.

### The one lever

The GPU-heavy draw on nebula is the part of the bill that moves. When the GPUs
aren't working, that wattage is waste. Historically I had `cluster-shutdown.sh` /
`cluster-powerup.sh` + WoL to spin heavy nodes down, though now that legion and
nebula are deliberately always-on for storage-replica and NAS reasons, I've traded
some of that saving for availability. That's a conscious tradeoff: a few dollars a
month for a cluster that's actually up when I want to build on it.

## 3. Value: what the cluster is worth

Here's where the numbers become lopsided in the cluster's favour.

### The cloud-equivalent bill

Mapping my nodes to the nearest AWS instances (a GPU-worker-heavy cluster like
mine maps to `g4dn.xlarge`-class GPU instances plus a `t3.xlarge`-class control
node, with EBS gp3 for storage):

- On-demand, this lands around **US\$970/month, roughly US$11,600/year.**
- On 1-year reserved pricing it drops to roughly **US$7,000/year.**
- And that's *before* the Singapore-region premium: `ap-southeast-1` runs
  meaningfully higher than the `us-east-1` reference (a `g4dn.xlarge` is ~US$0.526/hr
  in us-east-1; Singapore is higher still), so the real local figure is worse, not
  better.

### The comparison that matters

- **Capital:** my whole build (or a ~S$1,000 from-scratch equivalent) costs **less
  than a single month** of the AWS-equivalent on-demand bill.
- **Running:** ~S\$20-33/month of electricity vs. ~US$970/month of cloud compute.
  The homelab's *entire annual running cost* is roughly what the cloud equivalent
  bills in **a week and a half.**
- **Over a year:** owned hardware total (build + power) sits around **S\$1,300-1,400
  all-in**; the cloud equivalent is **US$7,000-11,600+.** Roughly an
  **order of magnitude** difference.

### But value isn't only the dollar delta

The cloud bill buys things my cluster doesn't have, genuine redundancy, 25Gbit
networking, managed durability, and no 3am debugging of a flapping MoCA link. I'm
not paying that premium because, for a *learning cluster*, I don't need any of it.
So the real value has two parts:

1. **The compute-per-dollar**, which is overwhelmingly in the homelab's favour for
   a steady always-on workload.
2. **The learning and portfolio value**, which is the actual point. The cluster is
   a standing, defensible demonstration of platform engineering, every
   post-mortem, every open item, every "here's *why* this CRD sync failed and how
   I fixed it" is evidence I can operate and reason about infrastructure. That's
   worth more to my career goal than the dollar saving, and it's something no
   amount of cloud credit would have produced, because the cloud hides the exact
   layers I needed to learn.

Put bluntly: I spent ~S\$1,200 once plus pocket change monthly to build something
that would cost ~US$11k/year to rent *and* that teaches me the thing I'm actually
trying to learn. The economics are firmly on the side of building.

Another thing that I must preface is that building the cluster my way would have
you touching the hardware and configuring very fragile components such as the BIOS
and manually tuning the custom setup. If you're rather new to PC hardware, it is 
perhaps not very ideal to buy random custom PCs online.

## 4. The reality of compute in Singapore, 2026

Zooming out from my cluster to the environment it sits in, a few things are true
about acquiring compute in Singapore right now.

### Electricity is the structural constraint on any local compute

Singapore generates ~95% of its electricity from imported natural gas, so the
tariff tracks global gas prices with a ~one-quarter lag. In 2026 that's meant a
climb to a **record 34.78 cents/kWh (with GST) for Q3**, a 17% single-quarter
jump. For a hobbyist homelab this is a rounding error (S$25-33/month). But it's
the reason Singapore is an *expensive place to run always-on compute at scale*,
and it's a real input into why large-scale compute here leans toward efficiency
and toward the cloud's shared-infrastructure model. Land and power are both scarce
and priced accordingly. The lesson scales inversely: **small always-on rigs are
cheap to run; large ones are penalised by exactly the same tariff.**

Fun fact: Despite being an island with no natural resources, the electricity prices
here are still cheaper than the UK or Australia.

### The cloud is dominant here, and the pricing reflects premium demand

Singapore is a major cloud hub, AWS (`ap-southeast-1`), GCP, and Azure all have
strong regional presence, and `ap-southeast-1` offers 800+ EC2 instance types. But
regional pricing runs *above* the US reference regions. For businesses with spiky
or unpredictable load, or that need managed durability and compliance, the cloud
is the default and a sensible one. The Singapore premium is the price of low
latency to a dense Southeast Asian market.

### The buy-vs-rent line: when local hardware wins

The principle that governs the whole comparison: **cloud wins for spiky or
unpredictable load; owned hardware wins for steady, high, predictable
utilisation**, because the cloud's core value is elasticity, and if you never
scale to zero you're paying an elasticity premium for nothing. The signals that
tip an organisation toward owning or colocating hardware:

- **Flat, high baseline utilisation** (nodes pinned at 60-80% around the clock),
  the classic "we're paying a 3-5x markup on compute we could own" realisation.
- **Sustained spend into the seven-figures-a-year range**, where it earns a
  dedicated finance/infra analysis.
- **GPU-heavy workloads**, where rental premiums accelerate the breakeven.
- **Heavy data egress**, where cloud pricing punishes moving data out.
- **Regulatory or latency requirements** that favour owned/colocated kit.

Real companies have made exactly this move at scale, Dropbox and 37signals/
Basecamp are the well-known cloud-repatriation cases. The honest counterweight is
that owning hardware demands headcount and operational burden that erodes the
savings, which is why the common landing spot is **hybrid**: own the steady
baseline, burst into cloud for peaks.

### My cluster as a micro-version of that same logic

My design *is* the hybrid principle in miniature. `deus` is the always-on baseline
(cheap to keep up); heavy GPU draw is spun up only when needed; Cloudflare Tunnel
(essentially free public ingress) handles the low-bandwidth public surface while
owned WireGuard handles bulk transfer. I'm making the same steady-baseline-vs-
elastic-burst tradeoffs a real infra team makes, just at a scale where the whole
year's power bill is less than a fortnight of the cloud-equivalent invoice.

## 5. The bottom line

- **Capital:** ~S$1,200 once, more with further upgrades or additional nodes.
- **Running:** ~S$25-33/month of electricity at the current record-high tariff
  the only recurring cost, and a small one.
- **Cloud-equivalent:** ~US$7,000-11,600+/year, higher still at Singapore-region
  pricing.
- **Value:** an order-of-magnitude compute-per-dollar advantage for this
  always-on workload, *plus* the learning and portfolio value that was the actual
  goal, which the cloud would have hidden rather than taught.

Money well spent. The hardware is already amortised many times over against the
cloud line item, the electricity is the rounding error, and the thing I walk away
with, the ability to operate and reason about a real platform, is exactly what
I set out to buy.

You would perhaps argue the true resource being spent here is **time**. *That is correct*.
But it is still time well spent learning the infrastructure in a practical manner,
as well as a hobby worth exploring.

---

*Tariff figure: SP Group regulated tariff, Q3 2026 (1 Jul-30 Sep), 34.78 cents/kWh
incl. GST, re-check quarterly. Cloud figures: AWS on-demand, us-east-1 reference
with note that ap-southeast-1 runs higher; re-verify at aws.amazon.com/ec2/pricing
before quoting.*
