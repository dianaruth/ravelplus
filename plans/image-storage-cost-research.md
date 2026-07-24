# Image Storage Cost Research (PRD §10 #6 / #14)

**Date:** 2026-07-21 · **Researcher:** Diana (with Claude) · **Status:** Complete
**Sources:** [S3 pricing](https://aws.amazon.com/s3/pricing/), [S3 2026 breakdown](https://infratally.com/articles/aws-s3-pricing-explained-2026/), [Supabase pricing](https://supabase.com/pricing), [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/), [R2 egress analysis](https://egresscost.com/cloudflare/)

## TL;DR

**Image storage is one of the cheapest things in this product — not a cost center.** At our mandated compression (~200 KB/image), the per-image cost is a fraction of a cent regardless of provider. The proposed ">5 images per project" paywall gate cannot be justified as cost recovery — the marginal cost of those extra images is effectively zero. It's a legitimate product/upsell lever, but that decision belongs to positioning (does it fit the §9 "no penny-pinching" principle?), not to infrastructure budget. The real cost "step" isn't per-GB pricing at all — it's crossing a platform's free-tier ceiling into its next paid plan.

## The unit economics

Client-side compression before upload (already mandatory per the plan — ~1600px, ~75% JPEG, ~200 KB/image) at S3 Standard's $0.023/GB/month:

| Scenario | Storage | Cost/month |
|---|---|---|
| One image | 200 KB | $0.0000046 |
| A project's 5 extra images | 1 MB | $0.000023 |
| Free-tier user at a 100-image cap | 20 MB | $0.00046 (≈½¢/year) |
| Subscriber with 1,000 images | 200 MB | ≈½¢/month |
| Extreme case, 10,000 images | 2 GB | $0.046 (≈1% of a $3.99 subscription) |

Fleet-wide: 10,000 users averaging 30 MB each ≈ 300 GB ≈ **~$7/month**. At 100K users, ~$70/month — trivially covered even at low paid-conversion rates.

**Egress** (devices downloading images) is normally the pricier line ($0.09/GB on S3), but two things cap it here: offline-first means each device downloads an image once and caches it locally, and MVP has no social/feed surface — users only ever load their own images. Provider choice (below) can also zero this cost entirely.

## Where the real cost steps are

Per-GB pricing is a red herring. What actually costs money is crossing a **platform tier boundary**:

1. **Supabase Storage (current plan):** free tier = 1 GB. Next step is **Pro at $25/month flat** (includes 100 GB). This ceiling arrives around ~50 heavy users. The $25/month step *is* the realistic image-storage budget for the first year — not a per-GB number.
2. **Cloudflare R2:** 10 GB free forever, **zero egress fees at any volume**, then $0.015/GB storage. S3-API-compatible — a drop-in "S3-style upload" target with a 10× longer free runway and no bandwidth line item, ever.
3. **Actual AWS S3:** the worst option for us. AWS ended the perpetual free tier in mid-2025 (replaced with $200 in credits for 6 months), and it's the only one of the three with a meaningful egress charge. No reason to pick it over R2 for this workload.

## Recommendation

- **MVP:** stay on Supabase Storage — integrated with RLS/auth, zero extra code, and the 1 GB free tier comfortably covers the friends/beta scale.
- **When approaching ~1 GB:** decide based on revenue at that point — either absorb the $25/mo Supabase Pro upgrade (likely wanted anyway for DB headroom), or move *just images* to R2 for a 10 GB-free, zero-egress runway. Contained migration since images are addressed by storage path, not foreign-keyed into query logic.
- **For the free-tier image limit (#6):** set it generously — cost is a non-factor at any number under discussion (even "500 MB per free user" costs ~$0.01/user/month). Choose the number for product-shape and upsell-psychology reasons, not infrastructure math.
- **For the >5-images-per-project paywall idea:** keep it if it works as a premium *feature* signal, but don't defend it as cost control — it isn't one.
