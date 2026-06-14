# Novvy Ads Android SDK — Feed Ads

Integration guide for Feed Ads. For other ad formats see [README.android.md](README.android.md).

## Requirements

- Android **minSdk 24** (Android 7.0+)
- Kotlin / Java 11+

## Installation

The SDK is published on Maven Central. Add the dependency in your app's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("ai.novvy.android:sdk:1.0.0-beta23")
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

---

## Feed Ads

Feed ads are designed for vertical feed/pager integration. The SDK supports two creative formats and selects the renderer automatically based on the server response — host code is identical for both:

- **Image creative** — renders natively with a hero image and a product info overlay (thumbnail, title, price, CTA).
- **H5 zip creative** — renders an interactive H5 bundle (HTML/CSS/JS, optional video) in a WebView. The zip is downloaded once per `material_url` and cached on disk; subsequent loads of the same creative reuse the cache.

Both formats share the same playback countdown, swipe-to-skip overlay, click handling, and listener callbacks.

### Step 1 — Create and load

```kotlin
val feedAd = NovvyFeedAd(
    adUnitId               = "your-ad-unit-id",
    durationSeconds        = 4,   // Optional: countdown duration (default 4, minimum 3)
    useDefaultSwipeOverlay = true // Optional: set false to supply a custom swipe UI
)

feedAd.delegate = object : NovvyFeedAdListener {
    override fun onAdLoaded(ad: NovvyFeedAd)                         { }
    override fun onAdFailedToLoad(ad: NovvyFeedAd, error: Throwable) { }
    override fun onAdImpression(ad: NovvyFeedAd)                     { }
    override fun onAdClicked(ad: NovvyFeedAd)                        { }
    override fun onAdPlaybackTick(ad: NovvyFeedAd, remainingSeconds: Int) {
        // Fires every second during playback.
        // remainingSeconds == 0 means the countdown has finished.
        // Use this to drive a custom overlay when useDefaultSwipeOverlay = false.
    }
}

feedAd.load()
```

When `useDefaultSwipeOverlay = false` the SDK skips its built-in "Swipe to skip" overlay; drive your own UI from `onAdPlaybackTick` instead.

`load()` accepts an optional `episodeNumber` parameter that overrides the episode number in the bid request for this specific load call, without changing the global content context:

```kotlin
feedAd.load(episodeNumber = 4)
```

Load the ad ahead of time (e.g. when the page opens) so it is ready before the user scrolls to the insertion point.

### Step 2 — Attach to container

Once `onAdLoaded` fires (or `feedAd.isReady == true`), attach the ad to any `ViewGroup`:

```kotlin
feedAd.attach(container = binding.feedAdContainer)
```

### Step 3 — Control playback

Call `setPlaying()` as the ad item scrolls in and out of view. A common threshold is 50% visible:

```kotlin
feedAd.setPlaying(true)   // Fires impression (once) and starts countdown
feedAd.setPlaying(false)  // Stops and resets countdown
```

For H5 creatives, `setPlaying(false)` also pauses any `<video>`/`<audio>` elements. When the user swipes back (next `setPlaying(true)` after a deactivation), the H5 page is reloaded from `index.html` so playback always resumes from a clean state.

### Step 4 — Dispose

```kotlin
feedAd.detach()   // Remove from container (ad data is preserved; can re-attach)
feedAd.destroy()  // Release all resources — do not use the instance after this
```

Call `detach()` when the ad scrolls out of the feed. Call `destroy()` when the host screen is destroyed or the ad will no longer be used. After `destroy()` the instance cannot be reused — call `load()` on a new instance instead.

---

## User & Content Context

Providing context improves targeting. Call after `initialize()` and update whenever the user navigates to new content.

```kotlin
// Set user identity
NovvySDK.setUserContext(
    NovvyUserContext(
        userId      = "publisher-user-id",
        hashedEmail = NovvySDK.helperHashEmail("user@example.com"),
        isPaidUser  = true,
        age         = 1  // opaque bucket code defined by your team — not a literal age
    )
)

// Set content context — update this before each load() to reflect the current episode
NovvySDK.setContentContext(
    NovvyContentContext(
        seriesName    = "Breaking Bad",
        episodeNumber = 3,
        url           = "https://example.com/episode/3"
    )
)

// Incrementally update one or more fields (null values preserve existing data)
NovvySDK.updateContentContext(episodeNumber = 4, url = "https://example.com/episode/4")
```

`helperHashEmail()` applies SHA-256 only — it does **not** trim or lowercase, so normalise the input yourself before calling it (e.g. `email.trim().lowercase()`).

`setUserContext` and `setContentContext` **replace** the entire object — any field you leave `null` clears the previously stored value. Use `updateUserContext` / `updateContentContext` to merge individual fields while preserving the rest.

Context is read when `load()` builds the bid request. For per-load episode overrides without mutating the global context, use `feedAd.load(episodeNumber = N)` instead.

---

## Error Handling

`onAdFailedToLoad` receives a `NovvyError` (subclass of `Throwable`):

| Error | Description |
|-------|-------------|
| `NovvyError.NoFill` | No ad available for this request |
| `NovvyError.NoBid` | Bid server returned no bid |
| `NovvyError.Timeout` | Request timed out |
| `NovvyError.NoData` | Empty response from server |
| `NovvyError.InternalError(details)` | Internal error with detail message |

```kotlin
override fun onAdFailedToLoad(ad: NovvyFeedAd, error: Throwable) {
    when (error) {
        is NovvyError.NoFill  -> { /* no inventory, try later */ }
        is NovvyError.Timeout -> { /* retry with backoff */ }
        else                  -> { /* log error.message */ }
    }
}
```
