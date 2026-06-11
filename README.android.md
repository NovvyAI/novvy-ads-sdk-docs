# Novvy Ads Android SDK

Android SDK for programmatic ad delivery, supporting interstitial, rewarded, banner, and feed ad formats via real-time bidding (RTB).

## Features

- **Interstitial Ads** — Full-screen ads shown at natural transition points (e.g. between episodes)
- **Rewarded Ads** — Full-screen ads that grant a reward when the user closes the ad
- **Banner Ads** — 260×90 dp native views with server-driven show/hide scheduling tied to video playback position
- **Feed Ads** — Native image or H5-zip ads inserted into scrollable feeds with visibility-driven playback countdown
- **AdMob Mediation** — Google AdMob mediation adapter included

## Quick Start

1. [Install the SDK](#installation) and add the required manifest permissions
2. [Initialize](#initialization) once in `Application.onCreate()`
3. Load and show ads — [Interstitial](#interstitial-ads) | [Rewarded](#rewarded-ads) | [Banner](#banner-ads) | [Feed](#feed-ads)
4. (Optional) [Set user and content context](#user--content-context) to improve targeting

---

## Requirements

- Android **minSdk 24** (Android 7.0+)
- Kotlin / Java 11+

## Installation

The SDK is published on Maven Central. Add the dependency in your app's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("ai.novvy.android:sdk:1.0.0-beta22")
}
```

Add the required permissions to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

## Initialization

Initialize once, as early as possible (e.g. in `Application.onCreate()`):

```kotlin
NovvySDK.initialize(
    context  = applicationContext,
    appId    = "your-app-id",
    endpoint = "https://your-bid-server.com",
    apiKey   = "your-api-key"
) { success ->
    // Optional: called when initialization completes
}
```

`initialize()` may be called again at any time — for example, after a user logs in or out — to update credentials and trigger a fresh server handshake.

## Interstitial Ads

```kotlin
val interstitialAd = NovvyInterstitialAd(adUnitId = "your-ad-unit-id")

interstitialAd.delegate = object : NovvyInterstitialAdListener {
    override fun onAdLoaded(ad: NovvyInterstitialAd) {
        ad.show(activity)
    }
    override fun onAdFailedToLoad(ad: NovvyInterstitialAd, error: Throwable) { }
    override fun onAdShown(ad: NovvyInterstitialAd)   { }
    override fun onAdClosed(ad: NovvyInterstitialAd)  { }
    override fun onAdClicked(ad: NovvyInterstitialAd) { }
}

interstitialAd.load()
```

Check `interstitialAd.isReady` before calling `show()` if you load and display at different points in time.

## Rewarded Ads

```kotlin
val rewardedAd = NovvyRewardedAd(adUnitId = "your-ad-unit-id")

rewardedAd.delegate = object : NovvyRewardedAdListener {
    override fun onAdLoaded(ad: NovvyRewardedAd) {
        ad.show(activity)
    }
    override fun onAdFailedToLoad(ad: NovvyRewardedAd, error: Throwable) { }
    override fun onAdShown(ad: NovvyRewardedAd)   { }
    override fun onAdClosed(ad: NovvyRewardedAd)  { }
    override fun onAdClicked(ad: NovvyRewardedAd) { }
    override fun onUserEarnedReward(ad: NovvyRewardedAd, reward: NovvyReward) {
        // Grant reward: reward.amount, reward.type
    }
}

rewardedAd.load()
```

`onUserEarnedReward` fires unconditionally when the user closes the ad — the SDK does not distinguish between "watched to completion" and "dismissed early". Grant the reward whenever this callback is received.

## Banner Ads

Banner ads render natively (no WebView) and automatically bind to the container's lifecycle. The server can return multiple ads per request, each with its own display time windows. The SDK supports:

- **Multi-ad selection**: Multiple ads are preloaded in parallel. When the player pauses at a position matching an ad's time window, one eligible ad is randomly selected and displayed.
- **Playback-position windows**: Each ad specifies `playback_positions` (e.g. `["1000,5000", "10000,15000"]`). The banner only appears when the player is paused within a matching window. When playback resumes, the banner hides. If the current position does not fall inside any window, or if `playback_positions` is empty, the banner is not shown.

### Player Adapter

Implement `NovvyPlayerAdapter` to let the SDK monitor playback state:

```kotlin
val playerAdapter = object : NovvyPlayerAdapter {
    override val currentPositionMs: Long get() = exoPlayer.currentPosition
    override val isPlaying: Boolean get() = exoPlayer.isPlaying
}
```

Both properties are polled at ~500 ms intervals — keep the getters lightweight (no I/O or locks).

- `currentPositionMs` — return the raw player position with no caching or rounding.
- `isPlaying` — return `true` only when the player is **actively playing**. Return `false` during pause, buffering/loading, and seek. The SDK relies on this to decide when to show and hide the banner.

### Load and Attach a Banner

Banner ads are not supported on Huawei devices or devices with a Chrome kernel older than version 116. On these devices, `load()` will immediately call `onAdFailedToLoad` with `NovvyError.InternalError("Unsupported device")`.

Banner ads follow the same load-then-display pattern as other ad formats:

```kotlin
val bannerAd = NovvyBannerAd(adUnitId = "your-ad-unit-id")

bannerAd.delegate = object : NovvyBannerAdListener {
    override fun onAdLoaded(ad: NovvyBannerAd)                        { }
    override fun onAdFailedToLoad(ad: NovvyBannerAd, error: Throwable) { }
    override fun onBannerShown(ad: NovvyBannerAd)                     { }
    override fun onBannerHidden(ad: NovvyBannerAd)                    { }
    override fun onAdImpression(ad: NovvyBannerAd)                    { }
    override fun onAdClicked(ad: NovvyBannerAd)                       { }
    override fun onAdClosed(ad: NovvyBannerAd)                        { }
}

bannerAd.load()
```

Once the ad is loaded (i.e. `onAdLoaded` fires or `bannerAd.isReady == true`), attach it to any `ViewGroup` container along with a player adapter:

```kotlin
bannerAd.attach(
    container = binding.bannerContainer,  // Any ViewGroup
    player    = playerAdapter
)
```

The banner has a fixed width of **260dp**, left-aligned within the container.

The SDK automatically monitors playback and determines when to show/hide the banner based on the server-configured display mode. The banner is automatically removed when the container's lifecycle is destroyed. To remove it manually:

```kotlin
bannerAd.detach()
```

When the user taps the banner's close button, `onAdClosed` fires and `detach()` is called automatically — no need to call it again in your callback.

## Feed Ads

Feed ads are designed for vertical feed/pager integration. The SDK supports two creative formats and selects the renderer automatically based on the server response — host code is identical for both:

- **Image creative** — renders natively with a hero image and a product info overlay (thumbnail, title, price, CTA).
- **H5 zip creative** — renders an interactive H5 bundle (HTML/CSS/JS, optional video) in a WebView. The zip is downloaded once per `material_url` and cached on disk; subsequent loads of the same creative reuse the cache.

Both formats share the same playback countdown, swipe-to-skip overlay, click handling, and listener callbacks.

```kotlin
val feedAd = NovvyFeedAd(
    adUnitId               = "your-ad-unit-id",
    durationSeconds        = 4,   // Optional: countdown duration (default 4, minimum 3)
    useDefaultSwipeOverlay = true // Optional: set false to supply a custom swipe UI
)

feedAd.delegate = object : NovvyFeedAdListener {
    override fun onAdLoaded(ad: NovvyFeedAd)                        { }
    override fun onAdFailedToLoad(ad: NovvyFeedAd, error: Throwable) { }
    override fun onAdImpression(ad: NovvyFeedAd)                    { }
    override fun onAdClicked(ad: NovvyFeedAd)                       { }
    override fun onAdPlaybackTick(ad: NovvyFeedAd, remainingSeconds: Int) {
        // Update countdown UI; remainingSeconds == 0 means completed
    }
}
```

When `useDefaultSwipeOverlay = false` the SDK skips its built-in "Swipe to skip" overlay; drive your own UI from `onAdPlaybackTick` instead.

```kotlin
feedAd.load()
```

`load()` accepts an optional `episodeNumber` parameter that overrides the episode number in the bid request for this specific load call (takes precedence over `NovvySDK.setContentContext`):

```kotlin
feedAd.load(episodeNumber = 4)
```

Once the ad is loaded, attach it to any `ViewGroup` container:

```kotlin
feedAd.attach(container = binding.feedAdContainer)  // Any ViewGroup
```

Control playback by notifying the SDK when the ad is visible (e.g. the current page in a pager). A common threshold is 50% visible:

```kotlin
feedAd.setPlaying(true)   // Fires impression (once) and starts countdown
feedAd.setPlaying(false)  // Stops and resets countdown
```

For H5 creatives, `setPlaying(false)` also pauses any `<video>`/`<audio>` elements. When the user swipes back (next `setPlaying(true)` after a deactivation), the H5 page is reloaded from `index.html` so playback resumes from a clean state — the user taps the creative to replay.

To remove the ad:

```kotlin
feedAd.detach()   // Remove from container (ad data is preserved; can re-attach)
feedAd.destroy()  // Release all resources — do not use the instance after this
```

Call `detach()` when the ad scrolls out of the feed. Call `destroy()` when the host screen is destroyed or the ad will no longer be used. After `destroy()` the instance cannot be reused — call `load()` on a new instance instead.

## User & Content Context

Provide optional context to improve ad targeting. Call after `initialize()` and update whenever relevant state changes.

```kotlin
// Set user identity and subscription context
NovvySDK.setUserContext(
    NovvyUserContext(
        userId      = "publisher-user-id",
        hashedEmail = NovvySDK.helperHashEmail("user@example.com"),
        isPaidUser  = true,
        age         = 28
    )
)

// Set content context (series / episode the user is watching)
NovvySDK.setContentContext(
    NovvyContentContext(
        seriesName    = "Breaking Bad",
        episodeNumber = 3,
        url           = "https://example.com/episode/3"
    )
)

// Incrementally update one or more fields (null values preserve existing data)
NovvySDK.updateUserContext(isPaidUser = false)
NovvySDK.updateContentContext(episodeNumber = 4, url = "https://example.com/episode/4")
```

`userId` is the publisher-side user identifier sent as `user.user_id`. `helperHashEmail()` applies SHA-256 only — it does **not** trim or lowercase, so normalise the input yourself before calling it (e.g. `email.trim().lowercase()`). `isPaidUser` is sent as `user.is_paid_user`. `age` is an **opaque bucket code** defined by your team — it is not the user's literal age in years; agree the bucket scheme with the Novvy team and pass the corresponding integer. All four fields are optional — pass any combination or none.

`setUserContext` and `setContentContext` **replace** the entire object — any field you leave `null` clears the previously stored value. Use `updateUserContext` / `updateContentContext` to merge individual fields while preserving the rest.

Context is read lazily when each ad's `load()` builds a bid request, so changes only take effect for subsequent `load()` calls.

## Error Handling

`onAdFailedToLoad` receives a `NovvyError` (subclass of `Throwable`):

| Error | Description |
|-------|-------------|
| `NovvyError.NoFill` | No ad available for this request |
| `NovvyError.NoBid` | Bid server returned no bid |
| `NovvyError.Timeout` | Request timed out |
| `NovvyError.InvalidRequest` | Malformed request |
| `NovvyError.NoData` | Empty response from server |
| `NovvyError.EncodingFailed` | Request/response serialization error |
| `NovvyError.InternalError(details)` | Internal error with detail message |

```kotlin
override fun onAdFailedToLoad(ad: NovvyInterstitialAd, error: Throwable) {
    when (error) {
        is NovvyError.NoFill  -> { /* no inventory, try later */ }
        is NovvyError.Timeout -> { /* retry with backoff */ }
        else                  -> { /* log error.message */ }
    }
}
```

## AdMob Mediation

The SDK ships a Google AdMob mediation adapter that supports **Interstitial** and **Rewarded** ad formats. Banner and Feed ads are not routed through AdMob mediation.

### Prerequisites

- Your app already depends on `play-services-ads` (AdMob). The Novvy adapter links against it at compile time and does not bundle it.
- `NovvySDK.initialize()` must be called in `Application.onCreate()` **before** AdMob loads any ads. The adapter does not call `initialize()` itself.

### AdMob Console Setup

For each ad unit you want Novvy to fill:

1. Open the ad unit in the AdMob dashboard and go to **Mediation → Add custom event**.
2. Set **Class name** to:
   ```
   ai.novvy.android.sdk.mediation.NovvyMediationAdapter
   ```
3. Set **Parameter** to your Novvy **Ad Unit ID** (the same value you pass to `NovvyInterstitialAd` / `NovvyRewardedAd`).
4. Save and publish. AdMob will invoke the adapter automatically during waterfall or bidding — no additional code changes are needed in your app.
