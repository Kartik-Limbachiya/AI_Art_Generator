# Tinder: Architecture, Matching Algorithm and Reverse-Engineering Research

Complete record of the research done in this session (November 2026): how Tinder is built, how its matching/ranking and photo systems work, how popularity is tied to monetization, and a fact-check of the October 2026 "I reverse engineered Tinder" video by ayoships.

**Contents**
1. Scope, method and ground rules
2. Session log: what was done, step by step
3. Key verified findings at a glance
4. Part A — Final research report (full)
5. Part B — Research notes: local code evidence (full)
6. Part C — Research notes: Tinder official engineering (full)
7. Part D — Research notes: patents, academia, journalism (full)
8. Part E — Research notes: reverse-engineering claims (full)
9. Appendix 1 — The ayoships video: transcript and on-screen claims
10. Appendix 2 — Artifacts, tools and file locations
11. Appendix 3 — Evidence labels used

---

## 1. Scope, method and ground rules

**Goal:** understand Tinder's engineering — architecture and internal algorithm — for learning purposes, and check the claims of the ayoships reel.

**Sources used**
- Public tinder.com web assets (HTML, headers, JavaScript bundles, service worker, manifest), downloaded with ordinary GET requests like any browser.
- REA ("Reverse Engineer Anything", `rea-agents` v6.9.0, built from the `Kartik030204/rea` copy of `morluto/rea`) for static JavaScript analysis.
- `Davis-3450/tinder-protos`: a public GitHub repo of 212 protobuf schemas self-described as "scraped from tinder.com" (last commit 2025-09-09).
- Tinder's engineering blog (lifeattinder.com), conference talks, help-center pages, press coverage.
- Patents (Google Patents), academic papers, investigative journalism, regulatory/pricing studies.
- The ayoships reel (screenshots + transcript supplied by the user).

**Ground rules followed**
- No Tinder account, no calls to Tinder's private APIs, no traffic interception.
- No bypassing of bot protection: APKMirror served a Cloudflare challenge, so the Android APK was not downloaded.
- No decryption of protected material: Tinder's current on-device models are encrypted (CryptoKit), so they were not extracted.
- No reproduction of Tinder's code; only identifiers, field names and one-line context.
- No endpoint catalog or anything that would help automate, scrape or bypass Tinder's protections.
- Static analysis only; REA never executed Tinder's code.

---

## 2. Session log: what was done, step by step

1. **Checked the REA repo.** `Kartik030204/rea` is a copy of the official `morluto/rea` (one upstream commit behind, 9 files differ, nothing suspicious; `install.sh` points at `morluto/rea`). REA is an MCP server + CLI that orchestrates Ghidra, Hopper, IDA, Rizin, GDB, objdump, apktool, Jadx, Frida, and analyzers for .NET, Flutter/Dart, IL2CPP, V8, WASM, Python, Electron and JavaScript. Its DISCLAIMER forbids DRM/anti-bot circumvention, cheats, unauthorized access.
2. **Fetched tinder.com publicly.** Response headers showed nginx + Express, `x-render-method: ssr`, AWS ALB cookies, CloudFront, a report-only CSP allowing `wss://keepalive.gotinder.com`, HSTS, `s-maxage=3024000`. The HTML loads 5 entry bundles (main, webpack runtime, vendor, tinderapp, shared chunk 79047), preloads fonts/CSS, dns-prefetches `api.gotinder.com` and the image CDN, and hydrates from `window.__data` (`context`, `internationalization`, `routing`, `webProfile`) plus ~290 KB of inline i18n strings. A Googlebot user-agent received a larger (976 KB vs 576 KB) page.
3. **Fingerprinted libraries.** React 18.3.1, Redux + redux-saga, React Router v6, Webpack (`webpackChunktinderweb`), react-intl/FormatJS, Sentry, Segment, AppsFlyer, GTM, Braintree, Adyen, Arkose Labs, reCAPTCHA, Lottie, Workbox. Webpack runtime references ~3,940 lazily loaded chunks (2,089 named, excluding polyfills), including per-locale Intl polyfills and an `async-protobuf-schema-keepalive` chunk. No source maps are published.
4. **Built REA and ran `analyze-javascript-application`** on the 6 files (5 bundles + service worker): 2.36 MB parsed, 460,655 AST nodes, 0 parse failures, 814 modules, 1,809 graph nodes, 4,762 edges, 41,580 semantic nodes, 12,094 function fingerprints, 51 route patterns, 221 low-confidence endpoint candidates (210 in `main`), storage in IndexedDB, sessionStorage (`what-intent`, `what-input`) and Cache Storage, one Web Worker. `main` holds 1,000 modules and 175 of 178 chunk references.
5. **Reviewed the ayoships reel** (25 screenshots + transcript).
6. **Verified the reel's web claim.** Downloaded the single `SuperlikeUpsellModal` chunk (4.4 KB): it logs an event with `location:"discovery"`, `name:"upsell_superlikes_high_rsrr"`, `sourceSessionEvent:"revenue"`, `type:"modal"`. The main bundle contains the lever `superlike.upsell_high_rsrr_enabled` and Redux state `superlikeHighRsrr`.
7. **Attempted to obtain the Android APK.** APKMirror returned a Cloudflare "Just a moment…" challenge; stopped rather than evade it. LiteRT 2.3.0 installed for model work; Jadx download was blocked by the session's permission checks and was not pursued further.
8. **Public-source research.** Tinder engineering posts (ES8 migration, recommendation freshness, recommendation embeddings, VLM photo ranker, Smart Photo + Kafka, LLM Photo Feedback, on-device Photo Selector), TinVec talk, Elo retirement, Smart Photos 2016, geosharding, TAG gateway, Kubernetes, pricing studies.
9. **Mined feature flags (levers).** ~238 flag strings in `main` across namespaces such as chat, trust, identity, profile, purchase, integrity, paywall, superlike, likes_you, bouncer, subscription_discount (incl. `personalized_offer_enabled`).
10. **Downloaded a few feature chunks** (`async-recs-feature`, `async-recs-index`, `async-likesYou-feature`, `async-match-feature`). Found that each recommendation object carries `is_superlike_upsell`, and every swipe sends a `recsStatusInfo` block with `duration_in_millis`, `method`, media/profile-element view counts, viewed element order, shared interests, `s_number`.
11. **Read the `tinder-protos` schemas.** `photo_selector_photos_collection_details` has `rsrr_scores`, `explicit_scores`, `text_scores`, `underage_scores`, `similarity_scores`; the upload event has `rsrr_scores` and `percentage_of_scrolling`; the moderation event counts `less_clothed_flagged`, `mirror_selfie_flagged`, `baby_filtered`, explicit/text/under-age filtered.
12. **Ran a structured deep-research pass** (code evidence, official engineering, patents/academia/journalism, reverse-engineering claims) and wrote the final report (Part A).

---

## 3. Key verified findings at a glance

| Finding | Evidence |
|---|---|
| Server-rendered React 18.3.1 + Redux/redux-saga web app, ~2,089 named lazy chunks, CloudFront → ALB → nginx → Express | Verified in tinder.com assets |
| 500+ microservices, Envoy mesh, TAG gateway on Spring Cloud Gateway, Kubernetes on AWS | Tinder first-party (via summaries where Medium blocked) |
| >90% of recommendations from one geo-sharded Elasticsearch cluster; custom Java scoring plugin; 100+ scoring scripts | Tinder first-party |
| Pipeline: eligibility filtering → retrieval → ranking → reranking; "dynamic recycling" of passed profiles | Tinder first-party |
| Two-tower P(Match): +6.5% match rate, +22% match volume; P(Like): +3.8% swipe-right rate | Tinder first-party |
| Elo retired 2019; patent US 12,105,941 (granted Oct 2024) claims an attractiveness score from views vs positive preferences | Tinder first-party; patent |
| `RSRR` drives a Super Like upsell logged as `"revenue"`; per-rec `is_superlike_upsell` | Verified in client code |
| Each swipe sends dwell time, input method, profile-element views and order | Verified in client code + schemas |
| On-device TFLite model scores camera-roll photos for "probability of receiving likes"; moderation model on top 100 | Tinder first-party |
| Photo Selector analytics schemas contain per-photo score arrays (rsrr, explicit, text, underage, similarity) | Verified in schemas; in tension with Tinder's "counts only" description |
| Smart Photos: epsilon-greedy bandit (2016) → pairwise 7B VLM, ~76% pairwise accuracy (prior ~68%) | Tinder first-party |
| Age-based pricing: 30–49 paid ~65% more (2022 study); ~$24M California settlement | Third-party study / legal |
| ayoships bias experiment (skin tone, clothing, mirror selfies…) | Reverse-engineer claim only; methodologically weak |

---

## Part A — Final research report

### Inside Tinder: Funnels, Ratios, and Photo Models

Tinder runs as a cloud-native microservice system: **500+ services on Kubernetes behind a config-driven Spring Cloud Gateway and an Envoy mesh, with Kafka as the source of truth and one geo-sharded Elasticsearch cluster serving over 90% of recommendations**. Ranking is a four-stage funnel (eligibility filtering, retrieval, ranking, reranking). Since at least 2026 it has been driven by two-tower deep models that predict P(Like) and P(Match) and run as custom scoring plugins inside Elasticsearch. Tinder says publicly that it retired the Elo "desirability score" in 2019. Popularity has not gone away, though. It survives as a learned signal. A 2024 Match Group patent claims an "attractiveness score" computed as the ratio of positive swipes to views. The production web client still uses a metric called **RSRR** (most plausibly "right-swipe received ratio") for one purpose: showing a Super Like upsell, tagged `revenue`, when you are about to like someone in high demand. Photo ML has moved through three generations. First came a 2016 epsilon-greedy bandit (Smart Photos). Next came on-device TFLite models that score your camera roll for "probability of receiving likes" and run moderation. The current generation is a 2026 pairwise vision-language model trained on bandit outcomes. The October 2026 "I reverse engineered Tinder" video by Ayo O (@ayoships) holds up on its static findings: RSRR naming, the revenue-tagged upsell, per-photo score fields and dwell-time telemetry all check out. Its behavioral claims are weak, especially the photo-bias experiment. That experiment probes a 2022-era on-device photo-picker model with about 800 synthetic images and small per-condition samples. It shows what that model outputs, not how Tinder ranks real people today. For a developer, Tinder is a useful case study in serving ranking inside the search engine, treating the event log as the database, and labelling ML with bandits.

**Evidence key used throughout:** **[T1]** = Tinder/Match first-party statement or engineering post; **[V]** = verified in shipped tinder.com client code or the public tinder-protos schema dump; **[P/A]** = patent or peer-reviewed/academic work; **[RE]** = reverse-engineer claim only, not independently confirmed; **[X]** = contradicted by, or in tension with, other evidence. A label attaches to the claim it follows.

#### Five hundred services sit behind one config-driven gateway

The system breaks down into six layers, from the user inward.

The **web client** is a server-side-rendered React 18.3.1 single-page app built on Redux and redux-saga. It is split into roughly 2,089 lazily loaded webpack chunks and ships with a Workbox service worker for push and offline caching, plus Sentry 7.42.2 for performance, profiling and session-replay sampling **[V]**. Response headers show nginx and Express doing SSR behind an AWS Application Load Balancer and CloudFront. The CDN cache lifetime is about 35 days and the browser cache lifetime is one hour **[V]**. The content-security policy is set to report-only, which means it monitors violations but does not block them **[V]**. Most create/read/update/delete traffic is JSON over REST. Several service families, including keepalive, media, dynamic UI, rooms and auth, use **protobuf schemas that are loaded lazily** ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**.

The **real-time channel** is a "keepalive" WebSocket. Its protobuf message types are almost all server-driven nudges: refresh-profile, dynamic-UI, direct-message, typing indicators, room events and CRM campaigns ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. So the WebSocket's main job is to tell the client when to re-fetch, not to carry the data itself (inference).

At the **edge**, the Tinder API Gateway (TAG) is an in-house JVM framework built on Spring Cloud Gateway. Each application team configures its own gateway instance. A request passes through pre-filters, is routed via Envoy service discovery, and then passes through post-filters. Tinder evaluated and rejected AWS API Gateway, Apigee, Kong, Tyk and KrakenD before building TAG, which was designed for **more than 1B swipes a day across 500+ microservices** ([Tinder Eng via ByteByteGo](https://blog.bytebytego.com/p/how-tinders-api-gateway-handles-a); [original post](https://medium.com/tinder/how-we-built-the-tinder-api-gateway-831c6ca5ceca)) **[T1, via secondary summary]**. Rate limiting is global, implemented in Envoy ([EnvoyCon 2020](https://envoycon2020.sched.com/event/eccm/how-tinder-implemented-envoy-global-rate-limiting-at-scale)) **[T1]**. The platform moved from hand-managed EC2 to Kubernetes in 2019 to get containerised, immutable deployments ([Tinder Eng](https://medium.com/@TinderEng/tinders-move-to-kubernetes-cda2a6372f44)) **[T1]**.

The **data plane** has Kafka at its center. In the Elasticsearch 8 migration post, a Kafka topic is the source of truth for the recommendation index. A backfill job rebuilds the index, and a real-time worker keeps it current ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)) **[T1]**. Photo re-ranking is also a Kafka consumer. It recomputes on every photo-update event and was rebuilt to commit offsets per record, with retry topics and a dead-letter topic, after the legacy consumer acknowledged messages before it had processed them ([Tinder Eng](https://lifeattinder.com/blog/dig-deeper-to-scale-further)) **[T1]**.

**Search and recommendations** run on Elasticsearch, which also powers Trust & Safety and logging. In Tinder's words, "over 90% of all our Recs comes from a single Elasticsearch cluster." The index is geo-sharded. Secondary summaries describe the shards as Google S2 cells, with each query fanning out to the cells within its search radius ([Geosharded Recs pt. 1](https://medium.com/tinder/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a); [HighScalability](https://highscalability.com/designing-tinder/)) **[T1, via secondary summary]**. Sharding cut CPU from about 50% on a single index to about 12% **[T1]**. Ranking runs inside the cluster through a custom Java "Tinder ES plugin" with **100+ scoring scripts**. The cluster runs on Elastic Cloud on Kubernetes (ECK) with in-house infrastructure-as-code. Tinder chose to stay on Elasticsearch rather than OpenSearch because of those plugins ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)) **[T1]**.

**Trust and safety** spans several vendors and checks. The web client integrates Arkose Labs and reCAPTCHA, which the backend can switch between. It also includes FaceTec liveness checks, selfie and ID verification, Jumio age verification, client-side automation detection and a 24-value ban-reason taxonomy with an appeals flow **[V]**. Fraud detection by image comparison has been patented since 2013 ([US 8,559,706](https://patents.google.com/patent/US8559706B2)) **[P/A]**. Tinder reports that Face Check cut bad-actor interactions by more than 50% ([TechCrunch, Feb 2026](https://techcrunch.com/2026/02/04/tinder-looks-to-ai-to-help-fight-swipe-fatigue-and-dating-app-burnout/)) **[T1]**.

| Layer | What is known | Evidence |
|---|---|---|
| Client | SSR React/Redux SPA, 2k lazy chunks, service worker, Sentry | [V] |
| Push | Protobuf "keepalive" WebSocket carrying server nudges | [V] |
| Edge | TAG on Spring Cloud Gateway, Envoy mesh, global rate limits | [T1] (secondary) |
| Compute | Kubernetes since 2019; ECK for search | [T1] |
| Events | Kafka as source of truth; per-record-ack consumers | [T1] |
| Recs | One geo-sharded ES cluster, in-cluster ML scoring plugin | [T1] |
| Telemetry | "Hubble" typed event schemas, batched analytics | [V] |

#### Recommendations run as a four-stage funnel inside Elasticsearch

Tinder names the pipeline as **eligibility filtering, then retrieval, then ranking, then reranking** ([Tinder Eng, fresh recs](https://lifeattinder.com/blog/how-to-keep-recommendations-fresh)) **[T1]**.

**Eligibility** is a hard filter on location, age, stated preferences and per-pair serving state. For each viewer and candidate, the system tracks the last interaction time, whether it was a pass, and how many times the candidate has been shown in a rolling window. A "dynamic recycling" policy keeps recent passes blocked, lets older passes back in through limited tiers with a cap, and gives priority to profiles the viewer has not seen. Tinder separates "eligibility" (may this person be shown?) from "competitiveness" (should they win the slot?). It found that letting its newest deep retrieval and ranking models score recycled candidates, instead of hard-excluding them, increased both the eligible pool and engagement. The post gives no figures for this **[T1]**. The schema shows that the system logs when a rec falls outside the viewer's preferences, through fields such as `exceeds_age_range_by` and `exceeds_distance_by` ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. Recs can therefore deliberately leak outside your stated filters (inference).

**Retrieval and ranking** use embeddings. Tinder described "TinVec" in 2017. It embedded users' preferences into vectors learned from swipes, at a time when the service was producing 26M matches a day ([MLconf](https://mlconf.com/?p=8768)) **[T1]**. The ES8 migration enabled kNN vector search, and its online tests reported the clearest numbers Tinder has published on its ranker ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)) **[T1]**:

- a **two-tower "2T P(Match)" model** raised match rate by +6.5% and match volume by +22%;
- a **"2T P(Like)" model** raised swipe-right rate by +3.8%.

A profile-embedding "MoreLikeThis" feature shipped alongside them, and p99 latency fell 12–56%. The objective is therefore reciprocal. P(Like) predicts whether the viewer will swipe right on the candidate. P(Match) also needs the candidate to like the viewer back, and that model produced the larger gains. Reciprocal relevance has an older patent lineage. US 11,989,220 trains a gradient-boosted tree ranker on two-way implicit signals, such as one-way versus two-way messages and views, imbalances and timing. It claims to rank candidates who have no interaction history ([patent](https://patents.google.com/patent/US20200175047A1)) **[P/A]**. Researchers call this family of methods reciprocal recommenders ([Palomares et al. survey](https://arxiv.org/abs/2007.16120)) **[P/A]**.

Tinder's own description of its embeddings is cautious. A 2026 internal exploration clustered more than 300,000 US Gen Z women's recommendation embeddings into five groups. The largest group (25.7%) had richer profiles and sent more likes but *received* fewer, a "profile-rich, high-outbound, lower-reciprocity" pattern. The authors stress that the embeddings "were built to improve recommendations, not to explain people" ([Tinder Eng](https://lifeattinder.com/blog/recommendation-embeddings)) **[T1]**. What replaced Elo, end to end, has never been described first-party. The model architecture beyond "two-tower," the feature set and the reranking objectives all remain unknown.

#### Elo was retired, but a swipe-ratio popularity score never left

The history runs in three steps.

In **2016**, Tinder confirmed to Fast Company that it ran an internal Elo-style "desirability" rating. Sean Rad said, "It's not just how many people swipe right on you," and the reporter was shown his own score of 946 ([Refinery29 summarising Fast Company](https://www.refinery29.com/en-us/2016/01/101011/tinder-secret-score)) **[T1]**.

In **2019**, Tinder called Elo "outdated" and said it no longer relies on it. Its replacement explanation is that potential matches adjust as your profile is liked or passed, within about 24 hours ([Engadget](https://www.engadget.com/2019/03/18/tinder-dumps-desirability-scores); [Tinder Help](https://www.help.tinder.com/hc/en-us/articles/7606685697037-Powering-Tinder-The-Method-Behind-Our-Matching)) **[T1]**. A 2017 GDPR request by journalist Judith Duportail returned about 800 pages of personal data but no score, and Tinder treated its matching tools as core IP ([FlowingData](https://flowingdata.com/2017/12/01/800-pages-of-tinder-data/)) **[T1/journalism]**. Ireland's Data Protection Commission fined Match €11M in November 2026 over gaps in its "Download My Data" tool. The fine concerned completeness of the data export, not ranking ([Extra.ie](https://extra.ie/2026/11/13/news/tinder-handed-major-fine)).

The patent record shows the concept persisting. **US 12,105,941**, granted to Match Group in October 2024, claims an **attractiveness score defined as the ratio between how often a profile is viewed and how often others express a positive preference**. That score feeds a viewer-specific similarity score, which orders the card stack ([Google Patents](https://patents.google.com/patent/US12105941B2)) **[P/A]**. The specification also describes showing users with similar ratings to each other, a mechanism EPFL researchers characterise as stratifying users by attractiveness, IQ and similarity ([Pidoux et al.](https://aiequalitytoolbox.github.io/social-impact-tinder-pidoux.html)) **[P/A]**. A patent does not prove production use.

The client code does prove that a swipe-ratio metric is live. The production web bundle contains a feature flag `superlike.upsell_high_rsrr_enabled`, Redux state `superlikeHighRsrr`, and a `SuperlikeUpsellModal` analytics event `upsell_superlikes_high_rsrr` whose `sourceSessionEvent` is `"revenue"`. Recommendation objects carry an `is_superlike_upsell` boolean **[V]**. The acronym is never expanded in the code. "Right-swipe received ratio," the expansion the video offered, matches the patent's ratio exactly, but it remains an interpretation **[RE, consistent with P/A]**.

The academic baseline explains why this matters. On messaging networks, people contact partners about **25% more desirable than themselves** on average, and reply rates fall as that gap widens ([Bruch & Newman, Science Advances 2018](https://pmc.ncbi.nlm.nih.gov/articles/PMC6082652)) **[P/A]**. Early Tinder data showed women matching quickly and men slowly, which creates a feedback loop toward extreme strategies ([Tyson et al.](https://arxiv.org/abs/1607.01952)) **[P/A]**.

My reading is that Tinder replaced one explicit scalar with learned models that absorb popularity implicitly. A P(Like) tower trained on population swipes encodes "how many people like this kind of profile," and P(Match) explicitly weighs whether the candidate is likely to reciprocate. "No Elo" and "popularity still governs exposure" can both be true.

#### Every swipe ships dwell time, input method and profile-element views

The web client sends a structured status payload with each like or pass. It records:

- **dwell time** (`duration_in_millis`);
- the **input method** (button, keyboard, swipe gesture, or one of two in-profile buttons);
- counts of media and profile-element views;
- which profile elements were viewed, in what order, and their values (bio, interests, anthem, top artists, intent, descriptors);
- shared interests;
- a server-issued per-rec token, `s_number`, echoed back to the server.

The schema dump defines these as `RecsStatusDetails` and `RecsCustomDetails`. Further fields there record whether the candidate was a Top Pick, which photo index the swipe happened on, and premium-preference fulfilment ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. All of this is implicit feedback, the raw material for training a ranker. The artifacts prove the data is collected, not how ranking uses it. A search of all 212 scraped schemas found **no ranking score, embedding, Elo or "chemistry" field**. Ranking is entirely server-side and hidden from the client **[V]**.

Swipe behaviour has leaked through observable telemetry before. In 2018 Checkmarx showed that swipe direction could be inferred from encrypted packet sizes and that mobile images were fetched over plain HTTP. Sophos corroborated the findings and Tinder committed to fixes ([Sophos](https://news.sophos.com/en-us/2018/01/24/tinder-user-lack-of-encryption-means-stalkers-can-watch-you-at-it/?amp=1)) **[T1-acknowledged]**.

The video's claim that photo-view timing is logged per photo (`media_view_details_card` with view count and total duration) is directionally confirmed by the dwell and media-view fields above **[V]**. Its claim that your "type" is the average embedding of photos you swiped right on (`liked_swipees_image_average`) has no independent support **[RE]**.

#### Photo ML evolved from bandits to on-device TFLite to a 7B VLM

**Smart Photos (2016)** treated photo order as a multi-armed bandit. It used epsilon-greedy exploration that decayed into exploitation, and Tinder claimed up to **+12% matches** in test groups ([Engadget](https://www.engadget.com/2016/10/13/tinder-smart-photos-uses-swipe-data-to-select-your-best-pic/)) **[T1]**.

The bandit later turned into a labelling engine. The **2026 photo ranker** is an open-source vision-language model fine-tuned with LoRA on *pairwise* questions: which of two photos of the same person performs better? Its labels come from the bandit's per-photo likes-per-impression, filtered to high-confidence, statistically significant pairs. Tinder found that this filtering beat using larger, noisier label sets. At inference the model scores all N-choose-2 pairs and aggregates them by win count and mean probability. Results ([Tinder Eng](https://lifeattinder.com/blog/how-tinder-r-uses-ai-to-rank-profile-photos)) **[T1]**:

- **76% pairwise accuracy**, against 68% for the prior deep-learning model;
- ROC-AUC of about 0.84;
- top-photo accuracy up 7–10 points offline;
- online gains in likes, matches and conversations.

Tinder also reported which design choices mattered. LoRA rank 32 was best. Training the vision encoder gave the biggest gain. Models of 3B parameters or smaller were unstable, while 7B was stable. Profile context added little overall, except target gender for men seeking men (+1.8 points) **[T1]**. A general-purpose LLM tuned for the same task reached only low-60s accuracy, so the hybrid design did not ship ([Tinder Eng](https://lifeattinder.com/blog/llm-features-you-can-bet-on)) **[T1]**. A related **Photo Feedback** feature shipped globally, built on rater labels with about 60% inter-rater agreement and tuned for precision over recall. About 30% of users who saw it added or replaced a photo **[T1]**. Tinder has not disclosed any disparate-impact testing for these models ([Dating Industry Insights](https://www.datingindustryinsights.com/news/tinder-ai-photo-ranking-system-explained)).

The **on-device Photo Selector** (iOS) is a separate system that suggests photos from your camera roll ([Tinder Eng](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)) **[T1]**. Its pipeline:

1. Download and decrypt TFLite models with CryptoKit.
2. For each photo, run Apple Vision face detection, requiring exactly one face.
3. Align the face to a 112×112 crop and check FaceMe SDK similarity against your selfie embedding.
4. Score each surviving photo with an in-house TFLite model that outputs the **"probability of receiving likes"**.
5. Sort the results, then run a TFLite **moderation model on the top 100** for underage subjects, violence and text-heavy images.

iCloud fetching is disabled. Tinder says processing never leaves the device and that analytics are counts only **[T1]**.

The schema dump complicates the privacy claim. The `PhotoSelectorPhotosCollectionDetails` and `PhotoSelectorUploadSelectedPhotosDetails` event types contain **per-photo arrays `rsrr_scores`, `explicit_scores`, `text_scores`, `underage_scores` and `similarity_scores`**, alongside moderation counters such as `num_photos_less_clothed_flagged` and `num_photos_mirror_selfie_flagged` ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. Score arrays in a client-to-server analytics schema sit uneasily with "counts only" **[X: in tension]**. However, nobody has publicly shown these fields populated in live traffic, and none of these schemas are referenced in web code, so they are mobile telemetry. The schema also ties the likes-probability model to the RSRR name, which links the photo model and the Super Like upsell to the same metric family (inference).

**Chemistry**, Tinder's 2026 AI matching pillar, learns from questions and, with permission, from camera-roll "photo insights." It is live only in Australia and New Zealand, and testing it cost about $14M in Q4 revenue ([TechCrunch](https://techcrunch.com/2025/11/05/tinder-to-use-ai-to-get-to-know-users-tap-into-their-camera-roll-photos/); [Axios](https://www.axios.com/2026/03/12/tinder-ai-features-hinge-bumble)) **[T1]**.

#### Popularity signals are wired directly into paywalls

The clearest link between ranking and revenue is the high-RSRR Super Like modal described above. When a rec is flagged as high-demand, the client prompts you to "send a Super Like instead for a better chance," and logs the impression as revenue **[V]**. The video's iOS string matches this behaviour ([hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)) **[RE, behaviour V]**.

Around it sits a large surface of feature flags, internally called "levers" **[V]**. They cover:

- personalized and discount offers (`subscription_discount.personalized_offer_enabled`, a discount paywall with cool-down and rate limits);
- "bouncer" paywalls;
- Platinum priority likes triggered by a swipe-right threshold (`platinum.priority_likes_swipe_right_threshold`);
- match expiration with urgency windows;
- Likes You filtering with minimum-count gates;
- ad cadence inside the card stack.

Purchase telemetry records `age`, base price, intro pricing and discount campaign ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. The paywall feature taxonomy includes BOOST, SUPER_BOOST, PRIME_TIME_BOOST, SUPER_LIKE, LIKES_ANNUITY, MATCH_EXPIRATION and PURCHASED_LIKE **[V]**. Each one sells either more exposure or more visibility into who is giving exposure to you.

Personalisation has a documented history. A 2022 six-country study found users aged 30–49 were charged on average **65.3% more** for Tinder Plus than 18–29s. Tinder ended age-based pricing in 2022 ([Consumers International](https://www.consumersinternational.org/news-resources/news/releases/new-research-consumers-worldwide-concerned-by-opaque-online-pricing/)), after a California class action settled for about $24M ([Consumer Reports](https://www.consumerreports.org/consumer-protection/tinder-is-phasing-out-higher-prices-for-users-ages-29-plus-a3881901881)) **[P/A, journalism]**. Today's code shows the *capability* to segment offers, not the pricing logic. A structural point follows: a two-sided market whose ranker concentrates attention creates scarcity, and every boost or priority product monetizes that scarcity. The ranker and the paywall are economically coupled even if they are separately engineered (inference).

#### The ayoships video gets the strings right and the science wrong

Ayo O (Ayomide Omolewa, co-founder of a0.dev) posted a 79-second video on 20 October 2026 that drew about 2.4M views ([X post via fxtwitter](https://api.fxtwitter.com/TheDonAyo/status/2112591782857109809); [YouTube Short](https://www.youtube.com/shorts/Qziwio_mo0s)). It belongs to a fast-turnaround "I reverse engineered X" series, at least one installment of which was openly AI-assisted ([X](https://x.com/ayoships/status/2110461702936695263)). The method had three parts: decompiling a **2022 Android build (v13.0.0)**, reading iOS strings, and running an extracted TFLite model on **800+ AI-generated images**, changing one variable at a time ([hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)). No repository, image set, prompts or model hashes were published. No Tinder response or mainstream press coverage was found. Ayo himself reportedly concedes that client reverse engineering cannot reveal server code ([LinkedIn snippet](https://www.linkedin.com/in/omolewa/)).

| Claim | Verdict |
|---|---|
| "RSRR" is a live metric name | **[V]** in web flags, state and protos |
| High-RSRR Super Like upsell tagged as revenue | **[V]** |
| Per-photo dwell time and view counts logged | **[V]** (dwell and media views per swipe) |
| Photo Selector schema carries per-photo rsrr/explicit/underage/similarity scores | **[V]** schema; **[X]** in tension with Tinder's "counts only" statement; live population unproven |
| On-device TFLite likes model plus moderation model | **[T1]** confirmed by Tinder engineering |
| Files fm.bin (~9.5 MB) and mn.bin (~2.5 MB), gender-split heads rsrrFemale/rsrrMale | **[RE]** only |
| Selector defaults: 20,000 photos, 2 years, 0.7 similarity, 27 results | **[RE]**; Tinder gives no numbers |
| "Your type" = average of right-swiped photo embeddings | **[RE]**, no corroboration |
| Findings describe how Tinder ranks people today | **[X]**: 2022 client model ≠ server ranker; Smart Photos moved to a VLM in 2026 |

**The photo-bias experiment** reported these results ([hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)) **[RE]**:

- mirror selfies scored about ×3.5;
- for men: outdoors +49%, shirtless +23%, sunglasses +15%;
- for women: low neckline +63%, sunglasses −26%, 15 years older −16%, heavier −54%;
- **darker skin −15%, lower in all 12 scenes**.

Some cells rest on as few as 8 images. No variance, confidence intervals or multiple-comparison correction were reported, and the writeups disagree slightly with each other (−24% vs −26%). The two most robust results are the 12/12 skin-tone drop (sign test p ≈ 0.0005) and the 16/16 mirror-selfie lift, and even those are among 13+ contrasts. Generative skin-tone edits typically shift global luminance and contrast as well. Twitter's own saliency audit traced part of its racial disparity to exactly those image properties ([Yee et al.](https://arxiv.org/abs/2105.08667v2)). Synthetic faces are out of distribution for a model trained on real swipes. The shirtless and low-neckline images would likely trip the `less_clothed` moderation path in production. The ×3.5 mirror-selfie score sits awkwardly beside a `mirror_selfie_flagged` counter **[V]**. Most importantly, if the label really is RSRR, the model learns *how audiences swipe*, so it reproduces audience bias by design. That is consistent with published beauty-model bias findings ([arXiv 2509.24138](https://arxiv.org/html/2509.24138v1)) and with racialised desirability on dating platforms ([Hutson et al.](https://arxiv.org/abs/1809.01563); [Bruch & Newman](https://pmc.ncbi.nlm.nih.gov/articles/PMC6082652)) **[P/A]**.

Treat the directions as plausible hypotheses about one old on-device model, and the magnitudes as unreliable. A credible replication would need real re-photographed images, luminance-matched skin edits, at least 30 images per condition with intervals, correction for multiple comparisons, and current model versions.

#### Engineering lessons worth stealing

**Rank where the data lives.** Instead of pulling candidates out to a separate ranking service, Tinder ships its scoring logic and ML into the search engine as a plugin. Filtering, kNN retrieval and model scoring then happen in one hop over a geo-sharded index. The cost of this choice is lock-in: the plugins are why Tinder could not move to OpenSearch. ES8 also banned negative scores, which forced Tinder to audit 100+ scripts that had minimal tests. If you go this route, put your scoring scripts under the same test discipline as application code ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)).

**Shard by geography with cells, not cities.** S2-style cells sized to query radius cut CPU about fourfold. Raising shards per node and adding primaries for scoring-heavy queries traded roughly 2× CPU for 25–30% lower latency. Measure that trade-off explicitly.

**Make the event log the source of truth.** With Kafka as the record, an index becomes a rebuildable projection: backfill, then tail. That made a zero-outage engine migration possible. Tinder verified the result with random field-by-field document checks (<0.2% discrepancy) and evaluated rankers offline by **replaying real traffic and comparing top-20 and top-100 lists by set overlap and Levenshtein distance**, before running three waves of online A/B tests. For consumers, acknowledge after processing, not before, and use retry topics plus a dead-letter topic ([Tinder Eng](https://lifeattinder.com/blog/dig-deeper-to-scale-further)).

**Separate eligibility from competitiveness.** Hard filters decide who *may* appear, and the ranker decides who *wins*. Keeping the two separate let Tinder recycle passed profiles under caps without hand-tuned exclusion rules ([Tinder Eng](https://lifeattinder.com/blog/how-to-keep-recommendations-fresh)).

**Optimize the reciprocal objective.** In a two-sided market, P(Match) beat P(Like) on match volume (+22% vs +3.8% on swipe-right rate). If your product succeeds only when both sides say yes, model both sides.

**Use bandits as label factories.** Smart Photos' exploration data became clean pairwise labels for a VLM. Filtering to statistically significant pairs beat using more data. Pairwise framing also removes per-person baselines, since the model compares two photos of the same person.

**Push ML to the edge when the data is sensitive, and make the telemetry honest.** The on-device pipeline does this well: encrypted model delivery, cheap face gates before expensive scoring, and moderation only on the top 100. The schema tension is the lesson. If you promise "counts only," do not leave score arrays in your analytics schema.

**Make the server push signals, not state.** A protobuf WebSocket that sends "refresh X" nudges keeps clients simple and lets backends own freshness. Lazily loading protobuf schemas keeps the initial bundle small.

**Know what your ranker optimizes for socially.** A model trained on aggregate swipes inherits population bias, and tying popularity signals to paywalls invites regulatory and reputational scrutiny, as the age-pricing settlement and the 2026 DPC fine show. Publishing disparate-impact testing is cheap insurance that Tinder has so far skipped.

#### Conclusion

The common question "does Tinder still have an Elo score?" is the wrong one. Tinder no longer stores a single rating that you could request through a data-access request. Popularity is spread across learned P(Like)/P(Match) towers, a patented views-to-likes ratio, and an RSRR signal that the shipped client uses in exactly one place: showing a revenue-tagged upsell. That turns a transparency question into an architectural one. Each rating is now implicit in model weights, so users cannot inspect it and regulators cannot easily audit it. The monetization coupling is visible only because client code reveals where the signal reaches the paywall.

For the ayoships video, the lesson is about method rather than Tinder. Static artifacts such as strings, flags and schemas are reliable and were largely confirmed. Behavioural inferences drawn from probing an old client-side model are not. The most consequential open question is whether those per-photo score arrays leave the device. A disclosed privacy audit of the mobile client could settle it, and nobody has yet published one.

---

## Part B — Research notes: local code evidence

### Local code evidence: what Tinder's public web assets, REA analysis and scraped protobuf schemas show

Scope: static, read-only analysis of local files only. No network requests were made to Tinder. All "Cited Findings" were **observed in the artifact named** (identifier or field name quoted). "Inferences" are my interpretation.

Source path abbreviations used in citations:
- `T/` = `/tmp/claude-0/-home-user-AI-Art-Generator/55bc6ae2-90d3-5f05-a3db-e96f12c00733/scratchpad/tinder/` (public tinder.com files, fetched 2026-11-16 per `date:` header)
- `P/` = `/home/user/davis-3450/tinder-protos/` (third-party GitHub repo; README: "a collection of all protobuf files scraped from tinder.com ... for educational purposes only"; last commit `e3ef441` 2025-09-09; 212 .proto files)
- `H/` = `P/tinder/events/model/app/hubble/details/`

Provenance caveats (apply to every section):
- Protos repo is third-party and self-described as scraped; I could not verify it against Tinder's current schemas. Schemas prove that a field *exists in a schema*, not that any client populates it or how the server uses it.
- The web client (tinder.com) may differ from iOS/Android. Several schemas (e.g. Photo Selector) have **no** references in the downloaded web JS — see below — so they are most likely mobile-client telemetry.
- `T/chunks/pc-*.js` (pc-r-RecsPage, pc-r-LikesYou, pc-r-RecProfile, pc-RecsGamepad, pc-r-ExploreCuratedRecs) are **not JavaScript**: each begins `<!doctype html>` and is ~576 KB like index.html (SPA fallback HTML). Only `async-recs-feature.js`, `async-recs-index.js`, `async-likesYou-feature.js`, `async-match-feature.js` are real chunk code.
- REA graph coverage is self-declared incomplete ("Graph coverage is incomplete; unavailable facts remain explicit and must not be read as absence").

#### 1. Tech stack and delivery

##### Takeaway
tinder.com is a server-side-rendered React 18.3.1 + Redux/redux-saga webpack SPA ("tinderweb") served by Express behind nginx, AWS ALB and CloudFront, with heavy code splitting (2,089 named lazy chunks), a Workbox service worker with web push, Sentry 7.42.2 observability, REST against api.gotinder.com plus protobuf (`application/x-protobuf`) schemas loaded lazily for keepalive/roomservice/mediaservice/dynamicui/authgateway, and a CSP (report-only) that whitelists `wss://keepalive.gotinder.com`.

##### Cited Findings
- Response headers: `server: nginx`, `x-powered-by: Express`, `x-render-method: ssr`, `via: ... (CloudFront)`, `x-cache: Miss from cloudfront`, `x-amz-cf-pop: ORD56-P1`, AWS ALB sticky cookies `AWSALB`, `AWSALBTG`, `AWSALBCORS` — [T/headers.txt](T/headers.txt)
- Caching: `cache-control: must-revalidate, public, max-age=3600, s-maxage=3024000` (CDN TTL 35 days, browser 1 h) — [T/headers.txt](T/headers.txt)
- Security headers: `strict-transport-security: max-age=31536000`, `x-frame-options: SAMEORIGIN`, `x-content-type-options: nosniff`, `x-xss-protection: 1; mode=block`, `referrer-policy: origin-when-cross-origin`, `cross-origin-opener-policy: same-origin-allow-popups`, `document-policy: js-profiling` — [T/headers.txt](T/headers.txt)
- CSP is **report-only** (`content-security-policy-report-only`): nonce + `'strict-dynamic'` script-src but also `'unsafe-eval' 'wasm-unsafe-eval' 'unsafe-inline'`; `connect-src 'self' data: https: wss://keepalive.gotinder.com wss://keepalive.ue1.d1.tstaging.com`; `frame-ancestors 'self'`; `object-src 'none'` — [T/headers.txt](T/headers.txt)
- React version strings `version:"18.3.1"` and `"18.3.1-next-f1338f8080-20240426"`; `hydrateRoot` present (SSR hydration) — [T/js/vendor-a1e4cdb622cdfc65ee00.js](T/js/) and main bundle
- Sentry: `SENTRY_RELEASE={id:"7.42.2"}` and SDK `version:"7.42.2"`; flags `observability.sentry_performance_sample_rate_web`, `observability.sentry_profiling_sample_rate_web`, `observability.sentry_replay_sample_rate_web` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/flags.txt](T/flags.txt)
- Other identifiable libs: `version:"3.27.0"` (core-js shared versions registry), `immer` (39 hits), `reselect`, redux-saga effects (`yield(0,s.Lt)(...)`, `yield(0,s.yJ)(...)` saga pattern in recs chunk), `lottie`, `emotion` — grep over [T/js/](T/js/)
- SSR state globals in HTML: `window.__data`, `window.__intlData` — [T/index.html](T/index.html)
- Webpack chunk namespace `webpackChunktinderweb` — REA node identity `namespace:"webpackChunktinderweb:module"` in [T/rea-app.json](T/rea-app.json)
- Code-splitting scale: 2,089 chunk names; of these ~205 are `emoji-button-message-*`/`intl-message-*` locale chunks, 100 `pc-*` page/component chunks, 98 `async-*-feature/-index` Redux feature modules, 11 `intent-handler-*` — [T/chunknames.txt](T/chunknames.txt)
- REA statistics: 6 parsed JS files, 814 modules, 3,751 findings, 460,655 AST nodes; graph 1,809 nodes (1,392 javascript-module, 221 endpoint, 178 javascript-chunk, 7 storage, 1 worker) and 4,762 edges; semantic_graph 41,580 nodes/20,564 relations; provider "REA JavaScript application analyzer" v1, `authority:"shipped-artifact"`, `confidence:"derived"` — [T/rea-app.json](T/rea-app.json) `records[0].normalized_result.statistics/graph`
- REA storage nodes: `call:indexedDB.open`, `call:window.sessionStorage.setItem`, `call:window.sessionStorage.clear`, `call:caches.open`, `call:self.caches.open`, `what-intent`, `what-input`; worker node `new Worker(t)` — [T/rea-app.json](T/rea-app.json)
- REA route_inventory: 51 route patterns incl. `/app/checkout`, `/app/product/subscription/{gold,platinum,plus}`, `/app/settings/{gold,platinum,plus}`, `/app/settings/plus/passport`, `/app/promo`, `/feature/stand-out`, `/feature/subscription-tiers`, `/giftcard/redeem`, `/swipe-stories` — [T/rea-app.json](T/rea-app.json) `route_inventory.routes`
- API base: `apiBase:"https://api.gotinder.com"` (client constructed with `apiToken`, `deviceId`, `build`, `acceptLanguage`); `imageProxyURL:"https://images-ssl.gotinder.com"`; analytics batch endpoint `sparks:{endpoint:"https://data.gotinder.com/v2/batch/event", v3endpoint:...}` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js)
- Protobuf over HTTP: `protobufServicesPath:["tinder","services"]`, MIME `application/x-protobuf`; lazy schema chunks `async-protobuf-schema-authgateway`, `-dynamicui`, `-keepalive`, `-mediaservice`, `-roomservice` — main bundle and [T/chunknames.txt](T/chunknames.txt)
- Real-time: websocket token endpoint `getWebsocketToken:{path:"/ws/generate"}`; redux action namespace `websocket`; `keepalive` case loads chunk 60404 = `async-protobuf-schema-keepalive` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js). The wss host itself appears only in the CSP header, not as a literal in the JS.
- Keepalive proto messages (22 files): `Nudge`, `Update`, `ClientData`, `TypingIndicator`, `RoomTypingIndicator`, `StatusChange`, `Notification`, `InAppBanner`, `InsendioCampaign`, `DynamicUiNudge`, `MediaHydrationNudge`, `RefreshProfileNudge`, `RefreshPullCampaign`, `DirectMessageNudge`, `ConnectNudge`, `RoomEvent`, `RoomReadyNudge`, `RoomInteractionNudge`, `EmailCollectionSettings`, `TinderLiteData`, `BackendWrapper` — [P/tinder/services/keepalive/](P/tinder/services/keepalive/)
- Event publishing schema: `AppEventPublishRequest`, `AppEventPublishResponse`, `AppEventValidationResult`, `EventPublishRequestAttributes` — [P/tinder/events/service/](P/tinder/events/service/)
- Service worker: Workbox (`workbox-routing`, `workbox-expiration`, `workbox-background-sync`, `workbox-google-analytics`, precache controller); listeners `install`, `activate`, `fetch`, `push`, `notificationclick`, `sync`, `message`; `showNotification`, `skipWaiting` — [T/service-worker.js](T/service-worker.js)
- PWA manifest: `display:"standalone"`, `gcm_sender_id:"465293127427"` (FCM push), related apps `com.tinder` / iTunes `id547702041`; description claims "55 billion matches to date" — [T/manifest.json](T/manifest.json)
- Third-party hosts referenced: `tinder-api.arkoselabs.com`, `www.google.com/recaptcha`, `connect.facebook.net`, `www.googletagmanager.com`, `analytics.tiktok.com`, `sc-static.net` (Snap), `s.pinimg.com` (Pinterest), `onelinksmartscript.appsflyer.com`, `tinder.cashstar.com` (gift cards), `fcm.googleapis.com`, `messagepublish.ue1.d1.tstaging.com` (staging) — main/vendor bundles and index.html

##### Inferences
- The `keepalive` websocket appears to be the push channel for server-driven "nudges" (refresh profile, dynamic UI, campaigns, typing, rooms) encoded with protobuf, while most CRUD (swipes, purchases) is JSON REST. This is consistent with the CSP and schema chunk names, but the actual wire framing was not observed.
- CSP being report-only with `unsafe-inline/unsafe-eval` means it is monitoring rather than enforcing.

##### Gaps
- Exact Redux version, router library and CSS framework not confirmed (no clear version strings found). `12.20.0` string is a theme object version (`themeId`), not identified.
- REA endpoint nodes are "candidate" string literals; no HTTP methods/edges were extracted from REA itself (methods came from the main bundle's endpoint maps).

#### 2. Swipe telemetry and per-recommendation server fields

##### Takeaway
On every like/pass the web client builds a `recsStatusInfo` payload with dwell time (`duration_in_millis`), input `method` (button/keyboard/swipe/profileButton/profileKeyboard), media and profile-element view counts, the order and values of profile elements seen, shared interests, and the server-supplied `s_number`; the schema `RecsStatusDetails` (Hubble field 100) and `RecsCustomDetails` (field 114) define the full set. Server rec objects carry `s_number`, `is_superlike_upsell`, `is_traveling`, `recently_active`, `matched_preferences`, `teaser`, `experiment_info.user_interests`, and `liked_content` (swipe notes).

##### Cited Findings
- Recs saga dispatches `recsStatusInfo:{... uid, category_id, duration_in_millis, method (default recsRateMethods.button), source_session_event, profile_element_views_card_users, profile_element_count_users, media_views_card_users, viewed_profile_elements_users ...}` (duo variant shown with `_users` arrays) — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Further per-swipe arrays built by helper `Ve`: `tappy_element_values_users`, `value_profile_element_bio_users`, `value_profile_element_interests_users`, `value_profile_element_anthem_users`, `value_profile_element_top_artists_users`, `value_profile_element_intent_users`, `value_profile_element_descriptors_users`, `value_profile_element_identity_users`, `profile_element_order_users`, `shared_interests_users` (from `experiment_info.user_interests.selected_interests` filtered by `is_common`), `page_viewed_durations_users` (sent as empty `[]` in this build) — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Input methods: `recsRateMethods.button`, `.keyboard`, `.swipe`, `.profileButton`, `.profileKeyboard` — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Rate-event analytics keys: `otherId`, `photoId`, `method`, `sNumber`, `superlikeType`, `didSuperLike`, `teaserType`/`teaserValue` (from `e.teaser.type/.string`), `recTraveling`, `userTraveling`, `isRecentlyActive`, `country`, `distance`, `videoCount`, `audibleVideoCount`, `availableMediaList`, `viewedProfileElements`, `hasPremiumPreferences`, `valueProfilePremiumPreferencesElement`, `valueProfilePremiumPreferencesFulfilled`, `viewedProfilePremiumPreferencesFulfilled`, `valueProfileElementCollectibles/Descriptors/Geo/Intent/Interests` — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Like request query params: `is_boosting`, `super`, `fast_match`, `s_number`, `rec_traveling`, `user_traveling`, `liked_content_id`, `liked_content_type`, `swipe_note` — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Rec object fields read from server: `_id`, `is_super_like`, `s_number`, `country`, `is_traveling`, `recently_active`, `photos`, `is_superlike_upsell`, `matched_preferences`, `teaser`, `experiment_info`, `duo_group.participants`, `liked_content.by_opener/by_closer.is_swipe_note`, `distance_mi`, `selected_descriptors`, `connections` — [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Endpoints: `/v2/recs/core`, `/like/{userId}`, `/like/{userId}/super`, `/pass/{userId}`, `/v2/like/{userId}/swipenote`, `/v1/group-ratings` (`swipeDuosRec`, POST), `/v2/explore`, `/v2/explore/recs`, `/v2/fast-match`, `/v2/fast-match/teaser(s)`, `/v2/fast-match/count`, `/dynamicui/tappycontent`, `/v2/buckets` (`preAuthBuckets`), `/stacksutil/run`, routes prefix `levers:"/levers"` — main bundle; REA endpoint nodes in [T/rea-app.json](T/rea-app.json)
- Schema `RecsStatusDetails`: `category_id, s_number(int64), profile_element_views_card, profile_element_count, viewed_profile_elements, value_profile_element_{collectibles, identity, bio, anthem, top_artists, geo, instagram, interests, swipe_surge, vibes, intent, descriptors}, profile_element_order, num_interests_collapsed, shared_interests, tappy_element_values, is_traveling, duration_in_millis(int64), media_views_card, media_views_profile, source_session_event, page_viewed_durations, method, value_prompted_bio` — [H/recs_status_details.proto](H/recs_status_details.proto)
- Schema `RecsCustomDetails`: `s_number, rec_traveling, exceeds_age_range_by, exceeds_distance_by, other_id_is_top_pick, connections_mystery, connections_names, connections, fof_interact_profile, real_media_views_card, has_premium_preferences, value_profile_premium_preferences_element, viewed_preferences_profile_detail, referral_string, value_profile_elements_height, value_profile_elements_pronoun, did_super_like, superlike_type, has_likes_you, swiped_media_index, distance_mi` — [H/recs_custom_details.proto](H/recs_custom_details.proto)
- `RecsCoreDetails`: `request_index, discovery_index, recs_cards_count`; `RecsPhotoDetails`: `photo_id, url, index`; `MediaDownloadDetails`: `media_id, media_type (PHOTO/LOOP/VIDEO), photo_index, photo_source (CARD/PROFILE/NEW_MATCH), media_url, fetch_time`; `PushedRecsFetchErrorDetails`: `source, status_code, push_id, ...` — [H/](H/)
- Hubble `InstrumentDetails` oneof numbering: `recs_status_details = 100`, `recs_custom_details = 114`, `recs_core_details = 20`, `recs_photo_details = 2`, photo_selector_* = 106–113, `paywall_purchase_details = 500`, `revenue_purchase_flow_details = 607` — [P/tinder/events/model/app/hubble/instrument_details.proto](P/tinder/events/model/app/hubble/instrument_details.proto)
- Flags `events.sdk_recs_hubble_is_enabled_web`, `events.sdk_paywall_hubble_is_enabled_web`, `events.sdk_is_enabled_web` — [T/flags.txt](T/flags.txt)

##### Inferences
- `s_number` is a server-issued per-rec identifier echoed back on swipe (likely a ranking/session or experiment cohort token); its semantics are not visible client-side.
- `exceeds_age_range_by` / `exceeds_distance_by` in the schema imply recs can be served outside a user's stated preferences and that this is logged.
- Dwell time, media views and which profile elements were viewed are exactly the implicit-feedback signals a ranking model would consume; the artifacts prove collection, not use in ranking.

##### Gaps
- No ranking scores, Elo/desirability values, or embeddings appear in rec payloads or schemas (searched `score|rank|embedding|elo|desirab|chemistry|smart_photo` across all 212 protos; only Photo Selector scores found).
- How `matched_preferences`, `teaser`, `s_number` are computed server-side is not observable.

#### 3. Photo Selector analytics schemas and RSRR

##### Takeaway
Eight `photo_selector_*` Hubble schemas describe an on-device (downloaded-model) camera-roll photo picker that scores photos with per-photo arrays `rsrr_scores`, `explicit_scores`, `text_scores`, `underage_scores`, `similarity_scores`, then filters by moderation (explicit, text, under-age, baby, less-clothed, mirror-selfie). On web, "RSRR" appears only as a Super Like upsell trigger (`superlike.upsell_high_rsrr_enabled`, `upsell_superlikes_high_rsrr`), tagged with `sourceSessionEvent:"revenue"`.

##### Cited Findings
- `PhotoSelectorPhotosCollectionDetails`: `total_number`, `repeated rsrr_scores (FloatValue)`, `repeated photo_ids`, `repeated explicit_scores`, `repeated text_scores`, `repeated underage_scores`, `repeated similarity_scores` — [H/photo_selector_photos_collection_details.proto](H/photo_selector_photos_collection_details.proto)
- `PhotoSelectorUploadSelectedPhotosDetails`: `number_of_photos`, `percentage_of_scrolling`, `repeated indices_of_selected_photos`, `repeated ids_of_selected_photos`, `repeated rsrr_scores`, `repeated is_image_cropped`, `repeated cropped_info`, `repeated orientation` — [H/photo_selector_upload_selected_photos_details.proto](H/photo_selector_upload_selected_photos_details.proto)
- `PhotoSelectorModerationSelectionDetails`: `process_time, num_photos_entering, num_photos_returned, num_photos_after_deduped, num_photos_after_moderation, exit_status (SUCCESS/CANCELLED), num_photos_explicit_filtered, num_photos_text_filtered, num_photos_under_age_filtered, num_photos_less_clothed_flagged, num_landscape_photos_returned, num_portrait_photos_returned, num_photos_other_filtered, num_photos_baby_filtered, num_photos_mirror_selfie_flagged` — [H/photo_selector_moderation_selection_details.proto](H/photo_selector_moderation_selection_details.proto)
- `PhotoSelectorAssetProcessingDetails`: `process_time, num_photos_granted_access, num_photos_scanned, num_photos_returned, num_photos_face_detected, num_photos_selfie_identified, num_photos_scored, exit_status (ALL_SCANNED/CANCELLED/TIME_OUT), num_landscape/portrait_photos_granted_access` — [H/photo_selector_asset_processing_details.proto](H/photo_selector_asset_processing_details.proto)
- `PhotoSelectorModelDownloadDetails.model_download_outcome` (SUCCESS/FAILURE); `PhotoSelectorOutcomeDetails.outcome`; `PhotoSelectorSelectedPhotoDetails`: `photo_index, percentage_of_scrolling, is_selected`; `PhotoSelectorFeedbackDetails`: `length_of_feedback, feedback` — [H/](H/)
- Related: `MandatedFacePhoto*` schemas (`modal_type`, `number_days_left_to_add`, `face_upload_detected`); `UploadedPhotosDetails.has_face_photo_required` — [H/](H/)
- No `photo_selector`/`photoSelector` strings in any downloaded web JS — grep over [T/js/](T/js/), [T/chunks/](T/chunks/), [T/index.html](T/index.html)
- Web RSRR occurrences (6 total): flag constant `superlike.upsell_high_rsrr_enabled` (main); Redux upsell state `superlikeHighRsrr:{amountShown:0,shown:!1,timestamp:0}` with selectors on `.shown`/`.timestamp` (main); saga dispatch `upsellType:"superlikeHighRsrr", upsellTs:Date.now()` (main); analytics `{location:"discovery", name:"upsell_superlikes_high_rsrr", sourceSessionEvent:"revenue", type:"modal"}` with `action:"show"` in the `SuperlikeUpsellModal` chunk — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/js-upsell.js](T/js-upsell.js); 1 occurrence in [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Related flags: `superlike.upsell_rate_limit_unit`, `superlike.attach_message_rate_limit`, `superlike.attach_message_targeting`, `superlike.attach_message_time_limit_sec`; rec field `is_superlike_upsell` — [T/flags.txt](T/flags.txt), [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)

##### Inferences
- "RSRR" is not expanded anywhere in the artifacts. Its pairing with photo scores and with a "high RSRR → upsell Super Like" modal is consistent with a *right-swipe-rate/ratio*-type predicted metric (e.g., "received swipe right rate"), but this expansion is **interpretation, not observed**.
- The web upsell suggests the server flags certain recs (likely ones predicted to be in high demand) as Super Like upsell candidates (`is_superlike_upsell`) — a monetization lever tied to a desirability-like signal.
- Photo Selector appears to be a mobile, on-device ML feature (model download, camera-roll access counts, face/selfie detection) whose per-photo scores are sent as analytics.

##### Gaps
- No artifact defines RSRR, the model, score ranges or whether scores influence ranking.

#### 4. Monetization and experimentation levers

##### Takeaway
Feature flags ("levers", served from a `/levers` route prefix) gate a large paywall/upsell surface: dynamic/personalized offers, discounts, bouncer paywalls, priority likes, Super Like upsells, match expiration and Likes You filtering. Web payments run through Braintree (card, Apple Pay, PayPal, Venmo) and Adyen (card, Apple Pay, Google Pay, 3DS), plus Blackhawk gift cards.

##### Cited Findings
- Flag namespaces and notable flags: `paywall.alc_discount_offer_paywall_enabled`, `paywall.alc_discount_offer_paywall_cool_down_period_in_seconds`, `paywall.alc_discount_offer_paywall_rate_limit_value`, `paywall.boost_subscription_enabled`, `paywall.premium_rebrand_*_web`, `paywall.show_terms_with_refund`; `purchase.cancel_save_offer_enabled_web`, `purchase.cc_discount`, `purchase.one_page_checkout_enabled`, `purchase.web_google_pay_steering_enabled`, `purchase.web_venmo_subscription_enabled`, `purchase.web_apple_pay_skip_braintree_fingerprint`; `bouncer.bounce_to_gold`, `bouncer.bouncer_bypass_enabled`, `bouncer.bouncer_bypass_modal_variant`; `subscription_discount.personalized_offer_enabled`, `subscription_discount.personalized_offer_ny_enabled`; `discount.banner_enabled`, `discount.checkout_pill_enabled`, `discount.login_banner_enabled`; `gold.new_likes_insendio_modal_enabled`, `gold.new_likes_modals_frequency`, `gold.weekly_intro_offer_enabled`; `platinum.priority_likes_swipe_right_threshold`, `platinum.priority_likes_modal_time_interval`, `platinum.match_list_upsell_enabled`, `platinum.weekly_gold_to_plat_offer_enabled`, `platinum.(gold_)bouncer_paywall_cta_enabled`; `likes_you.filtering_enabled`, `likes_you.min_likes_count_to_enable_filtering`, `likes_you.quick_filter_passions`; `postmatch.match_expiration_enabled`, `postmatch.match_expiration_extend_enabled`, `postmatch.match_expiration_urgency_period_seconds`; `pricing.show_weekly_savings`; `web_steering.open_ios_paywall`, `web_steering.web_discount_percentage`; `first_impression.enabled`; `ads.web_ads`, `ads.web_banner_cadence`, `ads.web_cardstack_cadence`; `logging.should_sample_hubble_revenue_purchase_flow_events` — [T/flags.txt](T/flags.txt)
- Upsell/paywall chunk names: `SuperlikeUpsellModal`, `SuperlikePaywallDialog`, `PersonalizedOfferModal`, `CancelSaveOfferModal`, `BoostPaywallModal`, `BoostUpsellModal`, `BoostSubscriptionPaywall`, `PrimetimeBoostModal`, `PrimetimeBoostUpsellModal`, `SuperBoostPaywallModal`, `YForXBoostModal`, `FirstImpressionsPaywallModal`, `BundleOfferBanner/Dialog`, `IntroPriceAndSubOfferBanner`, `GoldHomeFilterTrayModal`, `pc-PlatinumPayWallDialogWithFeaturesList`, `pc-r-CheckoutPage`, `pc-r-PreCheckoutPage`, `remote-checkout`, `async-paywalls-*`, `async-offers-*`, `async-purchase-*`, `async-boost-*`, `intent-handler-getGold/goldHome/purchaseFlow/startBoost`, `paypal-checkout(-v6)-index-js`, `visa-checkout-index-js` — [T/chunknames.txt](T/chunknames.txt)
- Payment provider enum: `web_braintree_applepay`, `web_braintree_paypal`, `web_braintree_venmo`, `web_adyen_card`, `web_adyen_applepay`, `web_adyen_googlepay`, `web_blackhawk_giftcard`, `web_discount`, `apple_store`, `android_store`; `PAYMENT_TYPE_{APPLE_PAY, CARD, CREDIT_CARD, GOOGLE_PAY, PAYPAL, VENMO}`; Adyen 3DS state `adyenV2ThreeDSResult` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js)
- Purchase endpoints: `/v1/paywall/offers`, `/purchase/offer`, `/purchase/offer/view`, `/v2/purchase/braintree`, `/v2/purchase/braintree/client-token`, `/v2/purchase/adyen`, `/v2/purchase/adyen/payment-details`, `/v2/purchase/discount/view`, `/v2/purchase/tax/{zip}`, `/v2/purchase/{purchaseId}/auto-renew`, `/v2/consumables/bouncer-bypass`, `/v2/incentives/introprice/redeem`, `/v2/incentives/eligibility`, `/v1/incentives/gift-card/redemption`, `/v2/insendio/campaigns` — main bundle; REA endpoint nodes [T/rea-app.json](T/rea-app.json)
- Schema `PaywallDetails.FeatureType`: BOOST, GOLD, PLATINUM, REWIND, SUPER_BOOST, SUPER_LIKE, READ_RECEIPTS, PLUS, BOOST_SUBSCRIPTION, SUB_MERCHANDISING, CONSUMABLE, SUBSCRIPTION, PRIME_TIME_BOOST, LIKES_ANNUITY, MATCH_EXPIRATION, PURCHASED_LIKE; fields `paywall_version, upsells, template_ids` — [H/paywall_details.proto](H/paywall_details.proto)
- Schema `RevenuePurchaseFlowDetails` includes `age`, `base_price`, `currency`, `discount_campaign`, `is_intro_pricing`, `likes_num`, `paywall_from`, `paywall_version`, `promo_code`, `required3ds`, `revenue_session_id`, `sku`, `template_uuids`, `upsells` — [H/revenue_purchase_flow_details.proto](H/revenue_purchase_flow_details.proto)
- `CrmCampaignDetails`: `campaign_id, experiment_name, variant_name, content_branch` — [H/crm_campaign_details.proto](H/crm_campaign_details.proto)

##### Inferences
- `platinum.priority_likes_swipe_right_threshold` and `gold.new_likes_modals_frequency` suggest upsell timing is tied to the user's own swiping activity and incoming-like counts.
- `age` in purchase-flow telemetry plus `subscription_discount.personalized_offer_enabled` is consistent with segmented/personalized pricing; the artifacts do not show the pricing logic.

##### Gaps
- Flag values (on/off, variants) are not in static assets; only names.

#### 5. Trust and safety

##### Takeaway
The web client integrates Arkose Labs and Google reCAPTCHA (backend-selectable), FaceTec-based biometric liveness and selfie/ID verification, Jumio-backed age verification flows (incl. a Japan-specific underage flow), client-side automation/bot detection ("agent signals" checking webdriver/headless/Selenium/Nightmare artifacts), and a detailed ban-reason taxonomy and appeals system.

##### Cited Findings
- Captcha: constants `"recaptcha"`, `"arkose"`, `CAPTCHA_V2`; `arkoseCaptcha:{baseUrl:"https://tinder-api.arkoselabs.com/v2", ...onboardingKey...}`; reCAPTCHA `scriptUrl:"https://www.google.com/recaptcha/api.js?..."`; `SET_ARKOSE_DATA_BLOB`; endpoint `/v2/auth/verify-captcha`, `/challenge/verify`; flags `captcha.logic`, `identity.backend_driven_web_captcha_enabled`, `identity.auth_guest_arkose_enabled` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/flags.txt](T/flags.txt)
- Bot/agent detection: arrays checking `"selenium","webdriver","driver"`, `/HeadlessChrome/`, `__webdriver_evaluate`, `__selenium_unwrapped`, `$cdc_asdjflasutopfhvcZLmcfl_`, `__nightmare`, empty `navigator.languages`; flags `integrity.agent_signals_enabled`, `integrity.agent_signal_block`; chunks `agent-signals`, `AgentBlockedDialog`; fingerprint keys in flag list `navigator.webdriver/platform/vendor/languages`, `webgl.renderer/vendor/extensions/version`, `screen.width/height`, `timezone.*`, `headless.webdriver`, `math.engine`, `fonts.available` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/flags.txt](T/flags.txt), [T/chunknames.txt](T/chunknames.txt)
- Liveness/biometrics: endpoints `/v1/biometrics/liveness`, `/v2/biometrics/liveness`, `/v1/biometrics/liveness/consent`, `/v1/biometrics/reset/liveness`, `/v2/verification/facemap/{info,id-verify,photo-match}`, `/v2/verification/selfie/{start,complete}`, `/v1/identity-verification/{sdk/init, idv/challenge, idv/challenge-status, selfie/challenges, selfie/id-verify, selfie/verifications}`, `/match-settings/selfie-verification/{matchId}`; FaceTec UI strings `FaceTec_camera_permission_header`; flags `integrity.biometrics_liveness_bouncer_enabled`, `integrity.liveness_check_m2_variant`, `identity.biometrics_liveness_blocking_skippable`, `identity.liveness_data_removal_enabled`, `trust.facetec_sdk_version`, `trust.selfie_verification_enabled`, `trust.idv_variant`, `trust.selfie_v2_biometric_consent_screen_version`; chunks `BiometricsBouncer`, `BiometricsLivenessCheckModal`, `SelfieChallengeV2Modal` — main bundle, [T/flags.txt](T/flags.txt), [T/chunknames.txt](T/chunknames.txt)
- Age verification: `/1.0/age-verification/url`, `/verification/underage/{url,state,jp,jp/status}`, `generateJumioUrl`, flags `age_verification.eureka_age_verification_enabled`, `age_verification.post_onboarding_enabled`, `onboarding.age_gate_jumio_appeal_enabled`; chunks `AgeVerificationEurekaDialog`, `AgeGateDialog` — main bundle, [T/flags.txt](T/flags.txt)
- Other trust flags: `integrity.csh_v1_enabled`, `integrity.eu_mlc_variant`, `integrity.preauth_sk_consent_enabled`, `trust.dsa_custom_ban_screen`, `trust.checkr_rso_enabled` (Checkr; "rso" plausibly registered sex offender — interpretation), `trust.appeal_center_access_control`, `safety.chat_shield_fab_*`, `postmatch.blur_expired_match_messages` — [T/flags.txt](T/flags.txt)
- Ban/violation taxonomy in web: `VIOLATING_PERSONAL_CONTENT`, `VIOLATING_MEDIA`, `HARMFUL_BEHAVIOR`, `FALSE_REPORTING`, `MULTIPLE_USERS`, `IMPERSONATION_ACCOUNT`, `IMAGES_OF_OTHER_PEOPLE`, `COMPROMISED_ACCOUNT`; schema `BanReason` (24 values incl. COMMERCIAL_SEX, UNDERAGE, UNDERAGE_IMAGES, STOLEN_PHOTOS, ATO, PREVIOUSLY_BANNED) — main bundle, [H/ban_reason_details.proto](H/ban_reason_details.proto)
- Appeals/reports endpoints: `/v1/appeals`, `/v1/banned-user/appeals/cases`, `/v1/case/warnings/acknowledge`, `/v3/report/{primaryId}/{secondaryId}/{offenderId}`, `/v4/report/new`, `/v4/report/tree` — REA endpoints [T/rea-app.json](T/rea-app.json)

##### Inferences
- Device/environment fingerprint attributes (WebGL, screen, timezone, fonts) are collected for integrity scoring; whether they feed Arkose or an in-house model is not determinable statically.

##### Gaps
- No moderation model scores observed in web code; Photo Selector moderation counts are the only ML-moderation telemetry found (schema only).

#### 6. Other algorithm-relevant schema fields (survey of hubble/details and related protos)

##### Takeaway
Beyond recs and Photo Selector, schemas reveal top-picks, "swipe surge", "vibes", Let's Meet (date intent), duration/session measures and server-driven UI nudges — but no explicit ranking score, embedding or "chemistry" field exists anywhere in the 212 protos.

##### Cited Findings
- 39 files in hubble/details (full list incl. `lets_meet_*`, `mandated_face_photo_*`, `hubble_duration_measure`, `network_request_details`, `share_my_date_details`) — [H/](H/)
- `HubbleDurationMeasure`: `category, type, subtype, duration, success, component_name, duration_millis, session_id`; `AppCloseDetails.session_length_seconds` — [H/](H/)
- Top Picks: `RecsCustomDetails.other_id_is_top_pick`; enums `TOP_PICKS`, `TOP_PICKS_ALC_ACCESS`, `TOP_PICKS_INTRO_V3` in [P/tinder/services/profile/profile_options.proto](P/tinder/services/profile/profile_options.proto); `SESSION_ATTRIBUTION_TOP_PICKS` in [P/tinder/events/model/common/message.proto](P/tinder/events/model/common/message.proto)
- Swipe Surge / Vibes: `value_profile_element_swipe_surge`, `value_profile_element_vibes` (RecsStatusDetails); `TAPPY_CONTENT_TYPE_SWIPE_SURGE` in [P/tinder/services/dynamicui/tappy_content/tappy_content.proto](P/tinder/services/dynamicui/tappy_content/tappy_content.proto)
- Let's Meet: `interested_in_date`, `is_matched`, `time_frame_and_description`, `date_suggestion_id` — [H/lets_meet_date_details.proto](H/lets_meet_date_details.proto)
- Server-driven UI: `tinder/services/dynamicui/clientnudge/{client_nudge_action,client_nudge_event}.proto`, insendio campaign DSL (`insendio/dsl/attribute`, `campaign/templates`) — [P/tinder/services/](P/tinder/services/)
- Explicit search for `score|rank|embedding|smart_photo|chemistry|elo|desirab|attractiv|dwell|algorithm|model_version` across all protos returned only Photo Selector score fields and the Top Picks/swipe-surge/vibes items above — grep over [P/](P/)

##### Inferences
- The client-visible surface is telemetry plus server-issued tokens; ranking happens server-side and is not exposed in client schemas.

##### Gaps
- No artifact shows Smart Photos ordering logic, Elo, or chemistry scoring.

---

## Part C — Research notes: Tinder official engineering

### Tinder official engineering sources — notes

Compiled from first-party Tinder engineering posts read in this session (lifeattinder.com, mlconf.com) plus secondary summaries where noted. [P] = read from primary source; [S] = secondary/snippet only. Posts on lifeattinder.com show no dates in fetched text; index reflects a 2026 snapshot.

#### Infrastructure
- **Kubernetes migration** (Tinder Eng, Medium, 2019) [S]: moved from EC2 to Kubernetes; goals: containerization, immutable deployments, low-touch ops. https://medium.com/@TinderEng/tinders-move-to-kubernetes-cda2a6372f44
- **TAG — Tinder API Gateway** (Tinder Tech Blog, Oct 2022) [S via ByteByteGo/Better Stack/Nordic APIs; Medium 403]: JVM framework on Spring Cloud Gateway; config-driven — each app team builds its own gateway instance; pre-filters → route via Envoy service discovery → post-filters; 500+ microservices on an Envoy service mesh; built for >1B swipes/day; evaluated and rejected AWS API GW, Apigee, Kong, Tyk, KrakenD, Express Gateway. https://medium.com/tinder/how-we-built-the-tinder-api-gateway-831c6ca5ceca ; https://blog.bytebytego.com/p/how-tinders-api-gateway-handles-a
- **Envoy global rate limiting** talk, EnvoyCon 2020 [S]. https://envoycon2020.sched.com/event/eccm/how-tinder-implemented-envoy-global-rate-limiting-at-scale
- **Kafka** as event backbone (see ES8 and Smart Photo posts) [P].

#### Recommendations ("Recs")
##### Geosharded recommendations (Tinder Tech Blog, 2019, 3 parts) [S; Medium 403]
- Part 1 (Frank Ren): sharding approach; secondary summaries say shards are Google S2 cells, queries hit nearby cells sized by radius. Part 2 (Xiaohu Li): all geoshard indices in one cluster with load balancing. Part 3 (Devin Thomson): documents move between indices → guaranteed write ordering + strongly consistent reads.
- https://medium.com/tinder/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a ; part-2 …3396a8a7efb ; part-3 …2d2cb2f0594b ; https://highscalability.com/designing-tinder/

##### Tinder's migration to Elasticsearch 8 [P] — Igor Sokolov, Jessica Hickey, Rongxin Du
https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8
- ES powers Recs, Trust & Safety, logging. "Over 90% of all our Recs comes from a single Elasticsearch cluster."
- Two capabilities: filter-based search over millions of profiles (location, age, interests, preferences); ranking via custom Java "Tinder ES plugin" running ML-based scoring; 100+ scoring scripts with minimal tests; ES8 bans negative scores.
- Geo-sharded index: 12% CPU vs 50% single index. Shards/node 3→14; primary shards 1→2 for scoring-heavy queries (CPU ~2×, response time −25–30%).
- ECK on Kubernetes, in-house "Scaffold" IaC; first ECK cluster in prod 2022. ES6 previously via Puppet + manual EC2. Stayed off OpenSearch (custom plugins).
- Write path: Kafka topic = source of truth; esreindexjob backfill then real-time worker; random doc field-by-field verification.
- Offline eval via event-based traffic replay; set overlap + Levenshtein list comparison at top-20/top-100. Three waves of online A/B.
- Results: zero outages; <0.2% data discrepancy; kNN vector search enabled; **Two-Tower "2T P(Match)": +6.5% match rate, +22% match volume; "2T P(Like)": +3.8% Swipe Right Rate (SRR)**; profile-embedding "MoreLikeThis"; p99 latency −12–56%; "Million plus dollar annual cost savings"; cross-AZ costs up. Node c6id.8xlarge→16xlarge.

##### Keeping recommendations fresh [P] — Ruicong Yan (Staff PM), Lichao Liu (Staff SWE ML)
https://lifeattinder.com/blog/how-to-keep-recommendations-fresh
- Pipeline named explicitly: **eligibility filtering → retrieval → ranking → reranking**.
- "Dynamic recycling": previously passed profiles can resurface; recent passes stay blocked; older passes return via limited tiers; unseen prioritized; resurfacing capped.
- Per-pair serving state: last interaction time, pass or not, times shown within rolling window ("seen" logic).
- Separates "eligibility" from "competitiveness". Letting newest deep-learning retrieval/ranking models rank recycled candidates increased eligible pool and engagement. No numbers.

##### Recommendation embeddings [P] — Kevin Thomas (EM II), AI Day project
https://lifeattinder.com/blog/recommendation-embeddings
- Recs-team models learn high-dimensional user representations "from profile and activity signals"; architecture unnamed. "Recommendation embeddings were built to improve recommendations, not to explain people."
- Databricks notebook: sample → cohort filter → PCA → K-means (HDBSCAN rejected: continuum) → UMAP → cluster profiling.
- Cohort >300,000 US Gen Z women 18–29, K=5. Largest cluster 25.7%: longer bios, more photos, higher verification, more likes sent, but fewer likes/swipes/conversations received — "profile-rich, high-outbound, lower-reciprocity". Linked to persona "Shelby / Sincere Seeker". Explicit limits: regions not natural types.

##### TinVec (MLconf SF 2017) [P abstract] — Dr. Steve Liu, Chief Scientist
https://mlconf.com/?p=8768
- "embeds users' preferences into vectors leveraging on the large amount of swipes"; 26M matches/day, >20B matches to date (2017). Slides on SlideShare (not read). Reportedly powered "Super Likeable" (TechCrunch 2017, [S]).

##### Elo retirement — "Powering Tinder: The Method Behind Our Matching" (2019, updated 2022) [S; help.tinder.com fetch failed]
- Elo "outdated", no longer relied on; system adjusts potential matches as your profile is liked/passed; changes within ~24h. https://www.help.tinder.com/hc/en-us/articles/7606685697037-Powering-Tinder-The-Method-Behind-Our-Matching ; https://www.engadget.com/2019/03/18/tinder-dumps-desirability-scores

#### Photo ML
##### Smart Photos (2016) [S]
- Mike Hall (ML lead): photo order as multi-armed bandit, epsilon-greedy; exploration then exploitation; epsilon decay. Test group up to +12% matches (Tinder claim). https://www.engadget.com/2016/10/13/tinder-smart-photos-uses-swipe-data-to-select-your-best-pic/

##### How Tinder Uses AI to Rank Profile Photos [P] — Trideep Rath, Skanda Suresh, Ben Shin
https://lifeattinder.com/blog/how-tinder-r-uses-ai-to-rank-profile-photos
- Open-source VLM (vision encoder + LLM), unnamed. Pairwise: which of two photos of same person performs better; optional prompt context age, gender, target gender, country. Output token probabilities; SFT on label token; LoRA.
- Inference: all N-choose-2 pairs; aggregate by win count + mean pairwise probability.
- Labels: multi-armed bandit rotates top photo → per-photo likes/impressions; high-confidence significant pairs only (beat larger noisy sets).
- Pairwise acc ~76% (prior DL ~68%); ROC-AUC ~0.84; top-photo acc +7–10pp offline; online A/B: more likes, matches, conversations.
- LoRA rank 32 best (8–64); larger batch better; higher resolution helps; training vision encoder biggest gain; profile features ~+0.2pp (+0.8pp at 2× data); target gender for men-seeking-men +1.8pp; ≤3B models struggled, 7B stable. In-house LLM serving (continuous batching, KV cache, quantization).

##### Smart Photo + Kafka [P] — Gabriela Rodriguez-Florido
https://lifeattinder.com/blog/dig-deeper-to-scale-further
- VLM comparisons on GPUs, computed asynchronously: Kafka worker consumes photo-update events, recomputes, stores order. Legacy consumer acked before processing → rebuilt on Spring Kafka, AckMode.RECORD, retry topics + DLT. Results: fewer errors, lower GPU cost (no numbers).

##### LLM features (Photo Feedback, hybrid Smart Photo) [P] — Siva Periyasamy
https://lifeattinder.com/blog/llm-features-you-can-bet-on
- Smart Photo hybrid: tuned LLM low-60s pairwise acc vs specialist model mid-70s; hybrid not shipped.
- Photo Feedback: human 1–5 ratings → majority vote labels; ~60% inter-rater agreement ceiling; precision over recall (74% example); failure modes: artistic photos, pets, unique experiences penalized; <10 few-shot examples for tone; deterministic thresholds in config; OpenAI prompt caching ~93% hit rate. Shipped globally: ~30% of users who saw it added/replaced a photo; engagement up low single digits.

##### On-device AI Photo Selector [P] — Thomas Yoon
https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos
- iOS pipeline: selfie or "BasePhotoInference" fallback → download & **decrypt (CryptoKit)** TFLite models + third-party SDK model → per asset (OperationQueue, max 8 concurrent): Apple Vision face detect (exactly one face; VNDetectFaceRectanglesRequest, landmarks rev 3) → 112×112 aligned crop → **FaceMeSDK** similarity vs base embedding (threshold) → **Tinder TFLite model outputs "probability of receiving likes"** → sort → **TFLite moderation model on top 100** (underage, violence, text-heavy, etc.).
- `isNetworkAccessAllowed = false` (no iCloud fetch). Analytics described as counts only (faces detected/identified). "without ever leaving the device". Benchmarked 1,000 photos. Base-photo fallback "monumental funnel drop improvement".
- Help center: processing on device; Tinder only collects photos you choose; biometric data deleted on exit. https://www.help.tinder.com/hc/en-us/articles/21276850679693-Photo-Selector

##### Chemistry (2025–26) [S]
- Announced with Match Q3 2025 results (Nov 2025): AI learns via questions and, with permission, Camera Roll; pilots NZ & Australia; "major pillar" of 2026 product (Spencer Rascoff); ~$14M Q4 revenue cost of testing. Feb 2026: still testing in Australia; "a single drop or two" instead of swiping; AI reordering of profiles shown to women credited with small gains; Face Check cut bad-actor interactions >50%. Mar 2026 (Axios snippet): live in AU/NZ, "camera roll scan" → "photo insights". No confirmed US launch as of Nov 2026.
- https://techcrunch.com/2025/11/05/tinder-to-use-ai-to-get-to-know-users-tap-into-their-camera-roll-photos/ ; https://techcrunch.com/2026/02/04/tinder-looks-to-ai-to-help-fight-swipe-fatigue-and-dating-app-burnout/ ; https://www.axios.com/2026/03/12/tinder-ai-features-hinge-bumble

#### Gaps
- Medium-hosted posts (geosharding, TAG) and help.tinder.com could not be fetched; details are from secondary summaries.
- No first-party description of what fully replaced Elo, nor of Keepalive WebSocket, Redis/DynamoDB usage in this session.

---

## Part D — Research notes: patents, academia, journalism

### Patents, academic research, regulation and journalism — notes

[P] = confirmed from primary source (patent page, paper, official page); [S] = secondary coverage or search snippet. Help Center, Justia, Axios and Guardian blocked direct fetching.

#### A) Patents (Tinder Inc. / Match Group)
1. **US 9,733,811 B2 "Matching process system and method"** [P] — filed 2013-10-21, granted 2017-08-15, priority 2008-12-19 (US 12/339,301 → US 8,566,327). Inventors Sean Rad, Todd Carrico, Kenneth Hoskins, James Stone, Jonathan Badeen; assignee Tinder Inc. Claims: one-at-a-time profile cards; swipe gestures for interest/no interest; next card auto-shown; messaging only after mutual interest, one-sided interest hidden; distance threshold. Asserted vs Bumble 2018 [S]. https://patents.google.com/patent/US9733811B2 ; https://ipwatchdog.com/2018/03/27/match-group-sues-bumble-former-tinder-employees-misappropriated-trade-secrets/
2. **Family continuations** [P family list]: US 8,566,327; 9,959,023; 10,203,854; 11,513,666; 11,733,841; US 2024/0393929 — variants of swipe/mutual-match flow (claims not individually read).
3. **US 12,105,941 B2 (same title)** [P] — filed 2022-11-23, granted 2024-10-01, Match Group LLC. Claim 1: each candidate gets an **attractiveness score = ratio between times profile viewed and times others expressed positive preference**; feeds a **similarity score** to the viewer; stack ordered by similarity; mutual interest unlocks messaging/voice. Shared spec: averaged 1–9 attractiveness ratings; similar-rated users shown to each other; attractiveness "imputed" from appearances in results and click-throughs. https://patents.google.com/patent/US12105941B2
4. **US 11,989,220 B2 (US 2020/0175047 A1) "System for determining and optimizing for relevance in match-making systems"** [P] — granted 2024-05-21, priority 2010; inventors Fernando Diaz, Donald Metzler, Sihem Amer-Yahia; Match Group → Tinder LLC. **Reciprocal relevance**: infer relevance from implicit two-way signals (one- vs two-way messages and views, imbalances, timing/density) → train **gradient-boosted decision tree** ranker usable for candidates without interaction data. https://patents.google.com/patent/US20200175047A1
5. **US 11,151,208 B2 "System and method for recommending users based on shared digital experiences"** [P] — filed 2019, granted 2021-10-19; related US 11,755,670, 12,585,717. Branching interactive-media choices used to find compatible users (likely Swipe Night — inference). https://patents.google.com/patent/US11151208B2
6. **US 8,559,706 B2 "Fraud detection using image analysis"** [P] — granted 2013; colour-based image representation compared to known users' images → fraud flag. https://patents.google.com/patent/US8559706B2
7. **US 9,418,057 / 8,862,461 "Fraud detection using text analysis"** [S] — Match.com 2011 family. https://patents.google.com/patent/US9418057B2/en
8. **US 9,158,821 B1 "optimizing interactions between users"** [S] — ranks matches by a "messaging score" relative to the user's own. https://patents.google.com/patent/US9158821
9. **US 12,273,850 B2 "matching using location information"** [S] — Match Group Americas, 2023 filing; claims not verified. https://patents.google.com/patent/US12273850B2/en
- Not found: granted patents for Super Like, Boost, Smart Photos. Claim that US 11,720,613 is a Match attractiveness patent is false (it's a Google patent).

#### B) Academic research
- **Bruch & Newman (2018), Science Advances 4(8): eaap9815** [P] — PageRank-style desirability on messaging networks in 4 US cities; people message partners ~25% more desirable than themselves (men ~26%, women ~23%); reply probability falls with gap; women's desirability declines with age, men's peaks ~50; Asian women and white men ranked highest. Confirms steep desirability hierarchy. https://pmc.ncbi.nlm.nih.gov/articles/PMC6082652
- **Hutson, Taft, Barocas & Levy (2018) "Debiasing Desire", Proc. ACM HCI 2(CSCW) Art. 73** [P abstract] — platform design (race filters, sorting, same-race defaults) can entrench sexual racism; design interventions can reduce bias. https://arxiv.org/abs/1809.01563
- **Tyson, Perta, Haddadi & Seto (2016) "A First Look at User Activity on Tinder", ASONAM** [P abstract] — women rapidly get many matches, men slowly; selectivity asymmetry → feedback loop into extreme strategies (press: men ~0.6% match rate; 21% of female vs 7% of male matches message [S]). https://arxiv.org/abs/1607.01952 ; https://www.technologyreview.com/2016/07/15/158803/how-tinder-feedback-loop-forces-men-and-women-into-extreme-strategies/amp/
- **Reciprocal recommenders**: Pizzato et al., RECON, RecSys 2010 (foundational); Palomares et al. 2021 survey, Information Fusion 69:103–127. https://arxiv.org/abs/2007.16120
- **Pidoux (EPFL PhD 2021); Pidoux, Kuntz & Gatica-Perez, CSCW 2021** [P summary] — from Tinder patent analysis: scores for attractiveness, IQ, similarity; ordering weighted by age difference, attractiveness ratio, education; "patriarchal" default pairing; binary gender assumptions. https://aiequalitytoolbox.github.io/social-impact-tinder-pidoux.html ; https://infoscience.epfl.ch/record/288400

#### C) Journalism & data access
- **Fast Company, Austin Carr (Jan 2016)** [S] — Tinder confirmed internal Elo-style "desirability" score; Sean Rad: "It's not just how many people swipe right on you"; Carr's score 946. https://www.refinery29.com/en-us/2016/01/101011/tinder-secret-score ; https://www.digitaltrends.com/phones/tinder-rank-elo-score/
- **Judith Duportail, Guardian (26 Sep 2017)** [S] — GDPR request (with Paul-Olivier Dehaye, Ravi Naik) returned ~800 pages: Facebook likes, Instagram photos (even after deletion), education, logins, messages; **no Elo score disclosed**; Tinder called matching tools core IP. https://flowingdata.com/2017/12/01/800-pages-of-tinder-data/
- **Duportail, "L'Amour sous algorithme" (2019)** — argues attractiveness/desirability scoring persists, based on patents. https://www.journaldugeek.com/dossier/tinder-matchs-algos-judith-duportail-devoile-secrets-de-lappli-de-dating/
- **Irish DPC inquiry (2020)** [S] — into Tinder processing and access-request handling. https://www.irishtimes.com/news/ireland/irish-news/no-obligation-to-release-entire-chat-logs-tinder-tells-dpc-1.4161901

#### D) Pricing & regulation
- **Consumers International / Mozilla study (Feb 2022)** [S] — six countries (NZ, US, NL, KR, IN, BR); 30–49 y/o charged on average 65.3% more than 18–29 for Tinder Plus; US +42.4%; NL >2×; no significant evidence for gender/orientation. Tinder: age pricing ended in US/UK/BR/AU, elsewhere by end Q2 2022. https://www.consumersinternational.org/news-resources/news/releases/new-research-consumers-worldwide-concerned-by-opaque-online-pricing/ ; https://www.buzzfeednews.com/article/emilybakerwhite/tinder-plus-pricing-study
- **California age-pricing lawsuit (Candelore v. Tinder)** — ~$24M settlement [S]. https://www.consumerreports.org/consumer-protection/tinder-is-phasing-out-higher-prices-for-users-ages-29-plus-a3881901881

#### Bottom line
Primary sources establish: swipe mutual-opt-in matching; patented ranking combining an attractiveness ratio (positive preferences vs views) with similarity; reciprocal implicit-signal relevance model (GBDT); swipe-derived embeddings (TinVec); bandit-trained pairwise photo VLM; image-based fraud detection. Unverified: which patented mechanisms run in production today; what fully replaced Elo.

---

## Part E — Research notes: reverse-engineering claims

### Independent Reverse-Engineering Claims About Tinder Internals (Ayo O video, Oct 2026) and Their Credibility

Scope note: literature and web research only, current as of 16 Nov 2026. No APK, model or Tinder service was accessed. Evidence labels used below:
- **VERIFIED**: confirmed by the coordinator's own check of tinder.com client code or the public tinder-protos schema, or by Tinder's own published material.
- **CLAIMED**: rests only on Ayo's video or secondhand summaries of it.
- **CONTRADICTED / IN TENSION**: conflicts with Tinder's published statements or other evidence.

#### 1. Where is the original video, how was it made, and how was it received?

##### Takeaway
The original is a 79-second vertical video titled "I reverse engineered Tinder," posted on X on 20 Oct 2026 by Ayo O (@ayoships / @TheDonAyo, co-founder of a0.dev, YC W25). It got about 2.4M views. No first-party write-up or repo from Ayo was found. The method shown is decompiling a 2022 Android build, reading iOS strings, and running the bundled TFLite model offline on more than 800 AI-generated images. As of mid-Nov 2026, no Tinder/Match Group response and no mainstream tech-press coverage were found. The video sits in a serial "I reverse engineered X" format (TikTok, Block Blast, Royal Kingdom), with at least one entry explicitly AI-assisted.

##### Cited Findings
- Tweet text is just "I reverse engineered Tinder" with one video (720x1280, ~78.8 s). Posted Tue 20 Oct 2026 17:08 UTC. Stats at fetch time: about 2.37M views, 27.0k likes, 1.8k reposts, 295 replies, 248 quotes, 17.7k bookmarks. — [fxtwitter mirror of X post](https://api.fxtwitter.com/TheDonAyo/status/2112591782857109809); [X post](https://x.com/ayoships/status/2112591782857109809). The same video is cross-posted as a YouTube Short. — [YouTube Short](https://www.youtube.com/shorts/Qziwio_mo0s)
- View counts differ by source: hegetswipes cites about 1.5M (as of 21 Oct), while the X API shows about 2.37M later. The difference is consistent with growth over time. — [hegetswipes write-up](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- Ayo is identified as Ayomide Omolewa, co-founder of a0.dev (YC W25). — [LinkedIn profile (search snippet; page not fetchable)](https://www.linkedin.com/in/omolewa/)
- **Method (as summarised by hegetswipes, 21 Oct 2026):**
  - He shows decompiled Android code from version 13.0.0 (2022) and iOS app strings.
  - He says he extracted a TensorFlow Lite model from the app.
  - He ran more than 800 synthetic photos through it, "changing one variable at a time."
  - Source: [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- **AI assistance:** Ayo's earlier post in the same series reads "i used AI to reverse engineer the block blaster mobile app." He has since posted "I reverse engineered tiktok" and "I reverse engineered Royal Kingdom." — [X: Block Blast](https://x.com/ayoships/status/2110461702936695263); [X: TikTok](https://x.com/ayoships/status/2117650684774580249); [X: Royal Kingdom](https://x.com/ayoships/status/2120696957320749453)
  - No source found says outright that the Tinder analysis was AI-assisted. Given the series pattern, it is plausible but unconfirmed.
- **Repo:** A third-party X post claims "This guy reverse-engineered Tinder in just one day" and links a repo through a t.co shortlink. The X page returned HTTP 402, so the repo could not be identified. — [X: @imryanarya007](https://x.com/imryanarya007/status/2112937122848276564)
  - hegetswipes lists no repo and cites no file named fm.bin. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- **Caveats hegetswipes lists itself:**
  - The 2022 code may not match what runs today.
  - The privacy claim rests on a single payload snippet that shows field names only.
  - The results come from synthetic photos in his own setup, not the live app.
  - Some rows rest on as few as 8 photos.
  - Tinder has not confirmed any of it.
  - Source: [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- A LinkedIn post (snippet only) attributed to Omolewa concedes that reverse engineering an app does not reveal the code running on Tinder's servers. It also says the findings lack independent replication. — [LinkedIn (snippet)](https://www.linkedin.com/in/omolewa/)
- **No company response:** A targeted search for a Match Group or Tinder statement on the rsrr/Photo Selector claims found none. — [search summary; hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- **Unrelated regulatory news the same month:** On 13 Nov 2026, Ireland's Data Protection Commission (DPC) fined Match Group €11M. The fine concerns Tinder's "Download My Data" tool, which did not provide all stored data. Match says it will appeal. The article does not mention ranking or desirability scores. — [Extra.ie, 13 Nov 2026](https://extra.ie/2026/11/13/news/tinder-handed-major-fine)
- **Possible misattribution in secondary sources:** A search summary suggested that a darker-skin/older-face bias result came from Ayo's TikTok post rather than the Tinder one. However, the hegetswipes Tinder write-up explicitly lists "Darker skin: −15%, down in all 12 scenes." The Tinder attribution therefore stands, but readers should not merge the two experiments. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered); [X: TikTok](https://x.com/ayoships/status/2117650684774580249)

##### Inferences
- The primary artifact is a short video, not a reproducible write-up. Without released code, image sets, prompts or model hashes, nobody can replicate the synthetic experiment exactly.
- Major outlets (The Verge, 404 Media, TechCrunch, Wired) appear not to have covered it. Searches surfaced only blog and SEO-style write-ups, so the claims have not been through journalistic fact-checking or a company request for comment.
- The video belongs to a high-volume content series, which favours speed and virality over rigour. That is not disqualifying, but it raises the bar for the strongest claims, especially bias magnitudes.

##### Gaps
- The video transcript and on-screen screenshots could not be fetched directly. The claim list comes from the coordinator's brief plus the hegetswipes summary.
- The repo behind the t.co link in @imryanarya007's post is unknown, as is whether it is Ayo's or a copycat.
- Reddit and Hacker News threads on the video were not found by search.
- No statement from Tinder or Match Group was found. The ~295 replies to the tweet were not readable.

#### 2. Claim-by-claim evaluation: verified vs claimed vs contradicted

##### Takeaway
The telemetry and monetisation claims are well corroborated by tinder.com client code and the public tinder-protos schema: view-duration telemetry, the high-RSRR Super Like upsell tagged "revenue", and per-photo rsrr/explicit/underage/similarity score fields in the Photo Selector schema. The on-device models are partly corroborated by Tinder's own engineering blog, which confirms an in-house TFLite "likes" scoring model plus a moderation model. The specific model files, their outputs (rsrrFemale/rsrrMale) and the "type averaging" mechanism remain claims. The clearest tension is that the score schema exists while Tinder says Photo Selector is fully on-device with counts-only analytics. Schema existence alone does not prove the fields are populated and transmitted.

##### Cited Findings

**A. "Your type" = average of embeddings of right-swiped photos (`liked_swipees_image_average`)** — CLAIMED
- hegetswipes summarises this as a "type" score built by averaging embeddings of right-swiped photos. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- No independent source or Tinder publication found describes this. A field name alone would not show how, or whether, it is used in server-side ranking.

**B. Photo view timing (`media_view_details_card` {photo_id, view_count, total_duration_ms})** — VERIFIED (telemetry exists)
- hegetswipes reports this event. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- Coordinator's verification: tinder.com swipe events send `duration_in_millis` and media view counts (verified in tinder.com client code).
- Collecting dwell time is verified. Whether dwell time feeds ranking or "type" learning is not.

**C. RSRR = "right swipe received ratio"** — VERIFIED (term and usage), meaning plausible
- hegetswipes defines RSRR as the share of viewers who swipe right, citing Android 13.0.0 (2022). — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- Coordinator's verification: the rsrr naming appears in the tinder.com upsell code (`upsell_superlikes_high_rsrr`, flag `superlike.upsell_high_rsrr_enabled`) and in the tinder-protos `rsrr_scores` field (verified in tinder.com client code / tinder-protos schema).
- A general web search found no other public source defining RSRR. Third-party "algorithm" blogs discuss swipe ratios speculatively. — [SwipeStats](https://www.swipestats.io/blog/tinder-algorithm)
- Tinder publicly says it moved away from Elo in 2019 toward engagement-based models, but it has not named RSRR. — [SwipeStats summary of Tinder's 2019 statement](https://www.swipestats.io/blog/tinder-algorithm)

**D. Bundled TFLite models fm.bin (~9.5 MB) and mn.bin (~2.5 MB) with outputs rsrrFemale, rsrrMale, nudity, underage, face, fullBody** — PARTLY CORROBORATED
- Tinder's engineering blog (author Thomas Yoon) says Photo Selector uses in-house TensorFlow Lite models:
  - one scores photos for likelihood of likes;
  - a separate moderation model checks for things like underage individuals, violent content and text-heavy images;
  - the facial similarity SDK is FaceMeSDK;
  - Apple Vision handles face detection and 112×112 normalised crops;
  - only the top 100 candidates are moderated.
  - Source: [Tinder Engineering blog](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)
- File names, sizes, the "TFL3" header and gender-split outputs (rsrrFemale/rsrrMale) are CLAIMED only. hegetswipes does not mention fm.bin. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- A "TFL3" magic header is the standard TFLite FlatBuffer identifier, so that detail is internally consistent but not independently checked here.
- Inference: a "likes-probability" model, which is what Tinder describes, and an "RSRR predictor" are functionally the same idea, so the claim is directionally consistent with Tinder's own description.

**E. Photo Selector defaults (20,000 photos, 2 years, face similarity 0.7, 27 results) and the `PhotoSelectorPhotosCollectionDetails` payload to POST /v3/publish/app/binary** — MIXED
- The numeric defaults are CLAIMED.
  - Tinder's engineering blog gives no numeric face threshold. It mentions only a "defined threshold," benchmarking with 1,000 photos, and moderation of the top 100. — [Tinder Engineering blog](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)
  - hegetswipes reports 20,000 / 2 years / 0.7 / 27. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- The score-field schema is VERIFIED: tinder-protos' `photo_selector_photos_collection_details` contains `rsrr_scores`, `explicit_scores`, `text_scores`, `underage_scores` and `similarity_scores`. Moderation schema counts include `less_clothed_flagged` and `mirror_selfie_flagged` (verified in tinder-protos schema).
- tinder-protos describes itself as "a collection of all protobuf files scraped from tinder.com." It has 3 commits and 5 stars, does not state an extraction method or date, and includes an "educational purposes only" disclaimer. — [GitHub: Davis-3450/tinder-protos](https://github.com/Davis-3450/tinder-protos)
- **IN TENSION:**
  - Tinder's engineering blog says analytics capture "the total number of photos containing faces" and how many matched, "without exposing private or confidential user metadata." It also says processing happens on the device. — [Tinder Engineering blog](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)
  - A Tinder spokesperson told Mashable the company collects only "limited analytics data," such as time taken and number of photos recommended. — [as reported in search results summarising Mashable; see 9to5mac / candid.technology coverage](https://candid.technology/tinder-ai-powered-profile-photo-selector-tool/)
  - The help centre says the feature "operates entirely on your device." — [Tinder Help: Photo Selector (403 on fetch; quoted via search)](https://www.help.tinder.com/hc/en-us/articles/21276850679693-Photo-Selector)
  - The existence of per-photo score arrays in a client-to-server schema conflicts with a counts-only description. However, hegetswipes concedes the evidence is one payload snippet with field names only. Populated values in live traffic have not been shown publicly. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- One aggregator describes a separate camera-roll scan feature that processes images temporarily on servers, with 90-day retention. This is unverified and may concern a different, newer feature. — [letsdatascience](https://letsdatascience.com/news/tinder-launches-camera-roll-scan-feature-49658a59); [PetaPixel, Nov 2025](https://petapixel.com/2025/11/10/tinder-wants-to-see-your-photos-to-find-better-matches/)

**F. High-RSRR Super Like upsell** — VERIFIED
- hegetswipes cites the iOS string "Send a Super Like to %@ instead for a better chance to match!" and the event `upsell_superlikes_high_rsrr` tagged as revenue. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
- Coordinator's verification on tinder.com:
  - `SuperlikeUpsellModal` named "upsell_superlikes_high_rsrr", with sourceSessionEvent "revenue";
  - flag `superlike.upsell_high_rsrr_enabled`;
  - recs objects carry `is_superlike_upsell`.
  - (verified in tinder.com client code)
- Context: hegetswipes cites 8.5M monthly payers from Match Group's Q2 2026 shareholder letter as the revenue motive. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)

**G. Smart Photos / photo-scoring lineage (context)**
- In 2016, Tinder publicly launched Smart Photos, which reorders a user's photos so the "best" one shows first. — [Tinder blog (2016)](https://blog.gotinder.com/introducing-smart-photos-for-the-most-swipeworthy-you); [Fortune 2016](https://fortune.com/2016/10/13/tinder-smart-photos/)
- A 2026 upgrade reportedly uses vision-language models to judge photo "vibe." Reported accuracy rose from 68% to 76%. Tinder has not said whether it tested the model for disparate impact. — [Dating Industry Insights (Apr/Jul 2026)](https://www.datingindustryinsights.com/news/tinder-ai-photo-ranking-system-explained)
- Inference: Tinder openly does per-photo engagement scoring. The novelty of Ayo's claim is the specific label (RSRR), the on-device placement, the gender-split heads and the transmission of scores, not the existence of photo scoring.

##### Inferences
- The strongest claims are the ones that need only static artifacts: field names, strings, flags. Several are now independently matched on tinder.com.
- The weakest claims are about behaviour: how ranking uses these signals, whether scores are actually uploaded, and whether "type averaging" drives recommendations. Static analysis of a 2022 client cannot show server-side use.
- The schema tension is real and reportable. It should be framed as "schema supports transmitting per-photo scores; Tinder says only counts are collected; live traffic not publicly demonstrated."

##### Gaps
- Live-traffic confirmation of populated `rsrr_scores` in PhotoSelector uploads has not been publicly shown.
- No source independently confirms the fm.bin/mn.bin file names or sizes, or the rsrrFemale/rsrrMale outputs.
- The tinder-protos repo does not state its extraction date, so it cannot be tied to a specific client version.
- No Tinder or Match Group comment on the RSRR upsell.

#### 3. Other independent reverse-engineering work on Tinder (architectural level only)

##### Takeaway
Tinder has a decade-long history of credible independent security research that Tinder acknowledged and fixed:
- location trilateration (Include Security, 2013–14; later Robert Heaton);
- Checkmarx's 2018 finding of unencrypted image traffic and swipe size-inference, corroborated by Sophos.

There is also a hobbyist ecosystem of scraped protobufs and unofficial API clients. Ayo's work differs because it targets ML models and ranking signals rather than security flaws, and it has not been through a disclosure process.

##### Cited Findings
- **Location trilateration:**
  - Max Veytsman of Include Security found that Tinder's API returned precise distances, which let a user's location be re-derived. Discovery is dated 2013, with a fix before Jan 2014. — [PortSwigger](https://portswigger.net/daily-swig/trilateration-vulnerability-in-dating-app-bumble-leaked-users-exact-location); [Business Insider India](https://www.businessinsider.in/tinder-had-a-security-flaw-for-several-months-that-let-users-see-each-others-exact-location-according-to-a-report/articleshow/30691547.cms)
  - Robert Heaton later described how Tinder's server-side distance fix was still triangulable before proper rounding. — [mjtsai summary of Heaton (2018)](https://mjtsai.com/blog/2018/07/19/how-tinder-keeps-your-exact-location-a-bit-private)
  - He applied the same technique to Bumble in 2021 and received a $2,000 bounty. — [The Register](https://www.theregister.com/2021/08/26/bumble_location_leak/); [Heaton](https://robertheaton.com/bumble-vulnerability)
  - Synack's related research is dated 2015, not 2014. — [PortSwigger](https://portswigger.net/daily-swig/trilateration-vulnerability-in-dating-app-bumble-leaked-users-exact-location)
- **Checkmarx (Jan 23, 2018):**
  - Tinder's iOS and Android apps fetched profile images over plain HTTP, while the web app used HTTPS.
  - Swipe direction could be inferred from encrypted packet sizes: left = 278 bytes, right = 341 bytes. Checkmarx demonstrated this with its "TinderDrift" tool.
  - Tinder was notified in Nov 2017. It said it was working to encrypt images in the app.
  - Sophos independently corroborated the findings.
  - Sources: [Sophos](https://news.sophos.com/en-us/2018/01/24/tinder-user-lack-of-encryption-means-stalkers-can-watch-you-at-it/?amp=1); [Fortune](https://fortune.com/2018/01/23/researchers-say-tinders-limited-encryption-makes-it-ripe-for-hackers/); [Engadget](https://www.engadget.com/2018-01-23-tinder-flaws-allow-attacker-to-monitor-activity.html); [Checkmarx](https://checkmarx.com/in-the-news/tinders-lack-encryption-lets-strangers-spy-swipes)
- **Protobuf schemas:** tinder-protos, described as "scraped from tinder.com." — [GitHub](https://github.com/Davis-3450/tinder-protos)
- **Unofficial API and bot repos** (e.g., u-Aymane/Tinder-Reverse-Engineering, "Tinder Reverse Enginneering and AI Bot") show that hobbyist API reverse engineering is common. Per the brief, no endpoint or auth details are compiled here. — [GitHub](https://github.com/u-Aymane/Tinder-Reverse-Engineering)
- **Earlier algorithm disclosures:**
  - BuzzFeed News's "Cracking The Tinder Code" is cited by critics as an instance where Tinder showed a journalist under the hood. Treat it as a lead; it was not fetched. — [BuzzFeed News](https://www.buzzfeednews.com/article/alexkantrowitz/cracking-the-tinder-code-love-in-the-age-of-algorithms)
  - Tinder's 2016 Smart Photos announcement. — [Tinder blog](https://blog.gotinder.com/introducing-smart-photos-for-the-most-swipeworthy-you)
- **Data-driven outside analyses** (not reverse engineering): SwipeStats analyses user data exports. It warns that it cannot see Tinder's impression inventory or recommendation model. — [SwipeStats](https://www.swipestats.io/blog/tinder-algorithm)

##### Inferences
- Prior Tinder research that earned credibility shared three traits: reproducible demonstrations, vendor disclosure, and independent corroboration (e.g., Sophos re-testing Checkmarx). Ayo's work currently has partial corroboration of static artifacts but none of the three for the bias experiment.
- The Checkmarx packet-size finding is a precedent showing swipe behaviour is observable telemetry. That fits the newer finding that dwell time and view counts are logged.

##### Gaps
- No academic paper doing static or dynamic analysis of Tinder's ML components was found.
- No independent replication of Ayo's model-probing experiment was found.

#### 4. Methodological critique of the synthetic-photo bias experiment

##### Takeaway
The synthetic experiment can at most show what one 2022-era on-device scoring model outputs for out-of-distribution AI images. It cannot show how Tinder ranks real people today. Specific weaknesses:
- distribution shift from generated images;
- confounded edits, especially skin tone with exposure and contrast;
- tiny per-condition samples (some n≈8–16) with no reported variance;
- many uncorrected comparisons;
- unknown mapping from model outputs to server ranking;
- model age.

Even so, the direction of several findings matches a well-documented literature on demographic bias in attractiveness and saliency models. The results are plausible hypotheses, not established measurements.

##### Cited Findings
- **The reported results (Ayo's numbers via hegetswipes):**
  - Mirror selfie: about ×3.5.
  - Men: outdoors +49%, shirtless +23%, sunglasses +15%. Heavier or 15 years older: no consistent change.
  - Women: low neckline +63%, sunglasses −26%, 15 years older −16%, heavier −54%. Outdoors: no consistent change.
  - Darker skin: −15%, "down in all 12 scenes."
  - Some rows rest on as few as 8 photos.
  - Source: [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered)
  - The brief's figures differ slightly: men outdoors ~+50% vs +49%; women sunglasses −24%/−26%.
- **Sample size and multiple comparisons:**
  - With n=12 and all 12 down, a two-sided sign test gives p≈0.0005. That is notable for a single test, but it is one of roughly 13 or more contrasts reported.
  - The 16/16 mirror-selfie result is similarly strong as a sign test.
  - Conditions reported as "no change" may simply be underpowered.
  - No confidence intervals, variance or pre-registration were reported. — [hegetswipes caveats](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered) (sign-test arithmetic is the author's own inference)
- **Confounds in edits:**
  - Saliency and vision models are sensitive to contrast and background. Twitter's own audit attributed part of its cropping disparity to darker backgrounds and contrast effects. — [Yee et al., arXiv 2105.08667](https://arxiv.org/abs/2105.08667v2); [Twitter Eng blog](https://blog.x.com/engineering/en_us/topics/insights/2021/sharing-learnings-about-our-image-cropping-algorithm)
  - Skin-tone edits in generators commonly shift global luminance and face-region exposure, which could drive part of a −15% effect. (Inference; not tested in the source.)
  - Tinder's pipeline crops faces to 112×112 normalised references for similarity. If the RSRR model uses similar crops, background edits (outdoors) may matter differently than face edits. — [Tinder Engineering blog](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)
- **Distribution shift:**
  - Synthetic images have generator-specific textures, lighting and idealised faces. A model trained on real swipes may respond to these artifacts, which distorts the effect of each edit.
  - Moderation heads (nudity, less-clothed) may interact with the shirtless and low-neckline edits. The schema itself counts `less_clothed_flagged` and `mirror_selfie_flagged`, so in production those photos might be filtered or flagged rather than promoted.
  - Source: verified in tinder-protos schema. The interpretation is inference.
- **Model vs system:**
  - Tinder says the likes model only ranks camera-roll candidates in Photo Selector, after face matching and moderation. — [Tinder Engineering blog](https://lifeattinder.com/blog/how-on-device-ai-models-find-your-best-tinder-profile-photos)
  - A Photo Selector scoring model is not the server-side recommendation ranker. Ayo himself reportedly acknowledged that client reverse engineering cannot reveal server code. — [LinkedIn snippet](https://www.linkedin.com/in/omolewa/)
- **Model age:** The decompiled code is Android 13.0.0 (2022). The models may have been retrained, and Smart Photos moved to vision-language models in 2026. — [hegetswipes](https://hegetswipes.com/blog/tinder-algorithm-reverse-engineered); [Dating Industry Insights](https://www.datingindustryinsights.com/news/tinder-ai-photo-ranking-system-explained)
- **Label semantics:**
  - If the target is RSRR, the model learns aggregate swiper preferences, which mirror human biases rather than "objective attractiveness."
  - Gender-split heads (rsrrFemale/rsrrMale, claimed) would mean predicting how a given audience swipes. That makes demographic asymmetries in outputs expected rather than surprising. (Inference.)
- **Supporting literature on bias:**
  - A 2025 arXiv study fine-tuned ResNet-152 beauty regressors. On MEBeauty, mean predicted scores were 0.65–0.66 for Caucasian and Middle Eastern faces vs 0.56 for Asian and Black faces. Kruskal–Wallis tests rejected equal distributions. — [arXiv 2509.24138](https://arxiv.org/html/2509.24138v1)
  - Debiasing approaches exist (e.g., FairViT-GAN adversarial debiasing), which implies bias in baseline models. — [arXiv 2509.23859](https://arxiv.org/html/2509.23859v1)
  - A 2026 JMIR review warns that attractiveness-prediction AI risks perpetuating "homogenizing" bias. — [JMIR 2026](https://www.jmir.org/2026/1/e95452)
  - Twitter's saliency audit found a 4% white-over-Black preference and an 8% gap in favour of women, and Twitter removed the feature. — [Yee et al.](https://arxiv.org/abs/2105.08667v2); [PetaPixel](https://petapixel.com/2021/05/22/twitter-axing-ai-photo-cropping-after-tests-reveal-race-gender-bias)
  - Psychology research suggests facial structure may matter more than skin tone in perceived racial prototypicality, so isolating skin tone is hard. — [PMC3399873](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3399873/)
  - Tinder has not disclosed debiasing or disparate-impact testing for its photo models. — [Dating Industry Insights](https://www.datingindustryinsights.com/news/tinder-ai-photo-ranking-system-explained)

##### Inferences
- **Most robust result:** darker-skin decreases (12/12 direction-consistent) and the mirror-selfie increase (16/16). Even here, the cause could be the edit artifact rather than skin tone, and the effect size depends on undisclosed image construction.
- **Least robust:** gendered effects for sunglasses, outdoors, age and weight. These are small-n, inconsistently reported (−24 vs −26%), and some are "no consistent change" cells where the test may lack the power to detect an effect.
- The mirror-selfie ×3.5 result clashes with the existence of a `mirror_selfie_flagged` moderation count. A model that scores mirror selfies highly may coexist with downstream rules that discourage them. This is worth flagging as an unexplained interaction.
- **A credible replication would need:**
  - real photos with controlled re-photography rather than generative edits;
  - luminance-matched skin-tone manipulations;
  - per-condition n ≥ 30 with confidence intervals;
  - multiple-comparison correction;
  - current model versions;
  - a clear statement that model outputs are not the production ranking.

##### Gaps
- The image generator, prompts, edit procedure and scoring method (which output head; per-gender heads) are undisclosed, which blocks a precise critique.
- No study was found that specifically audits dating-app photo-scoring models for skin-tone bias.
- It is unknown whether the 2022 model is still shipped in current builds.

---

## Appendix 1 — The ayoships video: transcript and on-screen claims

Source: Instagram reel / X video by ayoships (@TheDonAyo), October 2026, as supplied by the user in this session (transcript and 25 screenshots). Recorded here as the user provided it; claims are the creator's, not verified facts.

**Transcript (as supplied by the user)**

> I reverse-engineered Tinder to figure out more about their algorithm and how they get 8.5 million people to pay every month. I discovered that Tinder's calculating two things, what you want and how much you're wanted. Tinder learns your type by turning every photo you swipe right on into an embedding and averaging them. It even times how long you stare at each photo. Tinder also tracks how often people swipe right on you. They call it RSRR, your right swipe received ratio, which is basically out of everyone who sees you, how many actually want you. Tinder even trained a model that runs on your phone and can predict that number for every one of your photos. Their photo selector feature scans up to 20,000 photos in the last two years, and they run this model on every photo of you and pick the best 27. Tinder claims everything stays on device, but they still collect each photo's RSRR score, plus how explicit it is and how young you look in it. So I pulled the model out and ran it against my own synthetic dataset to get a window into their algorithm. I discovered that across the board, mirror selfies triple your score, and more revealing clothes pushes it even higher for both men and women. However, the darker your skin, the lower your score. For men specifically, sunglasses and outdoor photos help while doing nothing or worse for women. For women, being heavier and older does hurt your score significantly, while neither does anything to the men's score. Then Tinder uses this RSRR score to sell you. If Tinder decides your profile is popular, some of the people who swipe right on you will enter a paywall flow, with the sale literally tagged as high RSRR upsell, so Tinder can measure exactly what your popularity earns. Let me know in the comments what content you'd like to see next.

**On-screen claims (from the screenshots)**

| # | Screen | Claim shown |
|---|---|---|
| 1 | "liked_swipees_image_average" | Embeddings of liked profiles' photos averaged into "your type" |
| 2 | `// every swipe: RecsStatusDetails` | `media_view_details_card [ { photo_id, view_count, total_duration_ms } ]` |
| 3 | `// Tinder Android 13.0.0 (2022)` | `RsrrField.description()` → "right swipe received ratio, could be null" |
| 4 | `unzip -l com.tinder.apk \| grep models` | `assets/models/fm.bin` (9,509,517 B), `assets/models/mn.bin` (2,538,056 B); header `TFL3`; outputs `rsrrFemale`, `rsrrMale`, `nudity`, `underage`, `face`, `fullBody` |
| 5 | `// Photo Selector defaults` | `max_num_of_photos_to_scan = 20000`, `max_creation_date_in_years = 2`, `face_similarity_threshold = 0.7`, `final_result_num_photos = 27` |
| 6 | `// PhotoSelectorPhotosCollectionDetails` | `total_number, rsrr_scores, explicit_scores, text_scores, underage_scores, similarity_scores` sent in an analytics publish call |
| 7 | "800+ synthetic photos" | Synthetic dataset run through the extracted model |
| 8 | Mirror selfies | ×3.5 typical change, 16 of 16 went up (individual: ×4.0, ×15.6, ×12.0, +34%, +35%, ×8.9, +38%, ×8.4, ×7.5, +64%, +85%, ×2.1, ×2.2, ×5.0, ×4.5, ×3.0) |
| 9 | Men · shirtless | +23% score, 25 of 25 went up |
| 10 | Women · low neckline | +63% score, 12 of 15 went up |
| 11 | Darker skin | −15% score, 12 of 12 scenes went down |
| 12 | Men · sunglasses | +15% score |
| 13 | Men · outdoors | ~+50% score |
| 14 | Women · sunglasses | −24% / −26% score, went down |
| 15 | Women · outdoors | No change (4 up, 4 down) |
| 16 | Women · heavier | −54% score, of 38 went down |
| 17 | Women · 15 yrs older | −16% score, of 38 went down |
| 18 | Men · heavier | No change (… up, 15 down) |
| 19 | Men · 15 yrs older | No change |
| 20 | `// Tinder iOS 17.39.2 strings` | "You just liked a popular user. Send a Super Like to %@ instead for a better chance to match!"; paywall source `super_like.high_rsrr_upsell` |
| 21 | `// tinder.com SuperlikeUpsellModal.js` | `location: "discovery", name: "upsell_superlikes_high_rsrr", sourceSessionEvent: "revenue", type: "modal"`; recs payload `is_superlike_upsell: true` |

**Verdict summary (details in Parts A and E)**

| Claim | Status |
|---|---|
| Upsell modal name / "revenue" tag / lever / `is_superlike_upsell` | Verified in tinder.com client code |
| Dwell-time and media-view telemetry per swipe | Verified (`duration_in_millis`, `media_views_card`, `RecsStatusDetails`) |
| Per-photo score fields in Photo Selector analytics | Verified as schema fields; in tension with Tinder's "counts only" statement; population in live traffic not shown |
| On-device likes-probability model + moderation (underage, text, explicit) | Corroborated by Tinder's own engineering blog |
| `fm.bin`/`mn.bin`, `rsrrFemale`/`rsrrMale`, numeric defaults, "your type" averaging | Reverse-engineer claim only |
| Bias percentages | Unreplicated; synthetic images, small n, no variance/correction, possible edit confounds; tests a 2022 photo-picker model, not the ranker |

---

## Appendix 2 — Artifacts, tools and file locations

| Item | Location |
|---|---|
| This document | `tinder.md` (repo root) |
| Final report | `reports/Tinder architecture and matching algorithm.md` |
| Research notes | `research_notes/Tinder architecture and matching algorithm/` (4 files) |
| REA source (copy of morluto/rea) | `/home/user/kartik030204/rea` (session container; not committed) |
| Public tinder.com assets + REA evidence (~100 MB) | session scratchpad `…/scratchpad/tinder/` (temporary; not committed) |
| tinder-protos clone | `/home/user/davis-3450/tinder-protos` (session container; not committed) |

Tools: REA 6.9.0 (`analyze-javascript-application`), Node 22.23.4, Python 3.13, LiteRT 2.3.0 (installed, unused — no model obtained), curl, grep, git. Network fetches to Tinder were limited to tinder.com's public HTML, entry bundles, service worker, manifest and ~5 small lazy chunks.

Not done, and why:
- Android APK not obtained (Cloudflare challenge on APKMirror; not bypassed).
- Models not extracted or run (current models are encrypted; the 2022 APK was not available).
- Jadx decompilation not run (download blocked by session permissions).
- Medium-hosted Tinder posts (geosharding, TAG) and help.tinder.com could not be fetched; secondary summaries used.

---

## Appendix 3 — Evidence labels used

| Label | Meaning |
|---|---|
| **[T1]** / ✅ | Tinder first-party (engineering blog, help center, official statements) |
| **[V]** / 🔬 | Verified directly in tinder.com client code or the tinder-protos schemas |
| **[P/A]** | Patent or peer-reviewed/academic source |
| **[RE]** / 📹 | Reverse-engineer claim only (e.g., the ayoships video) |
| **[X]** / ⚠️ | Contradicted or in tension with another source |
| 📰 | Third-party journalism or study |
