# Inside Tinder: Funnels, Ratios, and Photo Models

Tinder runs as a cloud-native microservice system: **500+ services on Kubernetes behind a config-driven Spring Cloud Gateway and an Envoy mesh, with Kafka as the source of truth and one geo-sharded Elasticsearch cluster serving over 90% of recommendations**. Ranking is a four-stage funnel (eligibility filtering, retrieval, ranking, reranking). Since at least 2026 it has been driven by two-tower deep models that predict P(Like) and P(Match) and run as custom scoring plugins inside Elasticsearch. Tinder says publicly that it retired the Elo "desirability score" in 2019. Popularity has not gone away, though. It survives as a learned signal. A 2024 Match Group patent claims an "attractiveness score" computed as the ratio of positive swipes to views. The production web client still uses a metric called **RSRR** (most plausibly "right-swipe received ratio") for one purpose: showing a Super Like upsell, tagged `revenue`, when you are about to like someone in high demand. Photo ML has moved through three generations. First came a 2016 epsilon-greedy bandit (Smart Photos). Next came on-device TFLite models that score your camera roll for "probability of receiving likes" and run moderation. The current generation is a 2026 pairwise vision-language model trained on bandit outcomes. The October 2026 "I reverse engineered Tinder" video by Ayo O (@ayoships) holds up on its static findings: RSRR naming, the revenue-tagged upsell, per-photo score fields and dwell-time telemetry all check out. Its behavioral claims are weak, especially the photo-bias experiment. That experiment probes a 2022-era on-device photo-picker model with about 800 synthetic images and small per-condition samples. It shows what that model outputs, not how Tinder ranks real people today. For a developer, Tinder is a useful case study in serving ranking inside the search engine, treating the event log as the database, and labelling ML with bandits.

**Evidence key used throughout:** **[T1]** = Tinder/Match first-party statement or engineering post; **[V]** = verified in shipped tinder.com client code or the public tinder-protos schema dump; **[P/A]** = patent or peer-reviewed/academic work; **[RE]** = reverse-engineer claim only, not independently confirmed; **[X]** = contradicted by, or in tension with, other evidence. A label attaches to the claim it follows.

## Five hundred services sit behind one config-driven gateway

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

## Recommendations run as a four-stage funnel inside Elasticsearch

Tinder names the pipeline as **eligibility filtering, then retrieval, then ranking, then reranking** ([Tinder Eng, fresh recs](https://lifeattinder.com/blog/how-to-keep-recommendations-fresh)) **[T1]**.

**Eligibility** is a hard filter on location, age, stated preferences and per-pair serving state. For each viewer and candidate, the system tracks the last interaction time, whether it was a pass, and how many times the candidate has been shown in a rolling window. A "dynamic recycling" policy keeps recent passes blocked, lets older passes back in through limited tiers with a cap, and gives priority to profiles the viewer has not seen. Tinder separates "eligibility" (may this person be shown?) from "competitiveness" (should they win the slot?). It found that letting its newest deep retrieval and ranking models score recycled candidates, instead of hard-excluding them, increased both the eligible pool and engagement. The post gives no figures for this **[T1]**. The schema shows that the system logs when a rec falls outside the viewer's preferences, through fields such as `exceeds_age_range_by` and `exceeds_distance_by` ([tinder-protos](https://github.com/Davis-3450/tinder-protos)) **[V]**. Recs can therefore deliberately leak outside your stated filters (inference).

**Retrieval and ranking** use embeddings. Tinder described "TinVec" in 2017. It embedded users' preferences into vectors learned from swipes, at a time when the service was producing 26M matches a day ([MLconf](https://mlconf.com/?p=8768)) **[T1]**. The ES8 migration enabled kNN vector search, and its online tests reported the clearest numbers Tinder has published on its ranker ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)) **[T1]**:

- a **two-tower "2T P(Match)" model** raised match rate by +6.5% and match volume by +22%;
- a **"2T P(Like)" model** raised swipe-right rate by +3.8%.

A profile-embedding "MoreLikeThis" feature shipped alongside them, and p99 latency fell 12–56%. The objective is therefore reciprocal. P(Like) predicts whether the viewer will swipe right on the candidate. P(Match) also needs the candidate to like the viewer back, and that model produced the larger gains. Reciprocal relevance has an older patent lineage. US 11,989,220 trains a gradient-boosted tree ranker on two-way implicit signals, such as one-way versus two-way messages and views, imbalances and timing. It claims to rank candidates who have no interaction history ([patent](https://patents.google.com/patent/US20200175047A1)) **[P/A]**. Researchers call this family of methods reciprocal recommenders ([Palomares et al. survey](https://arxiv.org/abs/2007.16120)) **[P/A]**.

Tinder's own description of its embeddings is cautious. A 2026 internal exploration clustered more than 300,000 US Gen Z women's recommendation embeddings into five groups. The largest group (25.7%) had richer profiles and sent more likes but *received* fewer, a "profile-rich, high-outbound, lower-reciprocity" pattern. The authors stress that the embeddings "were built to improve recommendations, not to explain people" ([Tinder Eng](https://lifeattinder.com/blog/recommendation-embeddings)) **[T1]**. What replaced Elo, end to end, has never been described first-party. The model architecture beyond "two-tower," the feature set and the reranking objectives all remain unknown.

## Elo was retired, but a swipe-ratio popularity score never left

The history runs in three steps.

In **2016**, Tinder confirmed to Fast Company that it ran an internal Elo-style "desirability" rating. Sean Rad said, "It's not just how many people swipe right on you," and the reporter was shown his own score of 946 ([Refinery29 summarising Fast Company](https://www.refinery29.com/en-us/2016/01/101011/tinder-secret-score)) **[T1]**.

In **2019**, Tinder called Elo "outdated" and said it no longer relies on it. Its replacement explanation is that potential matches adjust as your profile is liked or passed, within about 24 hours ([Engadget](https://www.engadget.com/2019/03/18/tinder-dumps-desirability-scores); [Tinder Help](https://www.help.tinder.com/hc/en-us/articles/7606685697037-Powering-Tinder-The-Method-Behind-Our-Matching)) **[T1]**. A 2017 GDPR request by journalist Judith Duportail returned about 800 pages of personal data but no score, and Tinder treated its matching tools as core IP ([FlowingData](https://flowingdata.com/2017/12/01/800-pages-of-tinder-data/)) **[T1/journalism]**. Ireland's Data Protection Commission fined Match €11M in November 2026 over gaps in its "Download My Data" tool. The fine concerned completeness of the data export, not ranking ([Extra.ie](https://extra.ie/2026/11/13/news/tinder-handed-major-fine)).

The patent record shows the concept persisting. **US 12,105,941**, granted to Match Group in October 2024, claims an **attractiveness score defined as the ratio between how often a profile is viewed and how often others express a positive preference**. That score feeds a viewer-specific similarity score, which orders the card stack ([Google Patents](https://patents.google.com/patent/US12105941B2)) **[P/A]**. The specification also describes showing users with similar ratings to each other, a mechanism EPFL researchers characterise as stratifying users by attractiveness, IQ and similarity ([Pidoux et al.](https://aiequalitytoolbox.github.io/social-impact-tinder-pidoux.html)) **[P/A]**. A patent does not prove production use.

The client code does prove that a swipe-ratio metric is live. The production web bundle contains a feature flag `superlike.upsell_high_rsrr_enabled`, Redux state `superlikeHighRsrr`, and a `SuperlikeUpsellModal` analytics event `upsell_superlikes_high_rsrr` whose `sourceSessionEvent` is `"revenue"`. Recommendation objects carry an `is_superlike_upsell` boolean **[V]**. The acronym is never expanded in the code. "Right-swipe received ratio," the expansion the video offered, matches the patent's ratio exactly, but it remains an interpretation **[RE, consistent with P/A]**.

The academic baseline explains why this matters. On messaging networks, people contact partners about **25% more desirable than themselves** on average, and reply rates fall as that gap widens ([Bruch & Newman, Science Advances 2018](https://pmc.ncbi.nlm.nih.gov/articles/PMC6082652)) **[P/A]**. Early Tinder data showed women matching quickly and men slowly, which creates a feedback loop toward extreme strategies ([Tyson et al.](https://arxiv.org/abs/1607.01952)) **[P/A]**.

My reading is that Tinder replaced one explicit scalar with learned models that absorb popularity implicitly. A P(Like) tower trained on population swipes encodes "how many people like this kind of profile," and P(Match) explicitly weighs whether the candidate is likely to reciprocate. "No Elo" and "popularity still governs exposure" can both be true.

## Every swipe ships dwell time, input method and profile-element views

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

## Photo ML evolved from bandits to on-device TFLite to a 7B VLM

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

## Popularity signals are wired directly into paywalls

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

## The ayoships video gets the strings right and the science wrong

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

## Engineering lessons worth stealing

**Rank where the data lives.** Instead of pulling candidates out to a separate ranking service, Tinder ships its scoring logic and ML into the search engine as a plugin. Filtering, kNN retrieval and model scoring then happen in one hop over a geo-sharded index. The cost of this choice is lock-in: the plugins are why Tinder could not move to OpenSearch. ES8 also banned negative scores, which forced Tinder to audit 100+ scripts that had minimal tests. If you go this route, put your scoring scripts under the same test discipline as application code ([Tinder Eng, ES8](https://lifeattinder.com/blog/tinders-migration-to-elasticsearch-8)).

**Shard by geography with cells, not cities.** S2-style cells sized to query radius cut CPU about fourfold. Raising shards per node and adding primaries for scoring-heavy queries traded roughly 2× CPU for 25–30% lower latency. Measure that trade-off explicitly.

**Make the event log the source of truth.** With Kafka as the record, an index becomes a rebuildable projection: backfill, then tail. That made a zero-outage engine migration possible. Tinder verified the result with random field-by-field document checks (<0.2% discrepancy) and evaluated rankers offline by **replaying real traffic and comparing top-20 and top-100 lists by set overlap and Levenshtein distance**, before running three waves of online A/B tests. For consumers, acknowledge after processing, not before, and use retry topics plus a dead-letter topic ([Tinder Eng](https://lifeattinder.com/blog/dig-deeper-to-scale-further)).

**Separate eligibility from competitiveness.** Hard filters decide who *may* appear, and the ranker decides who *wins*. Keeping the two separate let Tinder recycle passed profiles under caps without hand-tuned exclusion rules ([Tinder Eng](https://lifeattinder.com/blog/how-to-keep-recommendations-fresh)).

**Optimize the reciprocal objective.** In a two-sided market, P(Match) beat P(Like) on match volume (+22% vs +3.8% on swipe-right rate). If your product succeeds only when both sides say yes, model both sides.

**Use bandits as label factories.** Smart Photos' exploration data became clean pairwise labels for a VLM. Filtering to statistically significant pairs beat using more data. Pairwise framing also removes per-person baselines, since the model compares two photos of the same person.

**Push ML to the edge when the data is sensitive, and make the telemetry honest.** The on-device pipeline does this well: encrypted model delivery, cheap face gates before expensive scoring, and moderation only on the top 100. The schema tension is the lesson. If you promise "counts only," do not leave score arrays in your analytics schema.

**Make the server push signals, not state.** A protobuf WebSocket that sends "refresh X" nudges keeps clients simple and lets backends own freshness. Lazily loading protobuf schemas keeps the initial bundle small.

**Know what your ranker optimizes for socially.** A model trained on aggregate swipes inherits population bias, and tying popularity signals to paywalls invites regulatory and reputational scrutiny, as the age-pricing settlement and the 2026 DPC fine show. Publishing disparate-impact testing is cheap insurance that Tinder has so far skipped.

## Conclusion

The common question "does Tinder still have an Elo score?" is the wrong one. Tinder no longer stores a single rating that you could request through a data-access request. Popularity is spread across learned P(Like)/P(Match) towers, a patented views-to-likes ratio, and an RSRR signal that the shipped client uses in exactly one place: showing a revenue-tagged upsell. That turns a transparency question into an architectural one. Each rating is now implicit in model weights, so users cannot inspect it and regulators cannot easily audit it. The monetization coupling is visible only because client code reveals where the signal reaches the paywall.

For the ayoships video, the lesson is about method rather than Tinder. Static artifacts such as strings, flags and schemas are reliable and were largely confirmed. Behavioural inferences drawn from probing an old client-side model are not. The most consequential open question is whether those per-photo score arrays leave the device. A disclosed privacy audit of the mobile client could settle it, and nobody has yet published one.
