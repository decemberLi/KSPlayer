### 目的

梳理从 SwiftUI 入口 `KSVideoPlayerView` 传入的参数（例如 `url`、`options`、播放控制值）如何一路下传到播放内核 `KSMEPlayer`，以及底层播放状态如何回传到上层 UI。便于排查“值从哪来、到哪去、在哪里被消费/回调”。

### 总览路径

- `KSVideoPlayerView` (SwiftUI)
  -> `KSVideoPlayer` (`UIViewRepresentable` 桥接)
  -> `KSVideoPlayer.Coordinator`
  -> `KSPlayerLayer`
  -> `MediaPlayerProtocol` 实现（`KSAVPlayer` 或 `KSMEPlayer`）
  -> `KSMEPlayer` 内部（`MEPlayerItem` / `AudioOutput` / `VideoOutput`）

---

## 1) SwiftUI 入口：把 url/options 与 UI 控制值往下传

- 文件：`Sources/KSPlayer/SwiftUI/KSVideoPlayerView.swift`
- 关键点：
  - `KSVideoPlayerView` 中将 `url`、`options` 传给 `KSVideoPlayer`。
  - 通过链式回调 `.onStateChanged` / `.onBufferChanged` 监听底层状态与统计信息。
  - 进度滑块 `Slider` 绑定 `Coordinator.timemodel`，结束拖动时调用 `Coordinator.seek` 触发向下 seek。

示例（简化）：

```swift
// KSVideoPlayerView.swift
KSVideoPlayer(coordinator: playerCoordinator, url: url, options: options)
  .onStateChanged { playerLayer, state in /* 同步标题/控制显隐 */ }
  .onBufferChanged { bufferedCount, consumeTime in /* 调试输出 */ }

// 进度条 -> seek
Slider(value: Binding( /* currentTime 绑定 */ )) { editing in
  if editing { playerCoordinator.playerLayer?.pause() }
  else { playerCoordinator.seek(time: TimeInterval(model.currentTime)) }
}
```

---

## 2) 桥接层：`KSVideoPlayer` 与 `Coordinator`

- 文件：`Sources/KSPlayer/AVPlayer/KSVideoPlayer.swift`
- 关键点：
  - `KSVideoPlayer` 作为 `UIViewRepresentable`，在 `makeUIView`/`makeNSView` 内调用 `context.coordinator.makeView(url:options:)`。
  - `Coordinator` 持有 `playerLayer`，是所有“上层值 -> 底层播放器”的映射枢纽：
    - `isMuted`、`playbackVolume`、`playbackRate`、`isScaleAspectFill` 等 `@Published` 值，直接写入 `playerLayer?.player.xxx`。
    - `seek(time:)` -> `playerLayer?.seek(...)`。
    - `makeView(url:options:)` 内部创建或复用 `KSPlayerLayer` 并设置 delegate。

示例（简化）：

```swift
// KSVideoPlayer.makeUIView → Coordinator.makeView
context.coordinator.makeView(url: url, options: options)

// Coordinator 属性写回底层播放器
@Published var playbackRate: Float = 1.0 { didSet { playerLayer?.player.playbackRate = playbackRate } }
@Published var isMuted: Bool = false { didSet { playerLayer?.player.isMuted = isMuted } }
```

---

## 3) 播放承载层：`KSPlayerLayer` 选择并持有底层播放内核

- 文件：`Sources/KSPlayer/AVPlayer/KSPlayerLayer.swift`
- 关键点：
  - `KSPlayerLayer` 负责：
    - 根据策略确定使用 `KSAVPlayer` 还是 `KSMEPlayer`：
      - AirPlay 外接：强制 `KSAVPlayer`；
      - `options.display != .plane`（如 AR）：强制 `KSMEPlayer`；
      - 否则使用 `KSOptions.firstPlayerType`（默认 `KSAVPlayer`）。
    - 管理 `url`/`options` 变化：相同类型播放器则 `replace(url:options:)`，否则重建播放器实例。
    - 封装 `play/pause/seek`，并维护 `KSPlayerState`。
    - 通过 `Timer` 每 0.1s 将 `currentPlaybackTime/duration` 回调给 delegate。

示例（简化）：

```swift
// 选择播放内核
let type: MediaPlayerProtocol.Type = (options.display != .plane) ? KSMEPlayer.self : KSOptions.firstPlayerType
player = type.init(url: url, options: options)

// url 变更时复用或重建
player.replace(url: url, options: options) // 同内核
player = type.init(url: url, options: options) // 切换内核
```

---

## 4) 播放内核：`KSMEPlayer` 接收并消费上层值

- 文件：`Sources/KSPlayer/MEPlayer/KSMEPlayer.swift`
- 关键点：
  - 构造时建立：`MEPlayerItem`（解封装/解码）、`AudioOutput`、`VideoOutput`；并互相绑定。
  - 对上层暴露的“可读/可写”值：
    - `view`（用于承载图像输出）、`playbackRate`、`playbackVolume`、`isMuted`、`contentMode`；
    - `currentPlaybackTime`/`duration`/`seekable`/`dynamicInfo`；
    - `tracks(mediaType:)` 与 `select(track:)`；
    - `prepareToPlay/play/pause/seek/replace/shutdown`。
  - 通过 `MediaPlayerDelegate` 将“准备就绪/加载状态/播放结束/错误”等上报到 `KSPlayerLayer`。

示例（简化）：

```swift
public required init(url: URL, options: KSOptions) {
  audioOutput = KSOptions.audioPlayerType.init()
  playerItem = MEPlayerItem(url: url, options: options)
  videoOutput = options.videoDisable ? nil : KSOptions.videoPlayerType.init(options: options)
  // 绑定渲染源与帧回调
}

// 对上层的读写
public var playbackVolume: Float { get { audioOutput.volume } set { audioOutput.volume = newValue } }
public func seek(time: TimeInterval, completion: @escaping (Bool)->Void) { /* 委托 playerItem.seek */ }
```

---

## 5) 回调链：底层如何回传到 SwiftUI

- `KSMEPlayer` → `KSPlayerLayer`（实现 `MediaPlayerDelegate`）

  - `readyToPlay`：切到 `readyToPlay`，必要时自动播放/处理首帧 seek；同步系统 `NowPlayingInfo`。
  - `changeLoadState`：在 `buffering`/`bufferFinished` 之间切换，统计首开耗时。
  - `finish(error:)`：错误则尝试 `KSOptions.secondPlayerType`（默认 `KSMEPlayer`），否则置 `playedToTheEnd`。

- `KSPlayerLayer` → `Coordinator`（实现 `KSPlayerLayerDelegate`）
  - 定时器每 0.1s 上报 `currentTime/totalTime`，`Coordinator` 同步到 `timemodel`（驱动进度 UI）。
  - `onStateChanged/onBufferChanged/onFinish/onPlay` 透传给 SwiftUI 侧的链式回调。

---

## 6) 常见“取值/设值”映射速查

- 音量/静音：

  - SwiftUI 改 `Coordinator.playbackVolume` / `Coordinator.isMuted`
  - → `KSPlayerLayer.player.playbackVolume` / `isMuted`
  - → `KSMEPlayer.playbackVolume` / `isMuted`（最终作用于 `AudioOutput`）

- 倍速：

  - `Coordinator.playbackRate` → `player.playbackRate`
  - → `KSMEPlayer.playbackRate`（必要时更新 `options.audioFilters`）

- 填充模式：

  - `Coordinator.isScaleAspectFill` → `player.contentMode = .scaleAspectFill/.scaleAspectFit`（作用于 `VideoOutput.view`）

- Seek：

  - SwiftUI 滑块 → `Coordinator.seek` → `KSPlayerLayer.seek`
  - → `KSMEPlayer.seek` → `MEPlayerItem.seek`（并刷新 A/V 输出与显示时间基）

- 进度显示：

  - `KSMEPlayer.currentPlaybackTime/duration`
  - → `KSPlayerLayer.timer` → `Coordinator.timemodel` → SwiftUI 进度 UI

- 轨道读取/切换：

  - 读取：`config.playerLayer?.player.tracks(mediaType:)`
  - 切换：`config.playerLayer?.player.select(track:)`
  - → `KSMEPlayer.tracks/select` → `MEPlayerItem`

- 动态信息：

  - `config.playerLayer?.player.dynamicInfo` → `KSMEPlayer.dynamicInfo` → `MEPlayerItem.dynamicInfo`

- 内嵌字幕：
  - `Coordinator` 在 `readyToPlay` 后，从 `layer.player.subtitleDataSouce` 注入到 `subtitleModel`（有延迟处理，兼容内嵌字幕较晚可用的场景）。

---

## 7) 使用 `KSMEPlayer` 的选择策略

- 默认：`KSOptions.firstPlayerType = KSAVPlayer.self`，`secondPlayerType = KSMEPlayer.self`（出错自动兜底切换）。
- 特殊场景：
  - AirPlay 外接：强制走 `KSAVPlayer`。
  - `options.display != .plane`（如 AR 模式）：强制走 `KSMEPlayer`。
- 全局切换为 `KSMEPlayer`（示例，按需放在 App 启动处）：

```swift
KSOptions.firstPlayerType = KSMEPlayer.self
// 可保留 second 为 KSAVPlayer 作为兜底
KSOptions.secondPlayerType = KSAVPlayer.self
```

---

## 8) 快速定位索引（常用文件）

- SwiftUI 入口与控制：`Sources/KSPlayer/SwiftUI/KSVideoPlayerView.swift`
- 桥接与协调器：`Sources/KSPlayer/AVPlayer/KSVideoPlayer.swift`
- 播放承载层：`Sources/KSPlayer/AVPlayer/KSPlayerLayer.swift`
- 播放内核：`Sources/KSPlayer/MEPlayer/KSMEPlayer.swift`
- 全局选项：`Sources/KSPlayer/AVPlayer/KSOptions.swift`
- UIKit 控件版播放器（可参考对照）：`Sources/KSPlayer/Video/VideoPlayerView.swift`

---

如需在文档中加入更多代码引用或流程图，请告知偏好（例如补充 Mermaid 时序图）。
