---
title: "Navigation API：面向现代 SPA 的客户端路由原语"
date: 2026-07-19T20:00:10Z
draft: false
tags: ["Navigation API", "单页应用", "浏览器", "路由"]
categories: ["技术实践"]
author: "X 实验室"
description: "学习如何使用 Navigation API 构建健壮的 SPA 路由器：集中式导航控制、内置滚动恢复、无缝视图过渡——现已在所有主流浏览器中标记为 Baseline Newly available。"
summary: "本文提供 Navigation API 的生产级实践指南，涵盖 intercept 语义、两阶段提交、手动滚动控制、状态管理，以及与 View Transitions 的集成。包含特性探测、优雅降级策略，以及跨浏览器 SPA 路由的验证清单。"
toc: true
---

# Navigation API：面向现代 SPA 的客户端路由原语

## 背景与动机

单页应用（SPA）十余年来一直依赖 History API 来模拟多页导航而无需整页刷新。然而 History API 并非为现代 SPA 的需求而设计。开发者不得不拼凑一套脆弱的方案：

- 全局监听 `<a>` 点击，调用 `preventDefault()`，手动执行 `history.pushState()`，再更新 DOM。
- 单独监听 `popstate` 以处理浏览器前进/后退。
- 漏掉某些导航类型（例如程序式 `location.href` 更改、部分表单提交）。
- 无法读取完整历史栈或编辑非当前条目。
- `popstate` 行为不一致——在 `pushState`/`replaceState` 时不会触发。

这导致脆弱的路由器充满边界情况：用户可能跳转到错误视图，或丢失滚动位置。

**Navigation API** 解决了这些架构缺口。它提供一个集中的 `navigate` 事件，捕获**所有**导航触发源——链接点击、前进/后退、表单提交、程序式调用——并附带强大原语：`intercept()`、`commit`/`finished` 双阶段提交、手动滚动控制、状态管理。截至 2026 年初，随着 Safari 与 Firefox 加入 Chrome 阵营，Navigation API 已在所有主流浏览器中标记为 **Baseline Newly available**。

## 前置条件

采用 Navigation API 前，需验证支持度并准备降级方案。

### 浏览器支持矩阵

- **Chrome / Edge**: 102+
- **Samsung Internet**: 19.0+
- **Safari**: 26.2+（2025 年起）
- **Firefox**: 147+（2026 年起）
- **iOS Safari**: 26.2+

旧版浏览器完全不支持该 API。务必进行特性探测并提供传统路径。

### 特性探测

```js
const hasNavigationAPI = 'navigation' in window && typeof window.navigation === 'object';
```

若不支持，回退到基于 History API 的路由或最简 SPA shim。

```js
if (!hasNavigationAPI) {
  // 传统 History API 回退
  window.addEventListener('popstate', handlePopState);
  document.querySelectorAll('a[href]').forEach(link => {
    link.addEventListener('click', (e) => {
      if (isSameOrigin(link.href)) {
        e.preventDefault();
        const url = new URL(link.href);
        window.history.pushState({ path: url.pathname }, '', url.pathname);
        renderApp(url.pathname);
      }
    });
  });
}
```

### 构建工具配置

无需特殊打包器设置。若使用 TypeScript，确保 `tsconfig.json` 的 `lib` 包含 `"DOM"`，最好包含带有 Navigation API 类型的较新版本。

## 步骤一：准备工作

第一步是用单个监听器集中导航逻辑。

### 基础设置

在主入口文件（如 `src/router.ts`）中：

```js
if (hasNavigationAPI) {
  window.navigation.addEventListener('navigate', (event) => {
    // 在这里处理所有导航
  });
}
```

### 拦截资格检查

并非所有导航都能或都应该被拦截。检查 `event.canIntercept`：

- 跨源导航：`canIntercept` 为 `false`，浏览器以整页刷新处理。
- 同源、同文档：可拦截。

同时跳过 `event.hashChange`（片段滚动）和 `event.downloadRequest`（文件下载）。

```js
navigation.addEventListener('navigate', (event) => {
  if (!event.canIntercept) return;

  if (event.hashChange || event.downloadRequest !== null) {
    return; // 让浏览器原生处理这些情况
  }

  const url = new URL(event.destination.url);
  // 继续拦截
});
```

**关键洞察**：将所有合格导航集中到一处处理，消除了多事件监听器与边界情况胶合代码的需要。

## 步骤二：核心实现

现在实现路由器的核心：带异步内容加载的 intercept 处理器、正确的滚动管理、状态持久化。

### 带异步处理器的 Intercept

`intercept()` 方法接收一个带有 `handler` 回调的对象。该回调可以是异步的。Navigation API 会等待返回的 Promise 完成后才触发 `navigatesuccess`。

```js
navigation.addEventListener('navigate', (event) => {
  if (!event.canIntercept) return;
  if (event.hashChange || event.downloadRequest !== null) return;

  const url = new URL(event.destination.url);

  event.intercept({
    async handler() {
      // 立即显示加载状态（可选）
      renderPlaceholder();

      // 获取新内容（从网络或缓存）
      const content = await fetchPageContent(url.pathname);

      // 更新 DOM
      document.getElementById('app').innerHTML = content;
    }
  });
});
```

**两阶段提交**（高级）规范定义了 `precommitHandler`，用于在 URL 更新前进行重定向。若应用需要在登录流程或 A/B 测试中重写 URL，可使用它：

```js
event.intercept({
  precommitHandler: (controller) => {
    if (needsAuth) {
      return controller.redirect('/signin');
    }
  },
  async handler() {
    // 正常渲染
  }
});
```

### 手动滚动控制

默认情况下，浏览器在 `intercept()` 被调用时即尝试滚动恢复。对于异步渲染内容的 SPA，这往往导致在页面高度尚未确定时就滚动，结果落在错误位置或停留在顶部。

设置 `scroll: 'manual'`，在内容就绪后调用 `event.scroll()`：

```js
navigation.addEventListener('navigate', (event) => {
  if (!event.canIntercept) return;

  event.intercept({
    scroll: 'manual',
    async handler() {
      const data = await fetchListData();
      renderList(data);

      // 现在文档具有正确高度；恢复滚动位置
      event.scroll();
    }
  });
});
```

此模式对前进/后退导航至关重要——用户期望回到之前的滚动偏移量。

### 状态管理

Navigation API 允许在每个历史条目上存储任意状态。这优于 History API（状态只能在 `pushState` 时设置）。

导航时设置状态：

```js
navigation.navigate('/dashboard', { state: { visitCount: 5, filters: { status: 'open' } } });
```

检索状态：

```js
const entry = navigation.currentEntry;
const state = entry.getState(); // { visitCount: 5, filters: { status: 'open' } }
```

对于非导航相关的状态变更（例如展开 `<details>` 元素），使用 `updateCurrentEntry()`：

```js
details.addEventListener('toggle', () => {
  const expanded = details.open;
  navigation.updateCurrentEntry({
    state: { ...navigation.currentEntry.getState(), detailsExpanded: expanded }
  });
});
```

这确保浏览器前进/后退能恢复 UI 状态而无需重新加载。

### 与 View Transitions 集成

Navigation API 与 [View Transitions API](https://developer.mozilla.org/docs/Web/API/View_Transitions_API) 设计为协同工作。将 DOM 更新包裹在 `document.startViewTransition()` 中：

```js
navigation.addEventListener('navigate', (event) => {
  if (!event.canIntercept) return;

  const url = new URL(event.destination.url);

  event.intercept({
    async handler() {
      const content = await fetchPageContent(url.pathname);

      document.startViewTransition(() => {
        document.getElementById('app').innerHTML = content;
      });
    }
  });
});
```

浏览器会拍摄旧 UI 快照、执行你的回调、再动画过渡到新状态——无需自定义动画即可获得类原生应用的过渡效果。

### 表单提交处理

`navigate` 事件为同源 POST 提交暴露 `event.formData`。拦截以避免页面刷新：

```js
navigation.addEventListener('navigate', (event) => {
  if (event.formData && event.canIntercept) {
    event.intercept({
      async handler() {
        const data = event.formData;
        const username = data.get('username');

        await api.login(username, data.get('password'));
        renderSuccess(`欢迎，${username}！`);
      }
    });
  }
});
```

标准 HTML 表单即可正常工作——无需 `onsubmit` 处理器。

### 导航生命周期事件

跟踪导航完成与错误：

- `navigation.onnavigatesuccess`：`handler()` 解决后触发。
- `navigation.onnavigateerror`：`handler()` 拒绝时触发。
- `navigation.transition.finished`：当前导航完成的 Promise。

```js
navigation.addEventListener('navigatesuccess', () => {
  console.log('导航成功提交');
});

navigation.addEventListener('navigateerror', (event) => {
  console.error('导航失败:', event.error);
  // 显示回退 UI 或重试
});
```

### 历史遍历

使用类型化方法进行精确控制：

- `navigation.back()`、`navigation.forward()`：标准遍历。
- `navigation.traverseTo(key)`：通过 `NavigationHistoryEntry.key` 跳转到特定条目。
- `navigation.reload()`：重新加载当前条目。

```js
// 记住用户首次着陆的页面
const homeKey = navigation.currentEntry.key;

// 始终返回会话根节点
document.getElementById('home-btn').onclick = () => {
  navigation.traverseTo(homeKey);
};
```

## 步骤三：验证与调优

### 特性探测检查清单

在 CI 或自动化测试中断言：

```js
test('has Navigation API support', () => {
  expect('navigation' in window).toBe(true);
  expect(typeof window.navigation.addEventListener).toBe('function');
});
```

### 手动 QA 清单

1. 点击同源链接 → URL 更新，无刷新，UI 渲染。
2. 按浏览器后退 → URL 回退，滚动位置恢复（若正确使用 `scroll: 'manual'`）。
3. 提交 POST 表单 → 无刷新，渲染服务器响应。
4. 程序式调用 `navigation.navigate()` → 中央处理器被调用。
5. 跨源链接点击 → 按预期整页刷新。

### DevTools 调试

Chrome DevTools 在 **Application > Navigation** 面板（实验性）中显示导航拦截。在 `handler()` 回调内设置断点，单步调试异步 fetch 与渲染。

### 优雅降级策略

对于不支持的浏览器：

- 提供完整 SSR 回退（每个路由都是服务端渲染页面；链接是常规 `<a>` 标签）。
- 最小化 polyfill：使用 History API 进行 push/pop 路由。
- 注意：Navigation API 没有真正的 polyfill。设计应用时应以 History API 为底线。

### 已知限制

1. **首次加载**：初始页面加载时**不会**触发 `navigate`。SSR 应用自然处理此问题；CSR 应用必须在启动时调用 `init()`。
2. **单作用域**：API 仅在一个浏览上下文中运行。`<iframe>` 内的导航不可见。
3. **无法重排历史**：你不能移除或重新排列条目。不应持久化在历史中的临时模态框需要谨慎的状态设计。

### 性能考量

- **拦截期间 fetch**：在提交 URL 前加载内容，若 fetch 失败可能增加感知延迟。考虑乐观 UI 更新并在错误时回滚。
- **避免嵌套 intercept**：除非你谨慎处理 `finished` Promise 以避免循环，否则不要在 `handler()` 内调用 `navigation.navigate()`。
- **内存**：历史条目会累积。使用 `entry.dispose` 事件清理与已处置条目绑定的缓存：

```js
entry.addEventListener('dispose', () => {
  cache.delete(entry.key);
});
```

## 最佳实践小结

- **单一事实来源**：在一个 `navigate` 监听器中处理所有导航。不要与手动 `pushState` 调用混用。
- **使用 `scroll: 'manual'`**：始终将滚动恢复推迟到异步渲染完成后。
- **利用状态**：在历史条目上存储视图状态（过滤器、展开区段），以实现正确的前进/后退行为。
- **检查 `canIntercept`**：尊重跨源与下载导航——不要强制拦截。
- **集成 View Transitions**：将 DOM 更新包裹在 `document.startViewTransition()` 中，实现流畅的内置动画。
- **为传统浏览器规划**：为不支持的浏览器提供 History API 回退或 SSR 基线。

## 参考资料

- MDN Web Docs, [Navigation API](https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API)
- web.dev, [Navigation API - a better way to navigate, is now Baseline Newly Available](https://web.dev/blog/baseline-navigation-api)
- WICG, [Navigation API Explainer](https://github.com/WICG/navigation-api/blob/main/README.md)
- Can I use, [Navigation API support table](https://caniuse.com/mdn-api_navigation)
- MDN, [NavigateEvent.intercept()](https://developer.mozilla.org/en-US/docs/Web/API/NavigateEvent/intercept)
- MDN, [View Transitions API](https://developer.mozilla.org/docs/Web/API/View_Transitions_API)