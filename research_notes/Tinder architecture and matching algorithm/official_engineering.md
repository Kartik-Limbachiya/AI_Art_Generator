# Tinder official engineering sources — notes

Compiled from first-party Tinder engineering posts read in this session (lifeattinder.com, mlconf.com) plus secondary summaries where noted. [P] = read from primary source; [S] = secondary/snippet only. Posts on lifeattinder.com show no dates in fetched text; index reflects a 2026 snapshot.

## Infrastructure
- **Kubernetes migration** (Tinder Eng, Medium, 2019) [S]: moved from EC2 to Kubernetes; goals: containerization, immutable deployments, low-touch ops. https://medium.com/@TinderEng/tinders-move-to-kubernetes-cda2a6372f44
- **TAG — Tinder API Gateway** (Tinder Tech Blog, Oct 2022) [S via ByteByteGo/Better Stack/Nordic APIs; Medium 403]: JVM framework on Spring Cloud Gateway; config-driven — each app team builds its own gateway instance; pre-filters → route via Envoy service discovery → post-filters; 500+ microservices on an Envoy service mesh; built for >1B swipes/day; evaluated and rejected AWS API GW, Apigee, Kong, Tyk, KrakenD, Express Gateway. https://medium.com/tinder/how-we-built-the-tinder-api-gateway-831c6ca5ceca ; https://blog.bytebytego.com/p/how-tinders-api-gateway-handles-a
- **Envoy global rate limiting** talk, EnvoyCon 2020 [S]. https://envoycon2020.sched.com/event/eccm/how-tinder-implemented-envoy-global-rate-limiting-at-scale
- **Kafka** as event backbone (see ES8 and Smart Photo posts) [P].

## Recommendations ("Recs")
### Geosharded recommendations (Tinder Tech Blog, 2019, 3 parts) [S; Medium 403]
- Part 1 (Frank Ren): sharding approach; secondary summaries say shards are Google S2 cells, queries hit nearby cells sized by radius. Part 2 (Xiaohu Li): all geoshard indices in one cluster with load balancing. Part 3 (Devin Thomson): documents move between indices → guaranteed write ordering + strongly consistent reads.
- https://medium.com/tinder/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a ; part-2 …3396a8a7efb ; part-3 …2d2cb2f0594b ; https://highscalability.com/designing-tinder/

### Tinder's migration to Elasticsearch 8 [P] — Igor Sokolov, Jessica Hickey, Rongxin Du
https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8
- ES powers Recs, Trust & Safety, logging. "Over 90% of all our Recs comes from a single Elasticsearch cluster."
- Two capabilities: filter-based search over millions of profiles (location, age, interests, preferences); ranking via custom Java "Tinder ES plugin" running ML-based scoring; 100+ scoring scripts with minimal tests; ES8 bans negative scores.
- Geo-sharded index: 12% CPU vs 50% single index. Shards/node 3→14; primary shards 1→2 for scoring-heavy queries (CPU ~2×, response time −25–30%).
- ECK on Kubernetes, in-house "Scaffold" IaC; first ECK cluster in prod 2022. ES6 previously via Puppet + manual EC2. Stayed off OpenSearch (custom plugins).
- Write path: Kafka topic = source of truth; esreindexjob backfill then real-time worker; random doc field-by-field verification.
- Offline eval via event-based traffic replay; set overlap + Levenshtein list comparison at top-20/top-100. Three waves of online A/B.
- Results: zero outages; <0.2% data discrepancy; kNN vector search enabled; **Two-Tower "2T P(Match)": +6.5% match rate, +22% match volume; "2T P(Like)": +3.8% Swipe Right Rate (SRR)**; profile-embedding "MoreLikeThis"; p99 latency −12–56%; "Million plus dollar annual cost savings"; cross-AZ costs up. Node c6id.8xlarge→16xlarge.

### Keeping recommendations fresh [P] — Ruicong Yan (Staff PM), Lichao Liu (Staff SWE ML)
https://lifeattinder.com/blog/how-to-keep-recommendations-fresh
- Pipeline named explicitly: **eligibility filtering → retrieval → ranking → reranking**.
- "Dynamic recycling": previously passed profiles can resurface; recent passes stay blocked; older passes return via limited tiers; unseen prioritized; resurfacing capped.
- Per-pair serving state: last interaction time, pass or not, times shown within rolling window ("seen" logic).
- Separates "eligibility" from "competitiveness". Letting newest deep-learning retrieval/ranking models rank recycled candidates increased eligible pool and engagement. No numbers.

### Recommendation embeddings [P] — Kevin Thomas (EM II), AI Day project
https://lifeattinder.com/blog/recommendation-embeddings
- Recs-team models learn high-dimensional user representations "from profile and activity signals"; architecture unnamed. "Recommendation embeddings were built to improve recommendations, not to explain people."
- Databricks notebook: sample → cohort filter → PCA → K-means (HDBSCAN rejected: continuum) → UMAP → cluster profiling.
- Cohort >300,000 US Gen Z women 18–29, K=5. Largest cluster 25.7%: longer bios, more photos, higher verification, more likes sent, but fewer likes/swipes/conversations received — "profile-rich, high-outbound, lower-reciprocity". Linked to persona "Shelby / Sincere Seeker". Explicit limits: regions not natural types.

### TinVec (MLconf SF 2017) [P abstract] — Dr. Steve Liu, Chief Scientist
https://mlconf.com/?p=8768
- "embeds users' preferences into vectors leveraging on the large amount of swipes"; 26M matches/day, >20B matches to date (2017). Slides on SlideShare (not read). Reportedly powered "Super Likeable" (TechCrunch 2017, [S]).

### Elo retirement — "Powering Tinder: The Method Behind Our Matching" (2019, updated 2022) [S; help.tinder.com fetch failed]
- Elo "outdated", no longer relied on; system adjusts potential matches as your profile is liked/passed; changes within ~24h. https://www.help.tinder.com/hc/en-us/articles/7606685697037-Powering-Tinder-The-Method-Behind-Our-Matching ; https://www.engadget.com/2019/03/18/tinder-dumps-desirability-scores

## Photo ML
### Smart Photos (2016) [S]
- Mike Hall (ML lead): photo order as multi-armed bandit, epsilon-greedy; exploration then exploitation; epsilon decay. Test group up to +12% matches (Tinder claim). https://www.engadget.com/2016/10/13/tinder-smart-photos-uses-swipe-data-to-select-your-best-pic/

### How Tinder Uses AI to Rank Profile Photos [P] — Trideep Rath, Skanda Suresh, Ben Shin
https://lifeattinder.com/blog/how-tinder-r-uses-ai-to-rank-profile-photos
- Open-source VLM (vision encoder + LLM), unnamed. Pairwise: which of two photos of same person performs better; optional prompt context age, gender, target gender, country. Output token probabilities; SFT on label token; LoRA.
- Inference: all N-choose-2 pairs; aggregate by win count + mean pairwise probability.
- Labels: multi-armed bandit rotates top photo → per-photo likes/impressions; high-confidence significant pairs only (beat larger noisy sets).
- Pairwise acc ~76% (prior DL ~68%); ROC-AUC ~0.84; top-photo acc +7–10pp offline; online A/B: more likes, matches, conversations.
- LoRA rank 32 best (8–64); larger batch better; higher resolution helps; training vision encoder biggest gain; profile features ~+0.2pp (+0.8pp at 2× data); target gender for men-seeking-men +1.8pp; ≤3B models struggled, 7B stable. In-house LLM serving (continuous batching, KV cache, quantization).

### Smart Photo + Kafka [P] — Gabriela Rodriguez-Florido
https://lifeattinder.com/blog/dig-deeper-to-scale-further
- VLM comparisons on GPUs, computed asynchronously: Kafka worker consumes photo-update events, recomputes, stores order. Legacy consumer acked before processing → rebuilt on Spring Kafka, AckMode.RECORD, retry topics + DLT. Results: fewer errors, lower GPU cost (no numbers).

### LLM features (Photo Feedback, hybrid Smart Photo) [P] — Siva Periyasamy
https://lifeattinder.com/blog/llm-features-you-can-bet-on
- Smart Photo hybrid: tuned LLM low-60s pairwise acc vs specialist model mid-70s; hybrid not shipped.
- Photo Feedback: human 1–5 ratings → majority vote labels; ~60% inter-rater agreement ceiling; precision over recall (74% example); failure modes: artistic photos, pets, unique experiences penalized; <10 few-shot examples for tone; deterministic thresholds in config; OpenAI prompt caching ~93% hit rate. Shipped globally: ~30% of users who saw it added/replaced a photo; engagement up low single digits.

### On-device AI Photo Selector [P] — Thomas Yoon
https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos
- iOS pipeline: selfie or "BasePhotoInference" fallback → download & **decrypt (CryptoKit)** TFLite models + third-party SDK model → per asset (OperationQueue, max 8 concurrent): Apple Vision face detect (exactly one face; VNDetectFaceRectanglesRequest, landmarks rev 3) → 112×112 aligned crop → **FaceMeSDK** similarity vs base embedding (threshold) → **Tinder TFLite model outputs "probability of receiving likes"** → sort → **TFLite moderation model on top 100** (underage, violence, text-heavy, etc.).
- `isNetworkAccessAllowed = false` (no iCloud fetch). Analytics described as counts only (faces detected/identified). "without ever leaving the device". Benchmarked 1,000 photos. Base-photo fallback "monumental funnel drop improvement".
- Help center: processing on device; Tinder only collects photos you choose; biometric data deleted on exit. https://www.help.tinder.com/hc/en-us/articles/21276850679693-Photo-Selector

### Chemistry (2025–26) [S]
- Announced with Match Q3 2025 results (Nov 2025): AI learns via questions and, with permission, Camera Roll; pilots NZ & Australia; "major pillar" of 2026 product (Spencer Rascoff); ~$14M Q4 revenue cost of testing. Feb 2026: still testing in Australia; "a single drop or two" instead of swiping; AI reordering of profiles shown to women credited with small gains; Face Check cut bad-actor interactions >50%. Mar 2026 (Axios snippet): live in AU/NZ, "camera roll scan" → "photo insights". No confirmed US launch as of Nov 2026.
- https://techcrunch.com/2025/11/05/tinder-to-use-ai-to-get-to-know-users-tap-into-their-camera-roll-photos/ ; https://techcrunch.com/2026/02/04/tinder-looks-to-ai-to-help-fight-swipe-fatigue-and-dating-app-burnout/ ; https://www.axios.com/2026/03/12/tinder-ai-features-hinge-bumble

## Gaps
- Medium-hosted posts (geosharding, TAG) and help.tinder.com could not be fetched; details are from secondary summaries.
- No first-party description of what fully replaced Elo, nor of Keepalive WebSocket, Redis/DynamoDB usage in this session.
