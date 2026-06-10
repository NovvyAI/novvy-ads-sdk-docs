# Novvy Ads iOS SDK — Feed Ads

Integration guide for Feed Ads. For other ad formats see [README.ios.md](README.ios.md).

## Requirements

- iOS **13.0+**
- Xcode 15+
- Swift 5.0+

## Installation

Add the following to your `Podfile`:

```ruby
pod 'NovvyAds', :podspec => 'https://raw.githubusercontent.com/NovvyAI/novvy-ads-cocoapods/v1.0.0-beta.12/NovvyAds.podspec'
```

Then run:

```bash
pod install
```

To upgrade, replace `v1.0.0-beta.12` in the URL with the new version tag.

## Initialization

Initialize once, as early as possible (e.g. in `application(_:didFinishLaunchingWithOptions:)`):

```swift
NovvySDK.shared.initialize(
    appId:    "your-app-id",
    endpoint: "https://your-bid-server.com",
    apiKey:   "your-api-key"
) { success in
    // Optional: called when initialization completes
}
```

`initialize()` may be called again at any time — for example, after a user logs in or out — to update credentials and trigger a fresh server handshake.

---

## Feed Ads

Feed ads are designed for vertical feed/pager integration. The SDK supports two creative formats and selects the renderer automatically based on the server response — host code is identical for both:

- **Image creative** — renders natively with a background material image and a product info overlay (thumbnail, title, price, CTA).
- **H5 zip creative** — renders an interactive H5 bundle (HTML/CSS/JS, optional video) in a WebView. The zip is downloaded once per `material_url` and cached on disk; subsequent loads of the same creative reuse the cache.

Both formats share the same playback countdown, swipe-to-skip overlay, click handling, and delegate callbacks.

### Step 1 — Create and load

```swift
let feedAd = NovvyFeedAd(
    adUnitId:               "your-ad-unit-id",
    durationSeconds:        4,    // Optional: countdown duration (default 4, minimum 3)
    useDefaultSwipeOverlay: true  // Optional: set false to supply a custom swipe UI
)
feedAd.delegate = self

extension YourClass: NovvyFeedAdDelegate {
    func feedAdDidLoad(_ ad: NovvyFeedAd)                               { }
    func feedAd(_ ad: NovvyFeedAd, didFailToLoadWithError error: NovvyError) { }
    func feedAdDidRecordImpression(_ ad: NovvyFeedAd)                   { }
    func feedAdDidClick(_ ad: NovvyFeedAd)                              { }
    func feedAd(_ ad: NovvyFeedAd, didTickWithRemainingSeconds seconds: Int) {
        // Fires every second during playback.
        // seconds == 0 means the countdown has finished.
        // Use this to drive a custom overlay when useDefaultSwipeOverlay = false.
    }
}

feedAd.bidFloor = 1.50  // Optional: minimum eCPM floor
feedAd.load()
```

When `useDefaultSwipeOverlay = false` the SDK skips its built-in "Swipe to skip" overlay; drive your own UI from `didTickWithRemainingSeconds` instead.

`load()` accepts an optional `episodeNumber` parameter that overrides the episode number in the bid request for this specific load call, without changing the global content context:

```swift
feedAd.load(episodeNumber: 4)
```

Load the ad ahead of time (e.g. when the screen opens) so it is ready before the user scrolls to the insertion point.

### Step 2 — Attach to container

Once `feedAdDidLoad` fires (or `feedAd.isReady == true`), attach the ad to any `UIView` container:

```swift
feedAd.attach(to: feedAdContainerView)
```

### Step 3 — Control playback

Call `setPlaying()` as the ad item scrolls in and out of view. A common threshold is 50% visible:

```swift
feedAd.setPlaying(true)   // Fires impression (once) and starts countdown
feedAd.setPlaying(false)  // Stops and resets countdown
```

For H5 creatives, `setPlaying(false)` also pauses any `<video>`/`<audio>` elements. When the user swipes back (next `setPlaying(true)` after a deactivation), the H5 page is reloaded from `index.html` so playback always resumes from a clean state.

### Step 4 — Dispose

```swift
feedAd.detach()   // Remove from container (ad data is preserved; can re-attach)
feedAd.destroy()  // Release all resources — do not use the instance after this
```

Call `detach()` when the ad scrolls out of the feed. Call `destroy()` when the host screen is destroyed or the ad will no longer be used. After `destroy()` the instance cannot be reused — call `load()` on a new instance instead.

---

## User & Content Context

Providing context improves targeting. Call after `initialize()` and update whenever the user navigates to new content.

```swift
// Set user identity
NovvySDK.shared.setUserContext(
    NovvyUserContext(
        userId:      "publisher-user-id",
        hashedEmail: NovvySDK.helperHashEmail("user@example.com"),
        isPaidUser:  true,
        age:         1  // opaque bucket code defined by your team — not a literal age
    )
)

// Set content context — update this before each load() to reflect the current episode
NovvySDK.shared.setContentContext(
    NovvyContentContext(
        seriesName:    "Breaking Bad",
        episodeNumber: 3,
        url:           "https://example.com/episode/3"
    )
)

// Incrementally update one or more fields (nil values preserve existing data)
NovvySDK.shared.updateContentContext(episodeNumber: 4, url: "https://example.com/episode/4")
```

`helperHashEmail()` applies SHA-256 only — it does **not** trim or lowercase, so normalise the input yourself before calling it (e.g. `email.trimmingCharacters(in: .whitespaces).lowercased()`).

`setUserContext` and `setContentContext` **replace** the entire object — any field you leave `nil` clears the previously stored value. Use `updateUserContext` / `updateContentContext` to merge individual fields while preserving the rest.

Context is read when `load()` builds the bid request. For per-load episode overrides without mutating the global context, use `feedAd.load(episodeNumber:)` instead.

---

## Error Handling

`didFailToLoadWithError` receives a `NovvyError`:

| Error | Description |
|-------|-------------|
| `.noFill` | No ad available for this request |
| `.noBid` | Bid server returned no bid |
| `.timeout` | Request timed out |
| `.invalidRequest` | Malformed request |
| `.noData` | Empty response from server |
| `.encodingFailed` | Request/response serialization error |
| `.internalError(details)` | Internal error with detail message |

```swift
func feedAd(_ ad: NovvyFeedAd, didFailToLoadWithError error: NovvyError) {
    switch error {
    case .noFill:               break  // no inventory, try later
    case .timeout:              break  // retry with backoff
    case .internalError(let s): print(s)
    default:                    print(error.localizedDescription)
    }
}
```
