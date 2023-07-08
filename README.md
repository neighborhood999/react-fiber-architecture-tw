# React Fiber Architecture

> 原文：[acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)
## 介紹

React Fiber 是對 React 核心演算法的重新實作，目前正在進行中。這是 React 團隊經過兩年多研究的結果。

Fiber 的主要目標是提高 animation、layout、gesture 的適用性，讓 Virtual DOM 可以 **incremental rendering**，能夠將 render 工作拆分為多個 chunk，並將其分散到多個 frame 中；另一個關鍵功能是，隨著更新的出現而**暫停、中斷和恢復**工作的能力；為不同類型的更新分配優先權，以及新的 concurrency primitives。
## 關於這份文件

Fiber 介紹了一些難以直接通過閱讀程式碼理解的新穎概念。這份文件的開始是我在 React 專案中跟隨 Fiber 的實作所收集的收集的筆記。隨著它慢慢累積，我明白到這份文件可能對其他人也是有幫助的。

我將嘗試使用直白的方式，並通過避免透過明確定義的關鍵術語來避免一些電腦術語。我還會盡可能大量引用一些外部的資源。

請注意我不是來自 React team，也不從任何權威的角度發言。**這不是官方的文件**。我會請求 React team 的成員來幫忙檢閱其準確性。

**Fiber 仍然在開發當中的專案，在完成之前可能會進行重大的重構**。我也嘗試在此記錄它的設計。歡迎任何改善和建議。

我的目標是在讀完本份文件後，你可以對跟著 Fiber [實作並對它有足夠的了解](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，甚至最後能為 React 貢獻。
## 先前準備

我強烈建議你在開始閱讀之前，熟悉以下這些資源：

- [React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - 「Component」是很常被用到的術語。掌握這些術語非常重要。
- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - 一個對 React 的 reconciliation 演算法的 high-level 描述。
- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - 沒有實作負擔的 React 概念模型的描述。一開始有些東西可能讀起來沒意義。沒關係，隨著時間的推移它會變得更有意義。
- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - 請特別注意 scheduling 的部分。它很好地解釋了為什麼使用 React Fiber。
### 什麼是 Reconciliation？

- **_reconciliation_** - React 用來比較 tree 之間的 diff 演算法，決定哪些部分需要被改變。
- **_update_** - 用於 render React 應用程式中的資料更改。通常是 `setState` 的結果。最終結果會是一次 re-render。

這讓開發者可以進行聲明性（declaratively）的推導，而不必擔心如何有效的將應用程式從任何特定的狀態轉換到另一個狀態（A 到 B、B 到 C、C 到 A 等等）。

實際上，在每次改變時重新 re-render 整個應用程式只適合用於在最瑣碎的部分；在現實的世界應用中，就效能而言，它的成本是非常昂貴的。React 有最佳化的功能，可以在保持好的效能的同時建立整個應用程式 re-render 的外觀。這些最佳化大部分是一個被稱為 **reconciliation** 過程的一部份。

Reconciliation 是一個通常被大家稱為「Virtual DOM」背後的演算法。從 high-level 來描述是：當你 render 一個 React 應用程式，將會產生一個節點樹來描述你的應用程式，並且被儲存到記憶體中。這個 tree 被 flush 到 render 環境 - 例如，對於瀏覽器的應用程式，它被轉換成一組 DOM 的操作。當應用程式被更新時（通常經由 `setState`），會產生一個新的 tree。新的 tree 會與先前的 tree 做比較，計算出更新的應用程式所需要的操作。

儘管 Fiber 是 reconciler 徹底的重寫，在 [React 文件](https://facebook.github.io/react/docs/reconciliation.html)內 high-level 演算法的描述大致相同。關鍵點是：

-   假設不同的 component type 會 generate 實質上不同的 tree。React 不會嘗試去區分它們，而是完全替換舊的 tree。
-   列表（List）的區分是使用 key。Key 應該是「穩定、可預測而且唯一的。」
### Reconciliation 與 rendering

DOM 只是 React 可以 render 的環境之一，其他主要是透過 React Native render 到 native iOS 和 Android view。（這也是為什麼「Virtual DOM」有點用詞不當的原因）

它可以支援這麼多 target 原因是 React 被設計為 reconciliation 和 render 是獨立的階段。Reconciler 完成計算 tree 的哪些部分已經被改變的工作；Renderer 使用該實際的資訊來更新整個應用程式。

這種分離代表 React DOM 和 React Native 可以使用它們本身的 renderer 同時共享由 React core 所提供的相同 reconciler。

Fiber 重新實作了 reconciler。雖然 renderer 將需要更改並支援（並且利用）新的架構，但它與主要的 render 無關。
### Scheduling

- **_scheduling_** - 確定何時應執行工作的 process。
- **_work_** - 必須執行的任何計算。工作通常是更新的結果（例如 `setState`）。

React 的 [Design Principles](https://facebook.github.io/react/contributing/design-principles.html#scheduling) 文件在這個主題上非常出色，我在這裡引用一下：

> In its current implementation React walks the tree recursively and calls render functions of the whole updated tree during a single tick. However in the future it might start delaying some updates to avoid dropping frames.
>
> This is a common theme in React design. Some popular libraries implement the "push" approach where computations are performed when the new data is available. React, however, sticks to the "pull" approach where computations can be delayed until necessary.
>
> React is not a generic data processing library. It is a library for building user interfaces. We think that it is uniquely positioned in an app to know which computations are relevant right now and which are not.If something is offscreen, we can delay any logic related to it.
>
> If data is arriving faster than the frame rate, we can coalesce and batch updates. We can prioritize work coming from user interactions (such as an animation caused by a button click) over less important background work (such as rendering new content just loaded from the network) to avoid dropping frames.

關鍵點在於：

-   在 UI 中，不必立即 apply 每個更新。實際上，這樣做可能是浪費的，導致 frame 下降並降低使用者體驗。
-   不同 type 的更新具有不同的優先等級 - 一個 animation 更新需要比一個資料儲存的更新更快的完成。
-   一個基於 push 的方法要求應用程式（你，程式開發人員）來決定如何 scheduling 工作。一個基於 pull 的方法允許框架（React）變得聰明，為你做出決定。

目前 React 並沒有充分利用 schedule 的優勢。在一個 subtree 中，一個更新結果導致立即 re-render 整個 subtree。React 的核心演算法利用 scheduling 是 Fiber 背後的驅動想法。

* * *

現在我們準備要更深入 Fiber 的實作。下一個章節會比到目前為止我們討論的內容更具技術性，在開始前請確保你滿意先前的內容。
## 什麼是 Fiber？

我們將討論 React Fiber 架構的核心。Fiber 是比應用程式開發人員認為的更底層的抽象。如果你在嘗試理解它時感到沮喪，請不要灰心。持續嘗試，最終將變得更有意義（當你最後了解它時，請建議如何改善此章節。）

讓我們開始吧！

* * *

我們已經確定 Fiber 的主要目標是讓 React 能利用 scheduling 的優勢。具體來說，我們需要能夠：

- 暫停任務後能恢復原本的工作。
- 分配優先等級給不同類型的工作。
- 重新使用先前已完成的工作。
- 如果不再需要，則可以終止工作。

為了要做到這點，首先我們需要將工作拆分成多個單元的方法。從某種意義上來說，這就是 fiber。一個 fiber 帶有一個工作的單位。

更進一步，讓我們回頭看 [React component 作為一個資料函式](https://github.com/reactjs/react-basic#transformation)的概念，通常表示為：

```
v = f(d)
```

因此，render 一個 React 應用程式類似於呼叫一個函式，該函式主體包含對其他函式的呼叫。這個比喻對於思考 fiber 非常有幫助。

計算機通常使用 [call stack](https://en.wikipedia.org/wiki/Call_stack) 來追蹤程式的執行。當一個函式被執行，新的 **stack frame** 被加入到 call stack 中。Stack frame 表示該函式執行的工作。

在處理 UI 時，問題在於如果一次執行太多的工作，可能會造成動畫掉幀並且看起來斷斷續續的。而且，如果隨之而來的更新取代了某些工作，則這些工作可能是不必要的。這是 UI component 和函式的比較不同的地方，與一般函式相比，component 有更多具體的關注點。

較新的瀏覽器（和 React Native）實作有助於解決這個確切問題的 API：`requestIdleCallback` schedule 在空閒期間呼叫低優先級的函式，而 `requestAnimationFrame` schedule 在下一個 animation frame 上呼叫高優先權的函式。問題在於，為了使用這些 API，你需要一種將 render work 分解成 incremental unit。如果你僅依賴 call stack，它將持續工作直到 stack 為空。

如果我們可以自定義 call stack 的行為來最佳化呈現 UI，那不是很棒嗎？

這就是 React Fiber 的目的。Fiber 是重新實作了 stack，專門用於 React component。你可以把一個 fiber 視為一個 **virtual stack frame**。

重新實作 stack 的優點是，你可以[保留 stack frame 在記憶體](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)並根據需要（或*任何時候*）執行他們。這對於實現我們的 scheduling 目標至關重要。

除了 scheduling 之外，手動處理 stack frame 還可以釋放像是 concurrency 或 error boundary。我們將在後續的章節介紹這些主題。

在下一節中，我們將研究更多的 Fiber 結構。
### Fiber 的結構

_注意：隨著我們對實作的細節更加具體，某些事情的改變可能性增加了。如果發現任何錯誤或過時的訊息，請提交 PR。_

具體來說，Fiber 是一個 JavaScript 的 object，其中包含有關 component，其輸入和其輸出的資訊。

Fiber 對應於一個 stack frame，但它也對應於一個 component 的 instance。

這裡是一些 Fiber 重要的 field（這個清單並不完整。）
### `type` 和 `key`

Type 和 key 在 Fiber 中的用途與 React element 中的用途相同。（事實上，當從一個 element 建立 Fiber，這兩個 field 是直接拷貝過來的。）

Fiber 的 type 描述它對應的 component。對於 composite component，type 是 function 或是 class component 本身。對於 host component（`div`、`span` 等等）的 type 是一個 string。

概念上來說，type 是執行過程被 stack frame 追蹤的 function（如 `v = f(d)` ）。

伴隨著 type，key 在 reconciliation 被用來決定 Fiber 是否可被重複使用。
### `child` 和 `sibling`

這些 field 指向其他的 fiber，描述 Fiber 的 recursive tree 結構。

Child Fiber 對應於 component 的 `render` 方法回傳的值。所以在以下的範例中：

```jsx
function Parent() {
  return <Child />
}
```

`Parent` 的 Child Fiber 對應到 `Child` 。

Sibling field 說明了 `render` 回傳多個 children 的情況（Fiber 中的一項新功能！）：

```jsx
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

Child Fiber 表示一個 singly-linked list，它的 head 是第一個 child。所以在這個範例中，`Parent` 的 child 是 `Child1` ，而且 `Child1` 的 sibling 是 `Child2`。

回到我們的 function 比喻，你可以把 child Fiber 思考作為一個 [tail-called function](https://en.wikipedia.org/wiki/Tail_call)。
### `return`

Return Fiber 是指 program 處理完目前 Fiber 後應該回傳的 Fiber。它的概念上像是 stack frame 回傳 address。也可以認為是 parent Fiber。

如果 Fiber 有多個 child Fiber，每個 child Fiber 的 return Fiber 就是 parent Fiber。因此在我們前一個章節的範例中，`Child` 和 `Child2` 的 return Fiber 是 `Parent`。
### `pendingProps` 和 `memoizedProps`

概念上來說，props 是 function 的 argument。`pendingProps` 是 fiber 開始執行時的 set，而 `memoizedProps` 則是 Fiber 執行結束後的 set。

當傳入的 `pendingProps` 相等於 `memoizeProps`，意味著 Fiber 先前的 output 可以被 reuse，避免不必要的 work。
### `pendingWorkPriority`

數字說明了 Fiber 的工作優先順序。[ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) module 列出了不同的優先順序以及其代表的含義。

除了例外的 `NoWork` 是 0 之外，越大的數字說明了是越低的優先權。例如，你可以使用以下 function 來檢查 Fiber 的優先級是否至少與給定級別一樣高：

```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

_這個 function 只是用來說明；它不是 React Fiber codebase 實際的部分。_

Scheduler 使用優先順序的 field 去搜尋下一個需要被執行的工作單元。這個演算法將會在未來的章節被討論到。
### `alternate`

**_flush_** - Flush 一個 Fiber 就是把它的輸出 render 到螢幕上。

_**work-in-progress**_ - 一個還沒完成的 fiber；概念上就是一個 stack frame 還沒有被回傳。

在任何時候，一個 component instance 最多對應到兩個 Fiber：目前被 flush 的 Fiber 以及 work-in-progress 的 Fiber。

目前 Fiber 的 alternate 是 work-in-progress，而 work-in-progress 的 alternate 是目前的 Fiber，兩者互相交替。

Fiber 的交替過程是使用一個叫做 `cloneFiber` 的 function 延遲建立的。相對於總是建立一個相 object，`cloneFiber` 會嘗試去 reuse 已存在的 Fiber，最小化分配。

你應該將 `alternate` field 是為實作的細節，它經常在 codebase 出現所以在這裡有討論的價值。
### `output`

_**host component**_ - React 應用程式的 leaf node。它們 render 在特定的環境（例如：在瀏覽器，它們是 `div`、`span` 等等）。在 JSX，它們使用小寫 tag 名稱表示。

概念上說，Fiber 輸出的是 function 的回傳值。

每個 Fiber 最終都有輸出，但是輸出僅由 **hosot component** 上的 leaf node 被建立。輸出會接著被轉移到 tree 上。

輸出的內容最終會給到 renderer，所以它可以 flush 改變到 rendering environment。輸入如何被建立和更新是 renderer 的職責。
## 未來章節

目前僅此而已，但是本文件還遠遠不夠完整。後續的章節將描述在更新的整個生命週期中使用的演算法。涵蓋的主題包括：

- Scheduler 如何找到下一個工作單元去執行。
- 優先順序是如何在 Fiber tree 被追蹤和傳播。
- Scheduler 如何知道哪時候該要暫停或是恢復工作。
- 工作如何被 flush 和被標記成完成。
- Side-effect（像是生命週期方法）是如何工作的。
- 什麼是 coroutine 以及它是如何被用來實作像是 context 以及 layout 的功能。
## 相關影片

- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)
