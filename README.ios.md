# Novvy Ads iOS SDK

iOS SDK for programmatic ad delivery, supporting interstitial, rewarded, banner, and feed ad formats via real-time bidding (RTB).

## Features

- **Interstitial Ads** — Full-screen ads shown at natural transition points (e.g. between episodes)
- **Rewarded Ads** — Full-screen ads that grant a reward when the user closes the ad
- **Banner Ads** — 260×90pt native views with server-driven show/hide scheduling tied to video playback position
- **Feed Ads** — Native image or H5-zip ads inserted into scrollable feeds with visibility-driven playback countdown

## Quick Start

1. [Install the SDK](#installation) via CocoaPods
2. [Initialize](#initialization) once in `application(_:didFinishLaunchingWithOptions:)`
3. Load and show ads — [Interstitial](#interstitial-ads) | [Rewarded](#rewarded-ads) | [Banner](#banner-ads) | [Feed](#feed-ads)
4. (Optional) [Set user and content context](#user--content-context) to improve targeting

---

## Requirements

- iOS **13.0+**
- Xcode 15+
- Swift 5.0+

## Installation

### Direct Integration (without AdMob)

Add the following to your `Podfile`:

```ruby
pod 'NovvyAds', :podspec => 'https://raw.githubusercontent.com/NovvyAI/novvy-ads-cocoapods/v1.0.0-beta.14/NovvyAds.podspec'
```

Then run:

```bash
pod install
```

To upgrade, replace `v1.0.0-beta.14` in both URLs with the new version tag.

### With AdMob Mediation

Add both pods to your `Podfile`:

```ruby
pod 'NovvyAds', :podspec => 'https://raw.githubusercontent.com/NovvyAI/novvy-ads-cocoapods/v1.0.0-beta.14/NovvyAds.podspec'
pod 'NovvyAdsAdMob', :podspec => 'https://raw.githubusercontent.com/NovvyAI/novvy-ads-cocoapods/v1.0.0-beta.14/NovvyAdsAdMob.podspec'
```

Then run:

```bash
pod install
```

#### AdMob Configuration

1. **Info.plist** — add your AdMob App ID:

   ```xml
   <key>GADApplicationIdentifier</key>
   <string>your-admob-app-id</string>
   ```

2. **AppDelegate** — initialize both SDKs. `NovvySDK.shared.initialize()` **must** be called even when using AdMob mediation; the adapter calls `load()` directly without checking initialization state, so any ad request made before initialization completes will fail silently.

   ```swift
   import GoogleMobileAds
   import NovvyAds

   func application(_ application: UIApplication,
                    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
       MobileAds.shared.start()
       NovvySDK.shared.initialize(
           appId:    "your-app-id",
           endpoint: "https://your-bid-server.com",
           apiKey:   "your-api-key"
       )
       return true
   }
   ```

3. **AdMob Console** — add a Custom Event to each ad unit:

   | Field | Value |
   |-------|-------|
   | Class Name | `NovvyAdMobMediationAdapter` |
   | Parameter | Your Novvy Ad Unit ID |

   Supported formats via AdMob mediation: **Interstitial** and **Rewarded** only.
   Banner and Feed ads are not available through AdMob mediation — use direct integration for those.

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

## Interstitial Ads

```swift
let interstitialAd = NovvyInterstitialAd(adUnitId: "your-ad-unit-id")
interstitialAd.delegate = self

extension YourClass: NovvyInterstitialAdDelegate {
    func interstitialAdDidLoad(_ ad: NovvyInterstitialAd) {
        ad.show(from: viewController)
    }
    func interstitialAd(_ ad: NovvyInterstitialAd, didFailToLoadWithError error: NovvyError) { }
    func interstitialAdDidShow(_ ad: NovvyInterstitialAd)   { }
    func interstitialAdDidClose(_ ad: NovvyInterstitialAd)  { }
    func interstitialAdDidClick(_ ad: NovvyInterstitialAd)  { }
}

interstitialAd.load()
```

Check `interstitialAd.isReady` before calling `show(from:)` if you load and display at different points in time.

## Rewarded Ads

```swift
let rewardedAd = NovvyRewardedAd(adUnitId: "your-ad-unit-id")
rewardedAd.rewardedDelegate = self

extension YourClass: NovvyRewardedAdDelegate {
    func rewardedAdDidLoad(_ ad: NovvyRewardedAd) {
        ad.show(from: viewController)
    }
    func rewardedAd(_ ad: NovvyRewardedAd, didFailToLoadWithError error: NovvyError) { }
    func rewardedAdDidShow(_ ad: NovvyRewardedAd)   { }
    func rewardedAdDidClose(_ ad: NovvyRewardedAd)  { }
    func rewardedAdDidClick(_ ad: NovvyRewardedAd)  { }
    func rewardedAd(_ ad: NovvyRewardedAd, didEarnReward reward: NovvyReward) {
        // Grant reward: reward.amount, reward.type
    }
}

rewardedAd.load()
```

`didEarnReward` fires unconditionally when the user closes the ad — the SDK does not distinguish between "watched to completion" and "dismissed early". Grant the reward whenever this callback is received.

## Banner Ads

Banner ads render natively (no WebView) and automatically bind to the container's lifecycle. The server can return multiple ads per request, each with its own display time windows. The SDK supports:

- **Multi-ad selection**: Multiple ads are preloaded in parallel. When the player pauses at a position matching an ad's time window, one eligible ad is randomly selected and displayed.
- **Playback-position windows**: Each ad specifies `playback_positions` (e.g. `["1000,5000", "10000,15000"]`). The banner only appears when the player is paused within a matching window. When playback resumes, the banner hides. If the current position does not fall inside any window, or if `playback_positions` is empty, the banner is not shown.

### Player Adapter

Implement `NovvyPlayerAdapter` to let the SDK monitor playback state:

```swift
class MyPlayerAdapter: NovvyPlayerAdapter {
    var currentPositionMs: Int64 { Int64(player.currentTime * 1000) }
    var isPlaying: Bool { player.isPlaying }
}
```

Both properties are polled at ~500 ms intervals — keep the getters lightweight (no I/O or locks).

- `currentPositionMs` — return the raw player position with no caching or rounding.
- `isPlaying` — return `true` only when the player is **actively playing**. Return `false` during pause, buffering/loading, and seek. The SDK relies on this to decide when to show and hide the banner.

### Load and Attach a Banner

Banner ads follow the same load-then-display pattern as other ad formats:

```swift
let bannerAd = NovvyBannerAd(adUnitId: "your-ad-unit-id")
bannerAd.delegate = self

extension YourClass: NovvyBannerAdDelegate {
    func bannerAdDidLoad(_ ad: NovvyBannerAd)                              { }
    func bannerAd(_ ad: NovvyBannerAd, didFailToLoadWithError error: NovvyError) { }
    func bannerAdDidShow(_ ad: NovvyBannerAd)                              { }
    func bannerAdDidHide(_ ad: NovvyBannerAd)                              { }
    func bannerAdDidRecordImpression(_ ad: NovvyBannerAd)                  { }
    func bannerAdDidClick(_ ad: NovvyBannerAd)                             { }
    func bannerAdDidClose(_ ad: NovvyBannerAd)                             { }
}

bannerAd.load()
```

Once the ad is loaded (i.e. `bannerAdDidLoad` fires or `bannerAd.isReady == true`), attach it to any `UIView` container along with a player adapter:

```swift
bannerAd.attach(to: playerContainerView, player: playerAdapter)
```

The banner has a fixed size of **260×90pt**, left-aligned within the container.

The SDK automatically monitors playback and determines when to show/hide the banner based on the server-configured display mode. To remove the banner manually:

```swift
bannerAd.detach()
```

When the user taps the banner's close button, `bannerAdDidClose` fires and `detach()` is called automatically — no need to call it again in your callback.

## Feed Ads

Feed ads are designed for vertical feed/pager integration. The SDK supports two creative formats and selects the renderer automatically based on the server response — host code is identical for both:

- **Image creative** — renders natively with a background material image and a product info overlay (thumbnail, title, price, CTA).
- **H5 zip creative** — renders an interactive H5 bundle (HTML/CSS/JS, optional video) in a WebView. The zip is downloaded once per `material_url` and cached on disk; subsequent loads of the same creative reuse the cache.

Both formats share the same playback countdown, swipe-to-skip overlay, click handling, and delegate callbacks.

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
        // Update countdown UI; seconds == 0 means completed
    }
}
```

When `useDefaultSwipeOverlay = false` the SDK skips its built-in "Swipe to skip" overlay; drive your own UI from `didTickWithRemainingSeconds` instead.

```swift
feedAd.load()
```

`load()` accepts an optional `episodeNumber` parameter that overrides the episode number in the bid request for this specific load call (takes precedence over `NovvySDK.shared.contentContext`):

```swift
feedAd.load(episodeNumber: 4)
```

Once the ad is loaded, attach it to any `UIView` container:

```swift
feedAd.attach(to: feedAdContainerView)
```

Control playback by notifying the SDK when the ad is visible (e.g. the current page in a pager). A common threshold is 50% visible:

```swift
feedAd.setPlaying(true)   // Fires impression (once) and starts countdown
feedAd.setPlaying(false)  // Stops and resets countdown
```

For H5 creatives, `setPlaying(false)` also pauses any `<video>`/`<audio>` elements. When the user swipes back (next `setPlaying(true)` after a deactivation), the H5 page is reloaded from `index.html` so playback resumes from a clean state.

To remove the ad:

```swift
feedAd.detach()   // Remove from container (ad data is preserved; can re-attach)
feedAd.destroy()  // Release all resources — do not use the instance after this
```

Call `detach()` when the ad scrolls out of the feed. Call `destroy()` when the host screen is destroyed or the ad will no longer be used. After `destroy()` the instance cannot be reused — call `load()` on a new instance instead.

## User & Content Context

Provide optional context to improve ad targeting. Call after `initialize()` and update whenever relevant state changes.

```swift
// Set user identity and subscription context
NovvySDK.shared.setUserContext(
    NovvyUserContext(
        userId:      "publisher-user-id",
        hashedEmail: NovvySDK.helperHashEmail("user@example.com"),
        isPaidUser:  true,
        age:         28
    )
)

// Set content context (series / episode the user is watching)
NovvySDK.shared.setContentContext(
    NovvyContentContext(
        seriesName:    "Breaking Bad",
        episodeNumber: 3,
        url:           "https://example.com/episode/3"
    )
)

// Incrementally update one or more fields (nil values preserve existing data)
NovvySDK.shared.updateUserContext(isPaidUser: false)
NovvySDK.shared.updateContentContext(episodeNumber: 4, url: "https://example.com/episode/4")
```

`userId` is the publisher-side user identifier sent as `user.user_id`. `helperHashEmail()` applies SHA-256 only — it does **not** trim or lowercase, so normalise the input yourself before calling it (e.g. `email.trimmingCharacters(in: .whitespaces).lowercased()`). `isPaidUser` is sent as `user.is_paid_user`. `age` is an **opaque bucket code** defined by your team — it is not the user's literal age in years; agree the bucket scheme with the Novvy team and pass the corresponding integer. All four fields are optional — pass any combination or none.

`setUserContext` and `setContentContext` **replace** the entire object — any field you leave `nil` clears the previously stored value. Use `updateUserContext` / `updateContentContext` to merge individual fields while preserving the rest.

Context is read lazily when each ad's `load()` builds a bid request, so changes only take effect for subsequent `load()` calls.

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
func interstitialAd(_ ad: NovvyInterstitialAd, didFailToLoadWithError error: NovvyError) {
    switch error {
    case .noFill:               break  // no inventory, try later
    case .timeout:              break  // retry with backoff
    case .internalError(let s): print(s)
    default:                    print(error.localizedDescription)
    }
}
```
