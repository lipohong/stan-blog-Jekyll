---
title: "ReactJS From Zero Part 10: Capstone App, Testing, Accessibility, and Deployment Habits | ReactJS 從零開始之十：Capstone App、Testing、Accessibility 同 Deployment 習慣"
date: 2026-05-12 00:16:53 +0800
categories: [Frontend, React]
tags: [react, reactjs, react-19, beginner, hooks, frontend, javascript, tutorial, assisted_by_ai]
toc: true
mermaid: true
---


> Series navigation: **Part 10 of 10**  
> Previous: Part 9 explained how React reaches the server.  
> Next: This is the final post. Revisit Part 1 and rebuild the capstone without looking at the code.  
> Version note: npm reported `react@19.2.6` while the official docs currently document React 19.2.


## English Version

### Where This Part Fits

Part 9 explained how React reaches the server. In this part we focus on **capstone architecture, testing, accessibility, portals, error boundaries, deployment**. The goal is not to memorize API names. The goal is to build a mental model that helps you read code written by another developer and predict what React will do next.

The final step is not memorizing every hook. It is learning how to ship a small product: clear components, predictable state, useful tests, accessible UI, and a deployment checklist you can repeat.

React rewards small, explicit steps. A beginner often tries to understand the whole app at once, then gets lost. A real programmer usually moves in the opposite direction: find one component, find its inputs, find its local memory, find the event that changes that memory, then follow the render.

### Core Terms

| Term | Beginner meaning |
| --- | --- |
| `capstone` | A small project combines components, state, effects, context, async UI, and routing-like screens. |
| `testing` | Test behavior through the DOM, not implementation details. |
| `accessibility` | Use semantic HTML, labels, keyboard support, focus management, and clear error messages. |
| `production habits` | Use StrictMode, linting, type checks, error boundaries, and measured performance work. |

### The Render Story

A React screen is a tree. The root renders `App`, `App` renders smaller components, and those components render even smaller pieces. When state changes, React does not ask you to manually patch the DOM. It calls your component again, receives the next UI description, compares it with the previous description, and updates the browser efficiently.

The beginner mistake is thinking "render" means "delete the whole page and rebuild everything." That is not the useful mental model. A better model is: render means your component function is asked to describe what the UI should look like for the current inputs. React then decides the minimal DOM work.

For this post, keep these names in your head: capstone, testing, accessibility, production habits. Every code example below is just another way of combining those same ideas.

## 粵語中文版

### 今篇喺系列入面嘅位置

Part 9 explained how React reaches the server. 今篇集中講 **capstone architecture, testing, accessibility, portals, error boundaries, deployment**。目標唔係背 API 名，而係建立一個腦內地圖：當你睇到其他人寫嘅 React code，你可以估到下一步 React 會做咩。

最後一步唔係背晒每個 hook，而係學識點出一個細產品：清楚 component、可預測 state、有用 tests、易用 accessibility，同一份可以重複用嘅 deployment checklist。

React 最鍾意細步、清楚、可預測。新手成日一嚟就想理解成個 app，結果就迷路。實戰 programmer 通常倒轉做：先搵一個 component，睇佢收咩 input，睇佢自己記住咩 state，睇邊個 event 會改 state，最後跟住 render 條路行。

### 核心 Terms

| Term | 新手理解 |
| --- | --- |
| `capstone` | A small project combines components, state, effects, context, async UI, and routing-like screens. 呢個 term 我哋先保留英文，因為你之後睇官方 docs、Stack Overflow、GitHub issue 都會見到同一個字。 |
| `testing` | Test behavior through the DOM, not implementation details. 呢個 term 我哋先保留英文，因為你之後睇官方 docs、Stack Overflow、GitHub issue 都會見到同一個字。 |
| `accessibility` | Use semantic HTML, labels, keyboard support, focus management, and clear error messages. 呢個 term 我哋先保留英文，因為你之後睇官方 docs、Stack Overflow、GitHub issue 都會見到同一個字。 |
| `production habits` | Use StrictMode, linting, type checks, error boundaries, and measured performance work. 呢個 term 我哋先保留英文，因為你之後睇官方 docs、Stack Overflow、GitHub issue 都會見到同一個字。 |

### Render 呢件事點諗

React 畫面係一棵 tree。root render `App`，`App` render 細 component，細 component 再 render 更細嘅 UI。當 state 改變，React 唔係要你人手 patch DOM；佢會再 call 你個 component，攞到下一個 UI 描述，再同上一個描述比較，最後更新 browser 入面需要改嘅地方。

新手常見誤會係覺得 render 等於「成頁刪晒再砌過」。咁諗會令你驚。更實用嘅諗法係：render 即係 component function 根據而家嘅 inputs 描述畫面；至於 DOM 點樣最少量更新，交俾 React。


### Example 1: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it once in English and once in Cantonese. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
type AppScreen = 'inbox' | 'project' | 'settings';

function App() {
  const [screen, setScreen] = useState<AppScreen>('inbox');

  return (
    <ThemeProvider>
      <AppShell
        nav={<Sidebar screen={screen} onNavigate={setScreen} />}
        main={
          <ErrorBoundary fallback={<p role="alert">Screen crashed.</p>}>
            {screen === 'inbox' && <InboxScreen />}
            {screen === 'project' && <ProjectScreen />}
            {screen === 'settings' && <SettingsScreen />}
          </ErrorBoundary>
        }
      />
    </ThemeProvider>
  );
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

**粵語 walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。


### Example 2: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it once in English and once in Cantonese. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: unknown, info: unknown) {
    console.error('UI failed', error, info);
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

**粵語 walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。


### Example 3: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it once in English and once in Cantonese. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
import { createPortal } from 'react-dom';

function ConfirmDialog({
  open,
  title,
  onConfirm,
  onCancel
}: {
  open: boolean;
  title: string;
  onConfirm: () => void;
  onCancel: () => void;
}) {
  if (!open) return null;

  return createPortal(
    <div role="dialog" aria-modal="true" aria-labelledby="confirm-title">
      <h2 id="confirm-title">{title}</h2>
      <button onClick={onCancel}>Cancel</button>
      <button onClick={onConfirm}>Confirm</button>
    </div>,
    document.body
  );
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

**粵語 walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。


### Example 4: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it once in English and once in Cantonese. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { expect, test } from 'vitest';

test('creates a task from the form', async () => {
  const user = userEvent.setup();
  render(<TaskApp />);

  await user.type(screen.getByLabelText(/task title/i), 'Learn React');
  await user.click(screen.getByRole('button', { name: /create/i }));

  expect(screen.getByText('Learn React')).toBeInTheDocument();
});
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

**粵語 walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。


### Example 5: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it once in English and once in Cantonese. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
function AccessibleField({
  id,
  label,
  error
}: {
  id: string;
  label: string;
  error?: string;
}) {
  const errorId = `${id}-error`;
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        aria-invalid={Boolean(error)}
        aria-describedby={error ? errorId : undefined}
      />
      {error && <p id={errorId} role="alert">{error}</p>}
    </div>
  );
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

**粵語 walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。


## Practice Lab: Build It, Break It, Fix It

This lab connects this article to the rest of the series. Do not only paste the code. Type it, rename variables, remove one line at a time, and watch the browser complain. React becomes much less scary when you learn what each error message is trying to protect.

1. Create a fresh Vite React TypeScript project.
2. Copy the smallest example from this post first.
3. Add one feature from the previous post so the knowledge chain stays connected.
4. Add one tiny feature that prepares you for the next post: This is the final post. Revisit Part 1 and rebuild the capstone without looking at the code.
5. Open React DevTools and inspect the component tree.
6. Write down which values are props, which values are state, and which values are plain derived variables.

### Cantonese Practice Notes

今篇唔好只係 copy code。你要試吓打錯 prop 名、刪走 dependency、用錯 key、或者將 state 直接 mutate，睇吓 React 同 browser 會點樣提醒你。新手最容易驚 error，但其實 error 係老師傅喺你後面提你：「呢度遲早會爆，依家快啲修正。」

- 如果畫面冇更新，先問：我係咪改咗舊 object，而唔係建立新 object？
- 如果 effect 不停跑，先問：dependency 係咪每次 render 都變成新 reference？
- 如果 list 行為怪，先問：key 係咪穩定 ID，而唔係 array index？
- 如果 form 打唔到字，先問：input `value` 有冇配返 `onChange`？
- 如果 component 好難測，先問：我係咪將太多責任塞咗入同一個 function？

```mermaid
flowchart TD
    A[Read the UI] --> B[Find the component]
    B --> C[Identify props]
    C --> D[Identify state]
    D --> E[Trigger event]
    E --> F[React renders again]
    F --> G[Compare expected DOM]
    G --> H{Bug?}
    H -->|Yes| B
    H -->|No| I[Add the next small feature]
```

### Debugging Checklist for Part 10

- Read the browser console before changing code randomly.
- Check whether the bug happens before render, during render, or after render in an Effect.
- Use descriptive component names; `ProductCard` teaches your future self more than `Box`.
- Keep event handlers small. If an event needs ten steps, move the calculation into a helper function and test it separately.
- Prefer boring state names: `isOpen`, `selectedId`, `draftTitle`, `status`, `error`.
- When you feel tempted to add a library, first build the tiny version yourself so you know what problem the library solves.


## Common Beginner Mistakes

- Trying to learn React by memorizing syntax only. Syntax matters, but the mental model matters more.
- Mixing data calculation with DOM manipulation. In React, calculate data first, then describe UI.
- Keeping duplicate state. If a value can be calculated from existing props or state, calculate it during render.
- Making one giant component. When a JSX block becomes hard to scan, extract a named component.
- Ignoring official docs. For React 19.2, the official docs are practical and example-heavy.

## 新手常犯錯

- 只背 syntax。Syntax 要識，但 mental model 更重要。
- 將 data calculation 同 DOM manipulation 撈埋一齊。React 入面通常係先計 data，再描述 UI。
- 儲 duplicate state。如果一個 value 可以由現有 props 或 state 計出嚟，就唔好再開多個 state。
- 一個 component 寫到成座山咁大。JSX 一難 scan，就抽做有名 component。
- 唔睇官方 docs。React 19.2 docs 已經有好多實用例子，尤其係 Hooks、Effects、Server APIs 同 Compiler 部分。


## References

- React docs version page: <https://react.dev/versions>
- React 19.2 release notes: <https://react.dev/blog/2025/10/01/react-19-2>
- React Hooks reference: <https://react.dev/reference/react/hooks>
- React Activity reference: <https://react.dev/reference/react/Activity>
- React useEffectEvent reference: <https://react.dev/reference/react/useEffectEvent>
- React Compiler guide: <https://react.dev/learn/react-compiler>
