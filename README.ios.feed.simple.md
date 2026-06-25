# Novvy Ads iOS SDK — Feed Ads

### 一、环境要求

- iOS 13.0+
- Xcode 15+
- Swift 5.0+

### 二、安装

在 `Podfile` 中添加：

```ruby
pod 'NovvyAds', :podspec => 'https://raw.githubusercontent.com/NovvyAI/novvy-ads-cocoapods/v1.0.0-beta.14/NovvyAds.podspec'
```

然后执行：

```bash
pod install
```

升级时将 URL 中的 `v1.0.0-beta.14` 替换为新版本号即可。

---

### 三、完整一集流程

```
开始播放
        ↓
loadAdOnVideoStart         →  后台预加载（不阻塞）
        ↓
播放正片
        ↓
用户准备划走               →  displayAdOnVideoScroll
        ↓
        ├── 有广告  →  SDK 自动挂载到 container，广告展示
        │                       ↓
        │           用户划入广告页  →  ad.setPlaying(true)
        │                       ↓
        │           用户划走广告页  →  ad.setPlaying(false)
        │                       ↓
        │                   进入下一集
        │                       ↓
        │           需要释放资源时  →  ad.destroy()
        │
        └── 无广告  →  直接进入下一集
```

---

### 四、初始化 SDK

```swift
// AppDelegate.swift 或任意负责初始化的类
class AppDelegate: UIResponder, UIApplicationDelegate, NovvyAdObserver {

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        NovvySDK.shared.initialize(
            appId:    "your_app_id",
            endpoint: "https://bidding.novvy.ai",
            apiKey:   "your_api_key",
            userContext: NovvyUserContext(
                userId:               "user_123",
                hashedEmail:          NovvySDK.helperHashEmail("user@example.com"),
                isPaidUser:           true,       // IAP/IAA 付费用户标识
                age:                  2,          // 由宿主团队定义的年龄分段编码
                gender:               "female",   // "male" / "female" / "unknown"
                dramaWatchHistory:    [
                    NovvyDramaRecord(dramaTitle: "Drama Title 1", seriesId: "S1", lastWatchedEpisode: 5, unlockedEpisode: 8),
                    NovvyDramaRecord(dramaTitle: "Drama Title 2", seriesId: "S2", lastWatchedEpisode: 3, unlockedEpisode: 3),
                ],
                maximumAdsInOneDrama: 10          // 单部剧最多展示广告次数
            ),
            dramaObserver: self
        )
        return true
    }

    // MARK: - NovvyAdObserver
    func onNovvyAdEvent(_ event: NovvyAdEvent) {
        switch event {
        case .load(let adUnitId, let adType, let seriesId, let episodeIndex):
            print("[NovvyAd] 广告请求已发出 | 广告位=\(adUnitId) | 类型=\(adType) | 剧集=\(seriesId) 第\(episodeIndex)集")
        case .failed(let adUnitId, let adType, let seriesId, let episodeIndex, let error):
            print("[NovvyAd] 广告加载失败 | 广告位=\(adUnitId) | 类型=\(adType) | 剧集=\(seriesId) 第\(episodeIndex)集 | 原因=\(error.localizedDescription)")
        case .prepared(let adUnitId, let adType, let seriesId, let episodeIndex):
            print("[NovvyAd] 广告已就绪 | 广告位=\(adUnitId) | 类型=\(adType) | 剧集=\(seriesId) 第\(episodeIndex)集")
        case .show(let adUnitId, let adType, let seriesId, let episodeIndex):
            print("[NovvyAd] 广告已展示 | 广告位=\(adUnitId) | 类型=\(adType) | 剧集=\(seriesId) 第\(episodeIndex)集")
        case .destroy(let adUnitId, let adType, let seriesId, let episodeIndex):
            print("[NovvyAd] 广告已销毁 | 广告位=\(adUnitId) | 类型=\(adType) | 剧集=\(seriesId) 第\(episodeIndex)集")
        }
    }
}
```

`NovvyAdEvent.failed` 的 `error` 字段类型为 [`NovvyError`](ios-references/types.md#novvyerror)。

[`NovvyUserContext`](ios-references/types.md#novvyusercontext) 与 [`NovvyDramaRecord`](ios-references/types.md#novvydramarecord) 的字段说明详见类型参考。

---

### 五、片头调用 — `loadAdOnVideoStart`

每一集**开始播放时**调用，用于异步预加载广告并向服务端传递当前剧集上下文。此调用立即返回，不阻塞播放。

```swift
NovvySDK.shared.loadAdOnVideoStart(
    adUnitId:     "your_ad_unit_id",
    seriesId:     "S1",
    episodeIndex: 1,
    title:        "your_drama_title",
    adType:       AdType.feed
)
```

💡 **SDK 会做什么？**
- 立即返回，不阻塞当前播放
- 根据传入参数，包括 `seriesId` / `episodeIndex` 等上下文向服务端发起广告请求
- 根据广告类型，会自行预加载广告资源

---

### 六、视频流滑动时调用 — `displayAdOnVideoScroll`

用户准备切换视频时调用，同步返回已就绪的广告对象。

```swift
let ad: NovvyAd? = NovvySDK.shared.displayAdOnVideoScroll(
    adUnitId:     "your_ad_unit_id",
    seriesId:     "S1",
    episodeIndex: 1,
    adType:       AdType.feed,
    container:    container,   // 可选；传入时 SDK 内部自动 attach，不传则需手动调 ad.attach(to:)
    onAdPlaybackTick: { novvyAd, remainingSeconds in
        // remainingSeconds > 0：广告锁定中，应禁止划走
        // remainingSeconds == 0：倒计时结束，可划走
    }
)

// 如果没有广告，宿主可直接进入下一集
if ad == nil {
    // 直接进入下一集
}

// container 未提前传入时，在 container 就绪后手动挂载
ad?.attach(to: container)

// 视频划走，切入广告页时调用（触发曝光 + 开始倒计时）
ad?.setPlaying(true)

// 广告倒计时结束，用户划走，切换到下一集时调用（停止倒计时，停止广告）
ad?.setPlaying(false)

// 不再需要此广告时，或者其容器需要释放时，调用释放资源
ad?.destroy()
```

💡 **SDK 会做什么？**
- 同步返回，不发起任何网络请求
- 检查该集广告是否已就绪
- 已就绪 → 返回广告对象，SDK 内部自动将广告渲染到 `container`
- 未就绪 → 返回 nil，宿主直接进入下一集
- 如无需使用，可调用 `ad.destroy()` 销毁，完全释放资源
