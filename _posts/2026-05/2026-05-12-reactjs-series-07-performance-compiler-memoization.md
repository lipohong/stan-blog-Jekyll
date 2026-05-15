---
title: "ReactJS From Zero Part 7: Performance, Memoization, and React Compiler | ReactJS 從零開始之七：Performance、Memoization 同 React Compiler"
date: 2026-05-12 00:16:53 +0800
categories: [Frontend, React]
tags: [react, reactjs, react-19, beginner, hooks, frontend, javascript, tutorial, assisted_by_ai]
toc: true
mermaid: true
---


> Series navigation: **Part 7 of 10**  
> Previous: Part 6 built reusable composition and shared context.  
> Next: Part 8 introduces async UI patterns: Suspense, lazy loading, transitions, optimistic updates, and Actions.  
> Version note: npm reported `react@19.2.6` while the official docs currently document React 19.2.

## Introduction: Where This Part Fits

Part 6 built reusable composition and shared context. In this part we focus on **render performance, memo, useMemo, useCallback, Profiler, React Compiler**. The goal is not to memorize API names. The goal is to build a mental model that helps you read code written by another developer and predict what React will do next.

Performance work is not guessing which waiter is slow. You first watch the whole restaurant during rush hour, then optimize the station that actually blocks orders.

React rewards small, explicit steps. A beginner often tries to understand the whole app at once, then gets lost. A real programmer usually moves in the opposite direction: find one component, find its inputs, find its local memory, find the event that changes that memory, then follow the render.

## Core Terms

| Term | Beginner meaning |
| --- | --- |
| `measure first` | Use React DevTools Profiler before adding memo everywhere. |
| `memo` | Skip child re-rendering when props are unchanged. |
| `useMemo/useCallback` | Cache expensive values or stable function props when there is a concrete need. |
| `React Compiler` | A build-time optimizer that can reduce manual memoization in supported projects. |

## The Render Story

A React screen is a tree. The root renders `App`, `App` renders smaller components, and those components render even smaller pieces. When state changes, React does not ask you to manually patch the DOM. It calls your component again, receives the next UI description, compares it with the previous description, and updates the browser efficiently.

The beginner mistake is thinking "render" means "delete the whole page and rebuild everything." That is not the useful mental model. A better model is: render means your component function is asked to describe what the UI should look like for the current inputs. React then decides the minimal DOM work.

For this post, keep these names in your head: measure first, memo, useMemo/useCallback, React Compiler. Every code example below is just another way of combining those same ideas.

## Code Implementation: Examples and Walkthroughs

### Example 1: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it back in your own words. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
import { memo, useMemo, useState } from 'react';

const ProductCard = memo(function ProductCard({
  product,
  onSelect
}: {
  product: Product;
  onSelect: (id: string) => void;
}) {
  return (
    <button className="product-card" onClick={() => onSelect(product.id)}>
      <strong>{product.name}</strong>
      <span>${product.price}</span>
    </button>
  );
});
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

### Example 2: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it back in your own words. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
function ProductGrid({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  const visibleProducts = useMemo(() => {
    return products
      .filter(product => product.name.toLowerCase().includes(query.toLowerCase()))
      .toSorted((a, b) =>
        sortBy === 'name'
          ? a.name.localeCompare(b.name)
          : a.price - b.price
      );
  }, [products, query, sortBy]);

  return (
    <>
      <SearchBox value={query} onChange={setQuery} />
      <SortSelect value={sortBy} onChange={setSortBy} />
      {visibleProducts.map(product => (
        <ProductCard key={product.id} product={product} onSelect={openProduct} />
      ))}
    </>
  );
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

### Example 3: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it back in your own words. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
// React Compiler can automatically memoize many components.
// Manual memoization still matters when you are not using the compiler,
// when profiling proves a bottleneck, or when integrating older code.

function InvoiceTotal({ lines }: { lines: InvoiceLine[] }) {
  const total = lines.reduce((sum, line) => sum + line.quantity * line.price, 0);
  return <output>${total.toFixed(2)}</output>;
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

### Example 4: Read the Code Slowly

Before reading the snippet, predict three things: what data enters the component, what event can happen, and what visible output should change. After reading it, explain it back in your own words. This habit trains you to understand React code without depending on tutorials forever.

{% raw %}
```tsx
// Compiler directive examples for gradual adoption.
function StableReportTable({ rows }: { rows: ReportRow[] }) {
  "use memo";
  return <Table rows={rows} />;
}

function ThirdPartyWrapper() {
  "use no memo"; // Temporary opt-out while investigating library behavior.
  return <LegacyChart />;
}
```
{% endraw %}

**English walkthrough:** Start from the component boundary. Check the props or local variables first. Then scan for Hooks because Hooks tell you what the component remembers or synchronizes. Finally, read the returned JSX as a plain UI description. If the example has an event handler, imagine the click or typing action, then follow the state update into the next render.

## Practice Lab: Build It, Break It, Fix It

This lab connects this article to the rest of the series. Do not only paste the code. Type it, rename variables, remove one line at a time, and watch the browser complain. React becomes much less scary when you learn what each error message is trying to protect.

1. Create a fresh Vite React TypeScript project.
2. Copy the smallest example from this post first.
3. Add one feature from the previous post so the knowledge chain stays connected.
4. Add one tiny feature that prepares you for the next post: Part 8 introduces async UI patterns: Suspense, lazy loading, transitions, optimistic updates, and Actions.
5. Open React DevTools and inspect the component tree.
6. Write down which values are props, which values are state, and which values are plain derived variables.

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

### Debugging Checklist for Part 7

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

## Looking Ahead

This part is one step in the series. Keep the code small, verify it in the browser, and continue with the next topic: Part 8 introduces async UI patterns: Suspense, lazy loading, transitions, optimistic updates, and Actions. The important habit is steady practice: read the component boundary, identify props and state, trigger one event, and confirm the next render.


---
---

## 簡介：今篇喺系列入面嘅位置

上一篇我哋用 composition 同 context 整可重用結構。今篇集中講 **render performance, memo, useMemo, useCallback, Profiler, React Compiler**。目標唔係背 API 名，而係建立一個腦內地圖：當你睇到其他人寫嘅 React code，你可以估到下一步 React 會做咩。

Performance 唔係估邊個侍應慢；你要先喺繁忙時間睇全場，搵到真係卡住出餐嗰個位，先落手優化。

React 最鍾意細步、清楚、可預測。新手成日一嚟就想理解成個 app，結果就迷路。實戰 programmer 通常倒轉做：先搵一個 component，睇佢收咩 input，睇佢自己記住咩 state，睇邊個 event 會改 state，最後跟住 render 條路行。

## 核心 Terms

| Term | 新手理解 |
| --- | --- |
| `measure first` | 優化前先用 React DevTools Profiler 量度，唔好憑感覺亂加 memo。 |
| `memo` | `memo` 可以喺 props 無變時跳過 child re-render。 |
| `useMemo/useCallback` | `useMemo` cache value，`useCallback` cache function；要有實際需要先用。 |
| `React Compiler` | React Compiler 係 build-time optimizer，可以自動做一部分 memoization。 |

## Render 呢件事點諗

React 畫面係一棵 tree。root render `App`，`App` render 細 component，細 component 再 render 更細嘅 UI。當 state 改變，React 唔係要你人手 patch DOM；佢會再 call 你個 component，攞到下一個 UI 描述，再同上一個描述比較，最後更新 browser 入面需要改嘅地方。

新手常見誤會係覺得 render 等於「成頁刪晒再砌過」。咁諗會令你驚。更實用嘅諗法係：render 即係 component function 根據而家嘅 inputs 描述畫面；至於 DOM 點樣最少量更新，交俾 React。

## Code 實作：Examples 同 Walkthrough

### 例子 1：慢慢讀段 Code

讀 code 之前，先估三樣嘢：呢個 component 收咩 data、user 可以做咩 event、畫面邊度應該會變。讀完之後，用自己嘅說話講返一次。呢個習慣會幫你慢慢脫離 tutorial，真正睇得明 React code。

{% raw %}
```tsx
import { memo, useMemo, useState } from 'react';

const ProductCard = memo(function ProductCard({
  product,
  onSelect
}: {
  product: Product;
  onSelect: (id: string) => void;
}) {
  return (
    <button className="product-card" onClick={() => onSelect(product.id)}>
      <strong>{product.name}</strong>
      <span>${product.price}</span>
    </button>
  );
});
```
{% endraw %}

**Walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。

### 例子 2：慢慢讀段 Code

讀 code 之前，先估三樣嘢：呢個 component 收咩 data、user 可以做咩 event、畫面邊度應該會變。讀完之後，用自己嘅說話講返一次。呢個習慣會幫你慢慢脫離 tutorial，真正睇得明 React code。

{% raw %}
```tsx
function ProductGrid({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  const visibleProducts = useMemo(() => {
    return products
      .filter(product => product.name.toLowerCase().includes(query.toLowerCase()))
      .toSorted((a, b) =>
        sortBy === 'name'
          ? a.name.localeCompare(b.name)
          : a.price - b.price
      );
  }, [products, query, sortBy]);

  return (
    <>
      <SearchBox value={query} onChange={setQuery} />
      <SortSelect value={sortBy} onChange={setSortBy} />
      {visibleProducts.map(product => (
        <ProductCard key={product.id} product={product} onSelect={openProduct} />
      ))}
    </>
  );
}
```
{% endraw %}

**Walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。

### 例子 3：慢慢讀段 Code

讀 code 之前，先估三樣嘢：呢個 component 收咩 data、user 可以做咩 event、畫面邊度應該會變。讀完之後，用自己嘅說話講返一次。呢個習慣會幫你慢慢脫離 tutorial，真正睇得明 React code。

{% raw %}
```tsx
// React Compiler can automatically memoize many components.
// Manual memoization still matters when you are not using the compiler,
// when profiling proves a bottleneck, or when integrating older code.

function InvoiceTotal({ lines }: { lines: InvoiceLine[] }) {
  const total = lines.reduce((sum, line) => sum + line.quantity * line.price, 0);
  return <output>${total.toFixed(2)}</output>;
}
```
{% endraw %}

**Walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。

### 例子 4：慢慢讀段 Code

讀 code 之前，先估三樣嘢：呢個 component 收咩 data、user 可以做咩 event、畫面邊度應該會變。讀完之後，用自己嘅說話講返一次。呢個習慣會幫你慢慢脫離 tutorial，真正睇得明 React code。

{% raw %}
```tsx
// Compiler directive examples for gradual adoption.
function StableReportTable({ rows }: { rows: ReportRow[] }) {
  "use memo";
  return <Table rows={rows} />;
}

function ThirdPartyWrapper() {
  "use no memo"; // Temporary opt-out while investigating library behavior.
  return <LegacyChart />;
}
```
{% endraw %}

**Walkthrough：** 由 component 邊界開始睇。先睇 props 或 local variables，再掃 Hooks，因為 Hooks 會話你知 component 記住咩或者同步咩。最後將 JSX 當成普通 UI 描述咁讀。如果有 event handler，就想像自己 click 或打字，跟住 state update 行去下一次 render。

## 練習 Lab：整出嚟、整壞佢、再修好佢

呢個 lab 會將今篇同成個系列連埋一齊。唔好只係 paste code；你要親手打、改 variable 名、逐行刪走試吓，睇 browser 同 React 會點樣投訴。當你明白 error message 想保護你咩，React 就冇咁可怕。

1. 開一個新嘅 Vite React TypeScript project。
2. 先 copy 今篇最細嗰個 example，確認畫面行到。
3. 加返上一篇學過嘅一個 feature，等知識鏈唔會斷。
4. 加入一個好細嘅 feature，預備下一篇會講嘅內容：第八篇會進入 async UI patterns，包括 Suspense、lazy loading、transitions、optimistic updates 同 Actions。
5. 打開 React DevTools，睇吓 component tree。
6. 寫低邊啲 value 係 props、邊啲係 state、邊啲只係 render 時即時計出嚟嘅 derived variables。


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

### Part 7 Debug Checklist

- 亂改 code 之前，先讀 browser console。
- 分清楚 bug 係 render 前、render 中，定係 Effect 之後先發生。
- Component 名要有意思；`ProductCard` 會比 `Box` 更幫到未來嘅你。
- Event handler 盡量細。如果一個 event 要做十步，將計算搬去 helper function，再分開測。
- State 名寧願悶但清楚：`isOpen`、`selectedId`、`draftTitle`、`status`、`error`。
- 想加 library 之前，先自己整一個 tiny version，明白 library 到底幫你解決咩問題。

## 新手常犯錯

- 只背 syntax。Syntax 要識，但 mental model 更重要。
- 將 data calculation 同 DOM manipulation 撈埋一齊。React 入面通常係先計 data，再描述 UI。
- 儲 duplicate state。如果一個 value 可以由現有 props 或 state 計出嚟，就唔好再開多個 state。
- 一個 component 寫到成座山咁大。JSX 一難 scan，就抽做有名 component。
- 唔睇官方 docs。React 19.2 docs 已經有好多實用例子，尤其係 Hooks、Effects、Server APIs 同 Compiler 部分。

## 總結與展望

今篇係系列入面其中一步。保持 code 細、喺 browser 入面驗證，然後繼續下一個 topic：第八篇會進入 async UI patterns，包括 Suspense、lazy loading、transitions、optimistic updates 同 Actions。 最重要嘅習慣係穩定練習：讀 component boundary、分清 props 同 state、觸發一個 event，再確認下一次 render 係咪符合預期。


## References

- React docs version page: <https://react.dev/versions>
- React 19.2 release notes: <https://react.dev/blog/2025/10/01/react-19-2>
- React Hooks reference: <https://react.dev/reference/react/hooks>
- React Activity reference: <https://react.dev/reference/react/Activity>
- React useEffectEvent reference: <https://react.dev/reference/react/useEffectEvent>
- React Compiler guide: <https://react.dev/learn/react-compiler>
