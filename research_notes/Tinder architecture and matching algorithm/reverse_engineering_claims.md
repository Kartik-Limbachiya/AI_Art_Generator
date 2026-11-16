# Independent Reverse-Engineering Claims About Tinder Internals (Ayo O video, Oct 2026) and Their Credibility

Scope note: literature and web research only, current as of 16 Nov 2026. No APK, model or Tinder service was accessed. Evidence labels used below:
- **VERIFIED**: confirmed by the coordinator's own check of tinder.com client code or the public tinder-protos schema, or by Tinder's own published material.
- **CLAIMED**: rests only on Ayo's video or secondhand summaries of it.
- **CONTRADICTED / IN TENSION**: conflicts with Tinder's published statements or other evidence.

## 1. Where is the original video, how was it made, and how was it received?

### Takeaway
The original is a 79-second vertical video titled "I reverse engineered Tinder," posted on X on 20 Oct 2026 by Ayo O (@ayoships / @TheDonAyo, co-founder of a0.dev, YC W25). It got about 2.4M views. No first-party write-up or repo from Ayo was found. The method shown is decompiling a 2022 Android build, reading iOS strings, and running the bundled TFLite model offline on more than 800 AI-generated images. As of mid-Nov 2026, no Tinder/Match Group response and no mainstream tech-press coverage were found. The video sits in a serial "I reverse engineered X" format (TikTok, Block Blast, Royal Kingdom), with at least one entry explicitly AI-assisted.

### Cited Findings
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

### Inferences
- The primary artifact is a short video, not a reproducible write-up. Without released code, image sets, prompts or model hashes, nobody can replicate the synthetic experiment exactly.
- Major outlets (The Verge, 404 Media, TechCrunch, Wired) appear not to have covered it. Searches surfaced only blog and SEO-style write-ups, so the claims have not been through journalistic fact-checking or a company request for comment.
- The video belongs to a high-volume content series, which favours speed and virality over rigour. That is not disqualifying, but it raises the bar for the strongest claims, especially bias magnitudes.

### Gaps
- The video transcript and on-screen screenshots could not be fetched directly. The claim list comes from the coordinator's brief plus the hegetswipes summary.
- The repo behind the t.co link in @imryanarya007's post is unknown, as is whether it is Ayo's or a copycat.
- Reddit and Hacker News threads on the video were not found by search.
- No statement from Tinder or Match Group was found. The ~295 replies to the tweet were not readable.

## 2. Claim-by-claim evaluation: verified vs claimed vs contradicted

### Takeaway
The telemetry and monetisation claims are well corroborated by tinder.com client code and the public tinder-protos schema: view-duration telemetry, the high-RSRR Super Like upsell tagged "revenue", and per-photo rsrr/explicit/underage/similarity score fields in the Photo Selector schema. The on-device models are partly corroborated by Tinder's own engineering blog, which confirms an in-house TFLite "likes" scoring model plus a moderation model. The specific model files, their outputs (rsrrFemale/rsrrMale) and the "type averaging" mechanism remain claims. The clearest tension is that the score schema exists while Tinder says Photo Selector is fully on-device with counts-only analytics. Schema existence alone does not prove the fields are populated and transmitted.

### Cited Findings

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

### Inferences
- The strongest claims are the ones that need only static artifacts: field names, strings, flags. Several are now independently matched on tinder.com.
- The weakest claims are about behaviour: how ranking uses these signals, whether scores are actually uploaded, and whether "type averaging" drives recommendations. Static analysis of a 2022 client cannot show server-side use.
- The schema tension is real and reportable. It should be framed as "schema supports transmitting per-photo scores; Tinder says only counts are collected; live traffic not publicly demonstrated."

### Gaps
- Live-traffic confirmation of populated `rsrr_scores` in PhotoSelector uploads has not been publicly shown.
- No source independently confirms the fm.bin/mn.bin file names or sizes, or the rsrrFemale/rsrrMale outputs.
- The tinder-protos repo does not state its extraction date, so it cannot be tied to a specific client version.
- No Tinder or Match Group comment on the RSRR upsell.

## 3. Other independent reverse-engineering work on Tinder (architectural level only)

### Takeaway
Tinder has a decade-long history of credible independent security research that Tinder acknowledged and fixed:
- location trilateration (Include Security, 2013–14; later Robert Heaton);
- Checkmarx's 2018 finding of unencrypted image traffic and swipe size-inference, corroborated by Sophos.

There is also a hobbyist ecosystem of scraped protobufs and unofficial API clients. Ayo's work differs because it targets ML models and ranking signals rather than security flaws, and it has not been through a disclosure process.

### Cited Findings
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

### Inferences
- Prior Tinder research that earned credibility shared three traits: reproducible demonstrations, vendor disclosure, and independent corroboration (e.g., Sophos re-testing Checkmarx). Ayo's work currently has partial corroboration of static artifacts but none of the three for the bias experiment.
- The Checkmarx packet-size finding is a precedent showing swipe behaviour is observable telemetry. That fits the newer finding that dwell time and view counts are logged.

### Gaps
- No academic paper doing static or dynamic analysis of Tinder's ML components was found.
- No independent replication of Ayo's model-probing experiment was found.

## 4. Methodological critique of the synthetic-photo bias experiment

### Takeaway
The synthetic experiment can at most show what one 2022-era on-device scoring model outputs for out-of-distribution AI images. It cannot show how Tinder ranks real people today. Specific weaknesses:
- distribution shift from generated images;
- confounded edits, especially skin tone with exposure and contrast;
- tiny per-condition samples (some n≈8–16) with no reported variance;
- many uncorrected comparisons;
- unknown mapping from model outputs to server ranking;
- model age.

Even so, the direction of several findings matches a well-documented literature on demographic bias in attractiveness and saliency models. The results are plausible hypotheses, not established measurements.

### Cited Findings
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

### Inferences
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

### Gaps
- The image generator, prompts, edit procedure and scoring method (which output head; per-gender heads) are undisclosed, which blocks a precise critique.
- No study was found that specifically audits dating-app photo-scoring models for skin-tone bias.
- It is unknown whether the 2022 model is still shipped in current builds.
