# Local code evidence: what Tinder's public web assets, REA analysis and scraped protobuf schemas show

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

## 1. Tech stack and delivery

### Takeaway
tinder.com is a server-side-rendered React 18.3.1 + Redux/redux-saga webpack SPA ("tinderweb") served by Express behind nginx, AWS ALB and CloudFront, with heavy code splitting (2,089 named lazy chunks), a Workbox service worker with web push, Sentry 7.42.2 observability, REST against api.gotinder.com plus protobuf (`application/x-protobuf`) schemas loaded lazily for keepalive/roomservice/mediaservice/dynamicui/authgateway, and a CSP (report-only) that whitelists `wss://keepalive.gotinder.com`.

### Cited Findings
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

### Inferences
- The `keepalive` websocket appears to be the push channel for server-driven "nudges" (refresh profile, dynamic UI, campaigns, typing, rooms) encoded with protobuf, while most CRUD (swipes, purchases) is JSON REST. This is consistent with the CSP and schema chunk names, but the actual wire framing was not observed.
- CSP being report-only with `unsafe-inline/unsafe-eval` means it is monitoring rather than enforcing.

### Gaps
- Exact Redux version, router library and CSS framework not confirmed (no clear version strings found). `12.20.0` string is a theme object version (`themeId`), not identified.
- REA endpoint nodes are "candidate" string literals; no HTTP methods/edges were extracted from REA itself (methods came from the main bundle's endpoint maps).

## 2. Swipe telemetry and per-recommendation server fields

### Takeaway
On every like/pass the web client builds a `recsStatusInfo` payload with dwell time (`duration_in_millis`), input `method` (button/keyboard/swipe/profileButton/profileKeyboard), media and profile-element view counts, the order and values of profile elements seen, shared interests, and the server-supplied `s_number`; the schema `RecsStatusDetails` (Hubble field 100) and `RecsCustomDetails` (field 114) define the full set. Server rec objects carry `s_number`, `is_superlike_upsell`, `is_traveling`, `recently_active`, `matched_preferences`, `teaser`, `experiment_info.user_interests`, and `liked_content` (swipe notes).

### Cited Findings
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

### Inferences
- `s_number` is a server-issued per-rec identifier echoed back on swipe (likely a ranking/session or experiment cohort token); its semantics are not visible client-side.
- `exceeds_age_range_by` / `exceeds_distance_by` in the schema imply recs can be served outside a user's stated preferences and that this is logged.
- Dwell time, media views and which profile elements were viewed are exactly the implicit-feedback signals a ranking model would consume; the artifacts prove collection, not use in ranking.

### Gaps
- No ranking scores, Elo/desirability values, or embeddings appear in rec payloads or schemas (searched `score|rank|embedding|elo|desirab|chemistry|smart_photo` across all 212 protos; only Photo Selector scores found).
- How `matched_preferences`, `teaser`, `s_number` are computed server-side is not observable.

## 3. Photo Selector analytics schemas and RSRR

### Takeaway
Eight `photo_selector_*` Hubble schemas describe an on-device (downloaded-model) camera-roll photo picker that scores photos with per-photo arrays `rsrr_scores`, `explicit_scores`, `text_scores`, `underage_scores`, `similarity_scores`, then filters by moderation (explicit, text, under-age, baby, less-clothed, mirror-selfie). On web, "RSRR" appears only as a Super Like upsell trigger (`superlike.upsell_high_rsrr_enabled`, `upsell_superlikes_high_rsrr`), tagged with `sourceSessionEvent:"revenue"`.

### Cited Findings
- `PhotoSelectorPhotosCollectionDetails`: `total_number`, `repeated rsrr_scores (FloatValue)`, `repeated photo_ids`, `repeated explicit_scores`, `repeated text_scores`, `repeated underage_scores`, `repeated similarity_scores` — [H/photo_selector_photos_collection_details.proto](H/photo_selector_photos_collection_details.proto)
- `PhotoSelectorUploadSelectedPhotosDetails`: `number_of_photos`, `percentage_of_scrolling`, `repeated indices_of_selected_photos`, `repeated ids_of_selected_photos`, `repeated rsrr_scores`, `repeated is_image_cropped`, `repeated cropped_info`, `repeated orientation` — [H/photo_selector_upload_selected_photos_details.proto](H/photo_selector_upload_selected_photos_details.proto)
- `PhotoSelectorModerationSelectionDetails`: `process_time, num_photos_entering, num_photos_returned, num_photos_after_deduped, num_photos_after_moderation, exit_status (SUCCESS/CANCELLED), num_photos_explicit_filtered, num_photos_text_filtered, num_photos_under_age_filtered, num_photos_less_clothed_flagged, num_landscape_photos_returned, num_portrait_photos_returned, num_photos_other_filtered, num_photos_baby_filtered, num_photos_mirror_selfie_flagged` — [H/photo_selector_moderation_selection_details.proto](H/photo_selector_moderation_selection_details.proto)
- `PhotoSelectorAssetProcessingDetails`: `process_time, num_photos_granted_access, num_photos_scanned, num_photos_returned, num_photos_face_detected, num_photos_selfie_identified, num_photos_scored, exit_status (ALL_SCANNED/CANCELLED/TIME_OUT), num_landscape/portrait_photos_granted_access` — [H/photo_selector_asset_processing_details.proto](H/photo_selector_asset_processing_details.proto)
- `PhotoSelectorModelDownloadDetails.model_download_outcome` (SUCCESS/FAILURE); `PhotoSelectorOutcomeDetails.outcome`; `PhotoSelectorSelectedPhotoDetails`: `photo_index, percentage_of_scrolling, is_selected`; `PhotoSelectorFeedbackDetails`: `length_of_feedback, feedback` — [H/](H/)
- Related: `MandatedFacePhoto*` schemas (`modal_type`, `number_days_left_to_add`, `face_upload_detected`); `UploadedPhotosDetails.has_face_photo_required` — [H/](H/)
- No `photo_selector`/`photoSelector` strings in any downloaded web JS — grep over [T/js/](T/js/), [T/chunks/](T/chunks/), [T/index.html](T/index.html)
- Web RSRR occurrences (6 total): flag constant `superlike.upsell_high_rsrr_enabled` (main); Redux upsell state `superlikeHighRsrr:{amountShown:0,shown:!1,timestamp:0}` with selectors on `.shown`/`.timestamp` (main); saga dispatch `upsellType:"superlikeHighRsrr", upsellTs:Date.now()` (main); analytics `{location:"discovery", name:"upsell_superlikes_high_rsrr", sourceSessionEvent:"revenue", type:"modal"}` with `action:"show"` in the `SuperlikeUpsellModal` chunk — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/js-upsell.js](T/js-upsell.js); 1 occurrence in [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)
- Related flags: `superlike.upsell_rate_limit_unit`, `superlike.attach_message_rate_limit`, `superlike.attach_message_targeting`, `superlike.attach_message_time_limit_sec`; rec field `is_superlike_upsell` — [T/flags.txt](T/flags.txt), [T/chunks/async-recs-feature.js](T/chunks/async-recs-feature.js)

### Inferences
- "RSRR" is not expanded anywhere in the artifacts. Its pairing with photo scores and with a "high RSRR → upsell Super Like" modal is consistent with a *right-swipe-rate/ratio*-type predicted metric (e.g., "received swipe right rate"), but this expansion is **interpretation, not observed**.
- The web upsell suggests the server flags certain recs (likely ones predicted to be in high demand) as Super Like upsell candidates (`is_superlike_upsell`) — a monetization lever tied to a desirability-like signal.
- Photo Selector appears to be a mobile, on-device ML feature (model download, camera-roll access counts, face/selfie detection) whose per-photo scores are sent as analytics.

### Gaps
- No artifact defines RSRR, the model, score ranges or whether scores influence ranking.

## 4. Monetization and experimentation levers

### Takeaway
Feature flags ("levers", served from a `/levers` route prefix) gate a large paywall/upsell surface: dynamic/personalized offers, discounts, bouncer paywalls, priority likes, Super Like upsells, match expiration and Likes You filtering. Web payments run through Braintree (card, Apple Pay, PayPal, Venmo) and Adyen (card, Apple Pay, Google Pay, 3DS), plus Blackhawk gift cards.

### Cited Findings
- Flag namespaces and notable flags: `paywall.alc_discount_offer_paywall_enabled`, `paywall.alc_discount_offer_paywall_cool_down_period_in_seconds`, `paywall.alc_discount_offer_paywall_rate_limit_value`, `paywall.boost_subscription_enabled`, `paywall.premium_rebrand_*_web`, `paywall.show_terms_with_refund`; `purchase.cancel_save_offer_enabled_web`, `purchase.cc_discount`, `purchase.one_page_checkout_enabled`, `purchase.web_google_pay_steering_enabled`, `purchase.web_venmo_subscription_enabled`, `purchase.web_apple_pay_skip_braintree_fingerprint`; `bouncer.bounce_to_gold`, `bouncer.bouncer_bypass_enabled`, `bouncer.bouncer_bypass_modal_variant`; `subscription_discount.personalized_offer_enabled`, `subscription_discount.personalized_offer_ny_enabled`; `discount.banner_enabled`, `discount.checkout_pill_enabled`, `discount.login_banner_enabled`; `gold.new_likes_insendio_modal_enabled`, `gold.new_likes_modals_frequency`, `gold.weekly_intro_offer_enabled`; `platinum.priority_likes_swipe_right_threshold`, `platinum.priority_likes_modal_time_interval`, `platinum.match_list_upsell_enabled`, `platinum.weekly_gold_to_plat_offer_enabled`, `platinum.(gold_)bouncer_paywall_cta_enabled`; `likes_you.filtering_enabled`, `likes_you.min_likes_count_to_enable_filtering`, `likes_you.quick_filter_passions`; `postmatch.match_expiration_enabled`, `postmatch.match_expiration_extend_enabled`, `postmatch.match_expiration_urgency_period_seconds`; `pricing.show_weekly_savings`; `web_steering.open_ios_paywall`, `web_steering.web_discount_percentage`; `first_impression.enabled`; `ads.web_ads`, `ads.web_banner_cadence`, `ads.web_cardstack_cadence`; `logging.should_sample_hubble_revenue_purchase_flow_events` — [T/flags.txt](T/flags.txt)
- Upsell/paywall chunk names: `SuperlikeUpsellModal`, `SuperlikePaywallDialog`, `PersonalizedOfferModal`, `CancelSaveOfferModal`, `BoostPaywallModal`, `BoostUpsellModal`, `BoostSubscriptionPaywall`, `PrimetimeBoostModal`, `PrimetimeBoostUpsellModal`, `SuperBoostPaywallModal`, `YForXBoostModal`, `FirstImpressionsPaywallModal`, `BundleOfferBanner/Dialog`, `IntroPriceAndSubOfferBanner`, `GoldHomeFilterTrayModal`, `pc-PlatinumPayWallDialogWithFeaturesList`, `pc-r-CheckoutPage`, `pc-r-PreCheckoutPage`, `remote-checkout`, `async-paywalls-*`, `async-offers-*`, `async-purchase-*`, `async-boost-*`, `intent-handler-getGold/goldHome/purchaseFlow/startBoost`, `paypal-checkout(-v6)-index-js`, `visa-checkout-index-js` — [T/chunknames.txt](T/chunknames.txt)
- Payment provider enum: `web_braintree_applepay`, `web_braintree_paypal`, `web_braintree_venmo`, `web_adyen_card`, `web_adyen_applepay`, `web_adyen_googlepay`, `web_blackhawk_giftcard`, `web_discount`, `apple_store`, `android_store`; `PAYMENT_TYPE_{APPLE_PAY, CARD, CREDIT_CARD, GOOGLE_PAY, PAYPAL, VENMO}`; Adyen 3DS state `adyenV2ThreeDSResult` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js)
- Purchase endpoints: `/v1/paywall/offers`, `/purchase/offer`, `/purchase/offer/view`, `/v2/purchase/braintree`, `/v2/purchase/braintree/client-token`, `/v2/purchase/adyen`, `/v2/purchase/adyen/payment-details`, `/v2/purchase/discount/view`, `/v2/purchase/tax/{zip}`, `/v2/purchase/{purchaseId}/auto-renew`, `/v2/consumables/bouncer-bypass`, `/v2/incentives/introprice/redeem`, `/v2/incentives/eligibility`, `/v1/incentives/gift-card/redemption`, `/v2/insendio/campaigns` — main bundle; REA endpoint nodes [T/rea-app.json](T/rea-app.json)
- Schema `PaywallDetails.FeatureType`: BOOST, GOLD, PLATINUM, REWIND, SUPER_BOOST, SUPER_LIKE, READ_RECEIPTS, PLUS, BOOST_SUBSCRIPTION, SUB_MERCHANDISING, CONSUMABLE, SUBSCRIPTION, PRIME_TIME_BOOST, LIKES_ANNUITY, MATCH_EXPIRATION, PURCHASED_LIKE; fields `paywall_version, upsells, template_ids` — [H/paywall_details.proto](H/paywall_details.proto)
- Schema `RevenuePurchaseFlowDetails` includes `age`, `base_price`, `currency`, `discount_campaign`, `is_intro_pricing`, `likes_num`, `paywall_from`, `paywall_version`, `promo_code`, `required3ds`, `revenue_session_id`, `sku`, `template_uuids`, `upsells` — [H/revenue_purchase_flow_details.proto](H/revenue_purchase_flow_details.proto)
- `CrmCampaignDetails`: `campaign_id, experiment_name, variant_name, content_branch` — [H/crm_campaign_details.proto](H/crm_campaign_details.proto)

### Inferences
- `platinum.priority_likes_swipe_right_threshold` and `gold.new_likes_modals_frequency` suggest upsell timing is tied to the user's own swiping activity and incoming-like counts.
- `age` in purchase-flow telemetry plus `subscription_discount.personalized_offer_enabled` is consistent with segmented/personalized pricing; the artifacts do not show the pricing logic.

### Gaps
- Flag values (on/off, variants) are not in static assets; only names.

## 5. Trust and safety

### Takeaway
The web client integrates Arkose Labs and Google reCAPTCHA (backend-selectable), FaceTec-based biometric liveness and selfie/ID verification, Jumio-backed age verification flows (incl. a Japan-specific underage flow), client-side automation/bot detection ("agent signals" checking webdriver/headless/Selenium/Nightmare artifacts), and a detailed ban-reason taxonomy and appeals system.

### Cited Findings
- Captcha: constants `"recaptcha"`, `"arkose"`, `CAPTCHA_V2`; `arkoseCaptcha:{baseUrl:"https://tinder-api.arkoselabs.com/v2", ...onboardingKey...}`; reCAPTCHA `scriptUrl:"https://www.google.com/recaptcha/api.js?..."`; `SET_ARKOSE_DATA_BLOB`; endpoint `/v2/auth/verify-captcha`, `/challenge/verify`; flags `captcha.logic`, `identity.backend_driven_web_captcha_enabled`, `identity.auth_guest_arkose_enabled` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/flags.txt](T/flags.txt)
- Bot/agent detection: arrays checking `"selenium","webdriver","driver"`, `/HeadlessChrome/`, `__webdriver_evaluate`, `__selenium_unwrapped`, `$cdc_asdjflasutopfhvcZLmcfl_`, `__nightmare`, empty `navigator.languages`; flags `integrity.agent_signals_enabled`, `integrity.agent_signal_block`; chunks `agent-signals`, `AgentBlockedDialog`; fingerprint keys in flag list `navigator.webdriver/platform/vendor/languages`, `webgl.renderer/vendor/extensions/version`, `screen.width/height`, `timezone.*`, `headless.webdriver`, `math.engine`, `fonts.available` — [T/js/main-a71729066faf0b508b60.js](T/js/main-a71729066faf0b508b60.js), [T/flags.txt](T/flags.txt), [T/chunknames.txt](T/chunknames.txt)
- Liveness/biometrics: endpoints `/v1/biometrics/liveness`, `/v2/biometrics/liveness`, `/v1/biometrics/liveness/consent`, `/v1/biometrics/reset/liveness`, `/v2/verification/facemap/{info,id-verify,photo-match}`, `/v2/verification/selfie/{start,complete}`, `/v1/identity-verification/{sdk/init, idv/challenge, idv/challenge-status, selfie/challenges, selfie/id-verify, selfie/verifications}`, `/match-settings/selfie-verification/{matchId}`; FaceTec UI strings `FaceTec_camera_permission_header`; flags `integrity.biometrics_liveness_bouncer_enabled`, `integrity.liveness_check_m2_variant`, `identity.biometrics_liveness_blocking_skippable`, `identity.liveness_data_removal_enabled`, `trust.facetec_sdk_version`, `trust.selfie_verification_enabled`, `trust.idv_variant`, `trust.selfie_v2_biometric_consent_screen_version`; chunks `BiometricsBouncer`, `BiometricsLivenessCheckModal`, `SelfieChallengeV2Modal` — main bundle, [T/flags.txt](T/flags.txt), [T/chunknames.txt](T/chunknames.txt)
- Age verification: `/1.0/age-verification/url`, `/verification/underage/{url,state,jp,jp/status}`, `generateJumioUrl`, flags `age_verification.eureka_age_verification_enabled`, `age_verification.post_onboarding_enabled`, `onboarding.age_gate_jumio_appeal_enabled`; chunks `AgeVerificationEurekaDialog`, `AgeGateDialog` — main bundle, [T/flags.txt](T/flags.txt)
- Other trust flags: `integrity.csh_v1_enabled`, `integrity.eu_mlc_variant`, `integrity.preauth_sk_consent_enabled`, `trust.dsa_custom_ban_screen`, `trust.checkr_rso_enabled` (Checkr; "rso" plausibly registered sex offender — interpretation), `trust.appeal_center_access_control`, `safety.chat_shield_fab_*`, `postmatch.blur_expired_match_messages` — [T/flags.txt](T/flags.txt)
- Ban/violation taxonomy in web: `VIOLATING_PERSONAL_CONTENT`, `VIOLATING_MEDIA`, `HARMFUL_BEHAVIOR`, `FALSE_REPORTING`, `MULTIPLE_USERS`, `IMPERSONATION_ACCOUNT`, `IMAGES_OF_OTHER_PEOPLE`, `COMPROMISED_ACCOUNT`; schema `BanReason` (24 values incl. COMMERCIAL_SEX, UNDERAGE, UNDERAGE_IMAGES, STOLEN_PHOTOS, ATO, PREVIOUSLY_BANNED) — main bundle, [H/ban_reason_details.proto](H/ban_reason_details.proto)
- Appeals/reports endpoints: `/v1/appeals`, `/v1/banned-user/appeals/cases`, `/v1/case/warnings/acknowledge`, `/v3/report/{primaryId}/{secondaryId}/{offenderId}`, `/v4/report/new`, `/v4/report/tree` — REA endpoints [T/rea-app.json](T/rea-app.json)

### Inferences
- Device/environment fingerprint attributes (WebGL, screen, timezone, fonts) are collected for integrity scoring; whether they feed Arkose or an in-house model is not determinable statically.

### Gaps
- No moderation model scores observed in web code; Photo Selector moderation counts are the only ML-moderation telemetry found (schema only).

## 6. Other algorithm-relevant schema fields (survey of hubble/details and related protos)

### Takeaway
Beyond recs and Photo Selector, schemas reveal top-picks, "swipe surge", "vibes", Let's Meet (date intent), duration/session measures and server-driven UI nudges — but no explicit ranking score, embedding or "chemistry" field exists anywhere in the 212 protos.

### Cited Findings
- 39 files in hubble/details (full list incl. `lets_meet_*`, `mandated_face_photo_*`, `hubble_duration_measure`, `network_request_details`, `share_my_date_details`) — [H/](H/)
- `HubbleDurationMeasure`: `category, type, subtype, duration, success, component_name, duration_millis, session_id`; `AppCloseDetails.session_length_seconds` — [H/](H/)
- Top Picks: `RecsCustomDetails.other_id_is_top_pick`; enums `TOP_PICKS`, `TOP_PICKS_ALC_ACCESS`, `TOP_PICKS_INTRO_V3` in [P/tinder/services/profile/profile_options.proto](P/tinder/services/profile/profile_options.proto); `SESSION_ATTRIBUTION_TOP_PICKS` in [P/tinder/events/model/common/message.proto](P/tinder/events/model/common/message.proto)
- Swipe Surge / Vibes: `value_profile_element_swipe_surge`, `value_profile_element_vibes` (RecsStatusDetails); `TAPPY_CONTENT_TYPE_SWIPE_SURGE` in [P/tinder/services/dynamicui/tappy_content/tappy_content.proto](P/tinder/services/dynamicui/tappy_content/tappy_content.proto)
- Let's Meet: `interested_in_date`, `is_matched`, `time_frame_and_description`, `date_suggestion_id` — [H/lets_meet_date_details.proto](H/lets_meet_date_details.proto)
- Server-driven UI: `tinder/services/dynamicui/clientnudge/{client_nudge_action,client_nudge_event}.proto`, insendio campaign DSL (`insendio/dsl/attribute`, `campaign/templates`) — [P/tinder/services/](P/tinder/services/)
- Explicit search for `score|rank|embedding|smart_photo|chemistry|elo|desirab|attractiv|dwell|algorithm|model_version` across all protos returned only Photo Selector score fields and the Top Picks/swipe-surge/vibes items above — grep over [P/](P/)

### Inferences
- The client-visible surface is telemetry plus server-issued tokens; ranking happens server-side and is not exposed in client schemas.

### Gaps
- No artifact shows Smart Photos ordering logic, Elo, or chemistry scoring.
