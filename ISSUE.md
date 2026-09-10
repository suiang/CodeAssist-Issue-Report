# 发现页先显示模板页面，然后跳转到发布页

## 问题描述
在首页下部导航栏点击"发现"（Store）按钮时，页面会先显示捆绑模板页面（ProjectsStoreScreen），然后立即跳转到发布/探索页面（ExploreFeed）。这导致用户体验不连贯，页面出现突兀的转换。

## 实际行为
1. 点击导航栏的发现按钮
2. 先短暂显示 ProjectsStoreScreen（模板页面）
3. 随后自动跳转到 ExploreFeed（发布页面）

## 期望行为
- 应该直接显示发布/探索页面（ExploreFeed）
- 或者在加载期间显示加载状态，而不是显示不同的内容页面

## 根本原因
在 `app/ide-ui/src/commonMain/kotlin/dev/ide/ui/AppNavGraph.kt` 的 `StoreRoute()` 函数中：

初始状态：`feed` 为 `null`（produceState 的初始值）
此时条件 `if (current == null)` 为真，显示 `ProjectsStoreScreen`
当服务器返回数据时，`feed` 变为非 null
UI 重新组合，条件变为假，显示 `ExploreFeed`

```kotlin
@Composable
private fun StoreRoute(app: CodeAssistAppState, fileActions: FileActions) {
    // ...
    val feed by produceState<dev.ide.ui.backend.UiStoreFeed?>(null, app.backend, app.epoch) {
        value = runCatching { app.backend.store.feed(seedItemId = null) }.getOrNull()
    }
    val current = feed
    if (current == null) {
        ProjectsStoreScreen(...)  // ← 先显示这个
        return
    }
    // feed 加载完成后，显示 ExploreFeed （这是"发布页"）
    ExploreFeed(...)  // ← 然后显示这个
}
```

## 建议方案

### 方案1：使用 Loading 状态
在 feed 加载中时显示加载指示器，而不是切换到不同页面

### 方案2：延迟显示
等待 feed 加载完成后再显示任何内容

### 方案3：保持页面稳定
使用 `ExploreFeed` 显示空状态（它已有 `EMPTY` 模式），避免页面切换

## 相关代码位置
- `app/ide-ui/src/commonMain/kotlin/dev/ide/ui/AppNavGraph.kt` (第 517-588 行)
- `app/ide-ui-screens/src/commonMain/kotlin/dev/ide/ui/screens/HomeScreen.kt`
- `app/ide-ui-screens/src/commonMain/kotlin/dev/ide/ui/screens/ExploreFeedScreen.kt`
- `app/ide-ui-screens/src/commonMain/kotlin/dev/ide/ui/screens/ProjectsStoreScreen.kt`

## 环境信息
- **项目**: CodeAssist
- **组件**: Home Screen / Store Tab
- **严重级别**: Low (UI UX issue)

## 复现步骤
1. 打开 CodeAssist
2. 进入首页（Projects 页面）
3. 点击下部导航栏的"发现"按钮
4. 观察：会先显示模板页面，然后跳转到发布页面
