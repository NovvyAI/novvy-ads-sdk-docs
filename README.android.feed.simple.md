# Novvy Ads Android SDK — Feed Ads

### 一、环境要求

- Android minSdk 24 (Android 7.0+)
- Kotlin / Java 11+

### 二、安装

SDK 发布于 Maven Central，在 `build.gradle.kts` 中添加依赖：

```kotlin
dependencies {
    implementation("ai.novvy.android:sdk:1.0.0-beta23")
}
```

在 `AndroidManifest.xml` 中添加所需权限：

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

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

```kotlin
NovvySDK.initialize(
    context  = applicationContext, // Android 全局应用 Context（即 Application 实例），在 Activity 中可直接用 applicationContext 获取
    appId    = "your_app_id",
    endpoint = "https://bidding.novvy.ai",
    apiKey   = "your_api_key",
    userContext = NovvyUserContext(
        userId               = "user_123",
        hashedEmail          = NovvySDK.helperHashEmail("user@example.com"),
        isPaidUser           = true,          // IAP/IAA 付费用户标识
        age                  = 2,            // 由宿主团队定义的年龄分段编码
        gender               = "female",      // "male" / "female" / "unknown"
        dramaWatchHistory    = listOf(
            NovvyDramaRecord(dramaTitle = "Drama Title 1", seriesId = "S1", lastWatchedEpisode = 5, unlockedEpisode = 8),
            NovvyDramaRecord(dramaTitle = "Drama Title 2", seriesId = "S2", lastWatchedEpisode = 3, unlockedEpisode = 3),
        ),  // 已观看剧集列表
        maximumAdsInOneDrama = 10             // 单部剧最多展示广告次数
    ),
    dramaObserver = object : NovvyAdObserver {
        override fun onNovvyAdEvent(event: NovvyAdEvent) {
            when (event) {
                is NovvyAdEvent.Load ->
                    log("[NovvyAd] 广告请求已发出 | 广告位=${event.adUnitId} | 类型=${event.adType} | 剧集=${event.seriesId} 第${event.episodeIndex}集")
                is NovvyAdEvent.Failed ->
                    log("[NovvyAd] 广告加载失败 | 广告位=${event.adUnitId} | 类型=${event.adType} | 剧集=${event.seriesId} 第${event.episodeIndex}集 | 原因=${event.error.message}")
                is NovvyAdEvent.Prepared ->
                    log("[NovvyAd] 广告已就绪 | 广告位=${event.adUnitId} | 类型=${event.adType} | 剧集=${event.seriesId} 第${event.episodeIndex}集")
                is NovvyAdEvent.Show ->
                    log("[NovvyAd] 广告已展示 | 广告位=${event.adUnitId} | 类型=${event.adType} | 剧集=${event.seriesId} 第${event.episodeIndex}集")
                is NovvyAdEvent.Destroy ->
                    log("[NovvyAd] 广告已销毁 | 广告位=${event.adUnitId} | 类型=${event.adType} | 剧集=${event.seriesId} 第${event.episodeIndex}集")
            }
        }
    }
)
```

`NovvyAdEvent.Failed` 的 `error` 字段类型为 `NovvyError`，可按需进一步 `when` 匹配：

| 类型 | 含义 |
|------|------|
| `NovvyError.NoFill` | 服务端无广告填充，属正常情况 |
| `NovvyError.Timeout` | 请求超时 |
| `NovvyError.NoBid` | 竞价无结果 |
| `NovvyError.InvalidRequest` | 请求参数有误 |
| `NovvyError.InternalError` | SDK 内部错误，附带 `details: String` 字段说明具体原因 |
| `NovvyError.NoData` | 响应数据为空 |
| `NovvyError.EncodingFailed` | 数据编解码失败 |

`NovvyUserContext` 字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `userId` | `String?` | 用户唯一标识 |
| `hashedEmail` | `String?` | 邮箱 SHA-256 哈希，可用 `NovvySDK.helperHashEmail()` 生成 |
| `isPaidUser` | `Boolean?` | 是否为付费用户（IAP/IAA） |
| `age` | `Int?` | 由宿主团队定义的年龄分段编码 |
| `gender` | `String?` | 性别，`"male"` / `"female"` / `"unknown"` |
| `dramaWatchHistory` | `List<NovvyDramaRecord>?` | 已观看剧集记录列表，用于广告定向 |
| `maximumAdsInOneDrama` | `Int?` | 单部剧最多展示广告次数，不传则由服务端统一控制 |

`NovvyDramaRecord` 字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dramaTitle` | `String` | 剧集title |
| `seriesId` | `String` | 剧集唯一标识 |
| `lastWatchedEpisode` | `Int` | 上次观看到第几集 |
| `unlockedEpisode` | `Int` | 已解锁到第几集 |

---

### 五、片头调用 — `loadAdOnVideoStart`

每一集**开始播放时**调用，用于异步预加载广告并向服务端传递当前剧集上下文。此调用立即返回，不阻塞播放。

```kotlin
NovvySDK.loadAdOnVideoStart(
    adUnitId     = "your_ad_unit_id",
    seriesId     = "S1",
    episodeIndex = 1,
    title        = "your_drama_title",
    adType       = AdType.FEED
)
```

💡 **SDK 会做什么？**
- 立即返回，不阻塞当前播放
- 根据传入参数，包括 `seriesId` / `episodeIndex` 等上下文向服务端发起广告请求
- 根据广告类型，会自行预加载广告资源

---

### 六、视频流滑动时调用 — `displayAdOnVideoScroll`

用户准备切换视频时调用，同步返回已就绪的广告对象。

```kotlin
val ad: NovvyAd? = NovvySDK.displayAdOnVideoScroll(
    adUnitId     = "your_ad_unit_id",
    seriesId     = "S1",
    episodeIndex = 1,
    adType       = AdType.FEED,
    container    = container,   // 可选；传入时 SDK 内部自动 attach，不传则需手动调 ad.attach(container)
    onAdPlaybackTick = { novvyAd, remainingSeconds ->
        // remainingSeconds > 0：广告锁定中，应禁止划走
        // remainingSeconds == 0：倒计时结束，可划走
        log(tag, "Feed Ad tick: ${remainingSeconds}s remaining")
    }
)

// 如果没有广告，宿主可直接进入下一集
if (ad == null) {
    // 直接进入下一集
}

// container 未提前传入时，在 container 就绪后手动挂载
ad?.attach(container)

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
- 未就绪 → 返回 null，宿主直接进入下一集
- 如无需使用，可调用 `ad.destroy()` 销毁，完全释放资源
