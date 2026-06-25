# Novvy Ads Android SDK — 类型参考

## NovvyError

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

## NovvyUserContext

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

## NovvyDramaRecord

`NovvyDramaRecord` 字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dramaTitle` | `String` | 剧集title |
| `seriesId` | `String` | 剧集唯一标识 |
| `lastWatchedEpisode` | `Int` | 上次观看到第几集 |
| `unlockedEpisode` | `Int` | 已解锁到第几集 |
