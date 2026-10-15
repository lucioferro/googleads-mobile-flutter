# Next-Gen Android Anchored Adaptive Banner Patch

## Summary

This patch resolves an issue where Anchored Adaptive Banners on Android take 1 to 3 minutes to display (or fail silently with no fill) when using the GMA Next-Gen SDK (`USE_NEXT_GEN_SDK=true`).

Target file:
`android/src/adsNextGenSdk/java/io/flutter/plugins/googlemobileads/FlutterBannerAd.java`

---

## Root Causes

### 1. Stripping the Anchored Adaptive Banner Flag

In the original `FlutterBannerAd.java`:
```java
final List<AdSize> allSizes = new ArrayList<AdSize>();
for (int i = 0; i < sizes.size(); i++) {
  FlutterAdSize flutterAdSize = sizes.get(i);
  if (flutterAdSize.getAdSize().isFluid()) {
    allSizes.add(AdSize.FLUID);
    continue;
  }
  AdSize adSize =
      new AdSize(flutterAdSize.getAdSize().getWidth(), flutterAdSize.getAdSize().getHeight());
  allSizes.add(adSize);
}
```

* `flutterAdSize.getAdSize()` already returns a properly configured `AdSize` created by `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize(...)`, where `isAnchoredAdaptiveBanner` is `true`.
* Calling `new AdSize(width, height)` instantiates a custom fixed size, clearing `isAnchoredAdaptiveBanner` to `false`.
* Bidding demand and mediation adapters (Meta Audience Network, Unity, Mintegral, Pangle, Chartboost) cannot match custom fixed sizes and return **Error Code 3 (NO FILL)** immediately.

### 2. Swallowing the Load Failure Callback

In the original `FlutterBannerAd.java`:
```java
@Override
public void onAdFailedToLoad(@NonNull LoadAdError adError) {
  bannerAd = null;
}
```

* The callback sets `bannerAd = null` but never invokes `manager.onAdFailedToLoad(adId, ...)`.
* As a result, the Flutter Dart layer never receives `BannerAdListener.onAdFailedToLoad`.
* The banner remains stuck in an un-notified loading state until a watchdog timer expires or until a background auto-refresh occurs several minutes later.

---

## Upstream Clarification (PR #1495)

The changelog for version 9.2.0 mentions:
> *Fixes Large Anchored Banner Size. [PR 1495]*

Commit `9bf993a87d0a87d113643f0f656adf75f63067f9` (PR #1495) only added support for serializing the `isLarge` boolean flag in `AdMessageCodec.java` and `FLTGoogleMobileAdsReaderWriter_Internal.m`. It did not modify `FlutterBannerAd.java`.

---

## Applied Changes

In `packages/google_mobile_ads/android/src/adsNextGenSdk/java/io/flutter/plugins/googlemobileads/FlutterBannerAd.java`:

```diff
@@ -94,13 +94,7 @@ class FlutterBannerAd extends FlutterAd implements FlutterAdLoadedListener {
     final List<AdSize> allSizes = new ArrayList<AdSize>();
     for (int i = 0; i < sizes.size(); i++) {
       FlutterAdSize flutterAdSize = sizes.get(i);
-      if (flutterAdSize.getAdSize().isFluid()) {
-        allSizes.add(AdSize.FLUID);
-        continue;
-      }
-      AdSize adSize =
-          new AdSize(flutterAdSize.getAdSize().getWidth(), flutterAdSize.getAdSize().getHeight());
-      allSizes.add(adSize);
+      allSizes.add(flutterAdSize.getAdSize());
     }
     BannerAdRequest adRequest = request.toBannerAdRequestBuilder(adUnitId, allSizes).build();
     adView.loadAd(
@@ -123,6 +117,7 @@ class FlutterBannerAd extends FlutterAd implements FlutterAdLoadedListener {
           @Override
           public void onAdFailedToLoad(@NonNull LoadAdError adError) {
             bannerAd = null;
+            manager.onAdFailedToLoad(adId, new FlutterAd.FlutterLoadAdError(adError));
           }
         });
   }
```

---

## Verification

* **Before patch:** Banner load took 60s to 180s due to repeated NO FILL (code 3) and swallowed callbacks.
* **After patch:** Anchored adaptive banners load on the initial attempt (`gen=1`) with fill latency of **~641ms**, matching native Android performance.


## Follow-up correction — 2026-10-15

The 641ms observation above is one successful request, not proof that the patch
resolved production no fill. Build 2537 still reports AdMob-source `Internal error.`
and Mintegral empty responses; the observed Home first fill came from Mintegral
after 30,786ms. The root-cause wording above should be read as the motivation for
the original size/callback fix, not a demonstrated cause of every no-fill result.

The follow-up bridge correction preserves the existing adaptive size fix and:

- Removes explicit `registerBannerAd()` after `AdView.loadAd()`. The SDK already
  registers/resizes the banner before notifying success. An explicit registration
  can cancel pending requests owned by `loadAd()`.
- Forwards SDK refresh success/failure to Flutter, updating the current response
  info and retaining a loaded banner when a refresh fails.
- Executes view-related callbacks on the main looper, ignores obsolete callbacks
  after disposal/replacement, and returns null for size before first fill.
- Does not add another load call, timers, application retries, or SDK changes.

Six focused Robolectric tests cover those lifecycle cases against Next-Gen 1.5.0.
They validate the bridge, not production fill or physical-device rendering.
The reproduction tests and incremental patch are kept in the Flutter app under
`tool/ads/`; the investigation is in
`docs/banner-production-followup-2026-10-15.md`.

References:

- https://developers.google.com/admob/android/next-gen/reference/com/google/android/libraries/ads/mobile/sdk/banner/AdView
- https://developers.google.com/admob/android/next-gen/banner#listen_for_ad_events
