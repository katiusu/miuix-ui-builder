# 组件契约：组件内部会动你的状态

## 核心

一个 Composable 的回调不是你的私有接口。组件在**同一条路径**上往往还会做自己的事：
清掉它持有的值、改 `enabled`、请求/清理焦点、安装返回手势拦截、按你的参数重建内部 state。

症状通常长得像**你自己的状态 bug**（"我明明没清，值怎么就没了"），于是你在自己的代码里找半天。
先读组件实现，再改自己的代码。

## 真实案例：Miuix `SearchBar` / `InputField` 回车清空搜索词

**用户看到的现象**：日志页搜索框里输入关键词、按回车，关键词"自动消失"，列表恢复成全部。

**归因过程**：本页的 `query` 是 `rememberSaveable` 的 hoisted state，回车回调原本只是
`searchExpanded = false`（收起搜索），没有任何清空动作 —— 所以第一嫌疑是组件。

**根因**（Miuix 0.9.3 `basic/SearchBar.kt`，`InputField` 内部）：

```kotlin
LaunchedEffect(expanded) {
    if (expanded) {
        focusRequester.requestFocus()
    } else if (focused) {          // ← "收起搜索"且当时仍有焦点
        delay(100)
        if (query.isNotEmpty()) {
            textAlpha.animateTo(0f)
            currentOnQueryChange("")   // ← 组件主动把 query 清空（调的是你的回调）
            textAlpha.snapTo(1f)
        }
        focusManager.clearFocus()
    }
}
```

也就是说：**在这个组件里 `expanded = false` 的语义是"退出搜索并重置"**，包括把 query 交回空串。
把"回车"映射成收起，就等于让组件清掉你的筛选条件。

**同一个根因的第二个入口**：点建议项时也 `searchExpanded = false`（同样收起）→ 同样被清空。
修一处漏一处就还会复现，**修根因要顺着"谁进入了那个状态"全部排查**。

**修法**（选一个，别用时序 hack）：

| 方案 | 适用 | 代价 |
|---|---|---|
| **不进入那个状态**：回车只收键盘（`focusManager.clearFocus()`）、保持"正在搜索" | 组件的 `expanded` 本身就是"搜索进行中"，语义一致 | 退出搜索改由返回键完成（组件自带 `NavigationBackHandler`） |
| 自己再存一份"已提交的条件"，UI 显示用它们 | 需要"收起但仍显示筛选"的形态 | 与组件内部态可能不一致，要自己维护 |
| 换组件 / 自组合 | 契约实在不合 | 自研件的语义、禁用态、无障碍由你负责 |

**禁止**：在回调里先 `clearFocus()` 再立刻 `expanded = false`，指望绕过上面那个分支。焦点状态的更新与
`LaunchedEffect` 的调度之间没有顺序保证 —— 这是 race，不是修法。

## 审计一个陌生组件（10 分钟）

```bash
S=/tmp/lib-src/**/basic/SearchBar.kt        # 换成目标组件
grep -n "LaunchedEffect\|SideEffect\|DisposableEffect\|rememberUpdatedState" "$S"
grep -n "currentOn[A-Z]" "$S"               # 它在哪里回调用你的回调
grep -n "enabled\s*=\|focusRequester\|clearFocus" "$S"
grep -n "BackHandler\|NavigationBackHandler\|requestFocus" "$S"
```

对着结果问五个问题：

1. **它会用"空/默认值"调我的回调吗？**（`onQueryChange("")`、`onExpandedChange(false)`…）
2. **我的哪个状态/回调会触发它进入那个分支？**（把所有入口列全，别只修你发现的那一个）
3. **它有没有 workaround 分支会让控件在某状态下不可用？**
   （例：Miuix 在 API 26–27 上收起时 `enabled=false`，改由 `pointerInput` 处理点击展开）
4. **它自己装的返回/手势拦截会不会和外壳的抢？**
   （Compose 里是"后注册的 enabled 回调优先"，组件在列表项里 compose，通常晚于外壳 → 它优先；
   但列表项滚出屏幕被 dispose 后，外壳的又会接管。**这条需要在真机上点一次确认，别只靠推断。**）
5. **它有没有按我的参数 `remember(key)` 重建内部 state？**（重建会重置动画/焦点等瞬时状态）

## 判断"该改谁"

- 组件的行为**是它的设计语义**（如"收起=重置"）→ 改**你的状态机**去适配，别去对抗它；
- 组件的行为是**实现细节**（如某个 workaround 分支）→ 你有理由绕过，但要写清依据并在真机验证；
- 两者都不是，是你没读文档 → 回到 `miuix` 技能读该组件的 doc/demo。
