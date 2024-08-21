---
title: '构建一个页面'
tocTitle: '页面'
description: '用组件构建一个页面'
commit: '2275632'
---

我们专注于从下到上构建 UI，从小处开始，逐步增加复杂性。这使我们能够在隔离的环境中开发每个组件，弄清楚它的数据需求，并在 Storybook 中进行测试。所有这些都不需要启动服务器或构建屏幕！

在本章中，我们通过将组件组合到一个界面中并在 Storybook 中开发该界面，进一步提升复杂性。

## 连接页面

由于我们的应用程序相对简单，我们要构建的界面也很容易，它只是从远程 API 获取数据，封装 `TaskList` 组件（该组件从 Redux 提供自己的数据），并从 Redux 中提取顶层的 `error` 字段。

我们将首先更新 Redux 存储（位于 `src/lib/store.js`），以连接到远程 API，并处理应用程序的各种状态（例如 `error`、`succeeded`）：

```diff:title=src/lib/store.js
/* A simple redux store/actions/reducer implementation.
 * A true app would be more complex and separated into different files.
 */
import {
  configureStore,
  createSlice,
+ createAsyncThunk,
} from '@reduxjs/toolkit';

/*
 * The initial state of our store when the app loads.
 * Usually, you would fetch this from a server. Let's not worry about that now
 */

const TaskBoxData = {
  tasks: [],
  status: 'idle',
  error: null,
};

/*
 * Creates an asyncThunk to fetch tasks from a remote endpoint.
 * You can read more about Redux Toolkit's thunks in the docs:
 * https://redux-toolkit.js.org/api/createAsyncThunk
 */
+ export const fetchTasks = createAsyncThunk('todos/fetchTodos', async () => {
+   const response = await fetch(
+     'https://jsonplaceholder.typicode.com/todos?userId=1'
+   );
+   const data = await response.json();
+   const result = data.map((task) => ({
+     id: `${task.id}`,
+     title: task.title,
+     state: task.completed ? 'TASK_ARCHIVED' : 'TASK_INBOX',
+   }));
+   return result;
+ });

/*
 * The store is created here.
 * You can read more about Redux Toolkit's slices in the docs:
 * https://redux-toolkit.js.org/api/createSlice
 */
const TasksSlice = createSlice({
  name: 'taskbox',
  initialState: TaskBoxData,
  reducers: {
    updateTaskState: (state, action) => {
      const { id, newTaskState } = action.payload;
      const task = state.tasks.findIndex((task) => task.id === id);
      if (task >= 0) {
        state.tasks[task].state = newTaskState;
      }
    },
  },
  /*
   * Extends the reducer for the async actions
   * You can read more about it at https://redux-toolkit.js.org/api/createAsyncThunk
   */
+  extraReducers(builder) {
+    builder
+    .addCase(fetchTasks.pending, (state) => {
+      state.status = 'loading';
+      state.error = null;
+      state.tasks = [];
+    })
+    .addCase(fetchTasks.fulfilled, (state, action) => {
+      state.status = 'succeeded';
+      state.error = null;
+      // Add any fetched tasks to the array
+      state.tasks = action.payload;
+     })
+    .addCase(fetchTasks.rejected, (state) => {
+      state.status = 'failed';
+      state.error = "Something went wrong";
+      state.tasks = [];
+    });
+ },
});

// The actions contained in the slice are exported for usage in our components
export const { updateTaskState } = TasksSlice.actions;

/*
 * Our app's store configuration goes here.
 * Read more about Redux's configureStore in the docs:
 * https://redux-toolkit.js.org/api/configureStore
 */
const store = configureStore({
  reducer: {
    taskbox: TasksSlice.reducer,
  },
});

export default store;
```

现在，我们已经更新了存储，以从远程 API 端点获取数据，并准备好处理应用程序的各种状态，接下来让我们在 `src/components` 目录中创建 `InboxScreen.jsx` 文件：

```jsx:title=src/components/InboxScreen.jsx
import { useEffect } from 'react';

import { useDispatch, useSelector } from 'react-redux';

import { fetchTasks } from '../lib/store';

import TaskList from './TaskList';

export default function InboxScreen() {
  const dispatch = useDispatch();
  // We're retrieving the error field from our updated store
  const { error } = useSelector((state) => state.taskbox);
  // The useEffect triggers the data fetching when the component is mounted
  useEffect(() => {
    dispatch(fetchTasks());
  }, []);

  if (error) {
    return (
      <div className="page lists-show">
        <div className="wrapper-message">
          <span className="icon-face-sad" />
          <p className="title-message">Oh no!</p>
          <p className="subtitle-message">Something went wrong</p>
        </div>
      </div>
    );
  }
  return (
    <div className="page lists-show">
      <nav>
        <h1 className="title-page">Taskbox</h1>
      </nav>
      <TaskList />
    </div>
  );
}
```

我们还需要修改 `App` 组件以渲染 `InboxScreen`（最终，我们会使用路由来选择正确的界面，但这里暂时不用担心这个问题）：

```diff:title=src/App.jsx
- import { useState } from 'react'
- import reactLogo from './assets/react.svg'
- import viteLogo from '/vite.svg'
- import './App.css'

+ import './index.css';
+ import store from './lib/store';

+ import { Provider } from 'react-redux';
+ import InboxScreen from './components/InboxScreen';

function App() {
- const [count, setCount] = useState(0)
  return (
-   <div className="App">
-     <div>
-       <a href="https://vitejs.dev" target="_blank">
-         <img src={viteLogo} className="logo" alt="Vite logo" />
-       </a>
-       <a href="https://reactjs.org" target="_blank">
-         <img src={reactLogo} className="logo react" alt="React logo" />
-       </a>
-     </div>
-     <h1>Vite + React</h1>
-     <div className="card">
-       <button onClick={() => setCount((count) => count + 1)}>
-         count is {count}
-       </button>
-       <p>
-         Edit <code>src/App.jsx</code> and save to test HMR
-       </p>
-     </div>
-     <p className="read-the-docs">
-       Click on the Vite and React logos to learn more
-     </p>
-   </div>
+   <Provider store={store}>
+     <InboxScreen />
+   </Provider>
  );
}
export default App;
```

不过，有趣的地方在于如何在 Storybook 中渲染。

正如我们之前所看到的，`TaskList` 组件现在是一个 **连接** 组件，依赖于 Redux 存储来渲染任务。由于我们的 `InboxScreen` 也是一个连接组件，我们将采取类似的方法，向 story 中提供一个存储。因此，当我们在 `InboxScreen.stories.jsx` 中设置我们的 stories 时：

```jsx:title=src/components/InboxScreen.stories.jsx
import InboxScreen from './InboxScreen';
import store from '../lib/store';

import { Provider } from 'react-redux';

export default {
  component: InboxScreen,
  title: 'InboxScreen',
  decorators: [(story) => <Provider store={store}>{story()}</Provider>],
  tags: ['autodocs'],
};

export const Default = {};

export const Error = {};
```

我们可以很快发现 `error` story 存在一个问题。它显示了一组任务，而不是正确的状态。回避这个问题的一种方法是为每种状态提供一个模拟版本，类似于我们在上一章中所做的。相反，我们将使用一个知名的 API 模拟库，并结合 Storybook 插件来帮助我们解决这个问题。

![Broken inbox screen state](/intro-to-storybook/broken-inbox-error-state-7-0-optimized.png)

## 模拟 API 服务

由于我们的应用程序相对简单且不过多依赖远程 API 调用，我们将使用 [Mock Service Worker](https://mswjs.io/) 和 [Storybook 的 MSW 插件](https://storybook.js.org/addons/msw-storybook-addon)。Mock Service Worker 是一个 API 模拟库，它依赖于服务工作者来捕获网络请求，并在响应中提供模拟数据。

在我们按照 [入门指南](/intro-to-storybook/react/en/get-started) 设置应用程序时，这两个包也已安装。我们只需配置它们并更新我们的 stories 使用它们即可。

在终端中运行以下命令，以在 `public` 文件夹中生成一个通用的服务工作者：

```shell
yarn init-msw
```

接下来，我们需要更新 `.storybook/preview.js` 并初始化它们：

```diff:title=.storybook/preview.js
import '../src/index.css';

// Registers the msw addon
+ import { initialize, mswLoader } from 'msw-storybook-addon';

// Initialize MSW
+ initialize();

//👇 Configures Storybook to log the actions( onArchiveTask and onPinTask ) in the UI.
/** @type { import('@storybook/react').Preview } */
const preview = {
  parameters: {
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/,
      },
    },
  },
+ loaders: [mswLoader],
};

export default preview;
```

最后，更新 `InboxScreen` 的 stories，并包含一个用于模拟远程 API 调用的 [参数](https://storybook.js.org/docs/writing-stories/parameters)：

```diff:title=src/components/InboxScreen.stories.jsx
import InboxScreen from './InboxScreen';

import store from '../lib/store';

+ import { http, HttpResponse } from 'msw';

+ import { MockedState } from './TaskList.stories';

import { Provider } from 'react-redux';

export default {
  component: InboxScreen,
  title: 'InboxScreen',
  decorators: [(story) => <Provider store={store}>{story()}</Provider>],
  tags: ['autodocs'],
};

export const Default = {
+ parameters: {
+   msw: {
+     handlers: [
+       http.get('https://jsonplaceholder.typicode.com/todos?userId=1', () => {
+         return HttpResponse.json(MockedState.tasks);
+       }),
+     ],
+   },
+ },
};

export const Error = {
+ parameters: {
+   msw: {
+     handlers: [
+       http.get('https://jsonplaceholder.typicode.com/todos?userId=1', () => {
+         return new HttpResponse(null, {
+           status: 403,
+         });
+       }),
+     ],
+   },
+ },
};
```

<div class="aside">

💡 顺便说一下，通过层级传递数据是一种合理的方法，尤其是在使用 [GraphQL](http://graphql.org/) 时。这也是我们如何与 800+ stories 一起构建 [Chromatic](https://www.chromatic.com/?utm_source=storybook_website&utm_medium=link&utm_campaign=storybook) 的方式。

</div>

检查你的 Storybook，你会发现 `error` story 现在按预期工作了。MSW 截获了我们的远程 API 调用，并提供了相应的响应。

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/inbox-screen-with-working-msw-addon-optimized-7.0.mp4"
    type="video/mp4"
  />
</video>

## 交互测试

到目前为止，我们已经能够从零开始构建一个功能齐全的应用程序，从一个简单的组件到一个界面，并使用我们的 stories 不断测试每个更改。但每个新 story 还需要手动检查所有其他 stories ，以确保 UI 没有问题。这是了不少额外的工作量。

难道我们不能自动化这个流程，并自动测试组件的交互吗？

### 使用 `play` 函数编写一个交互测试。

Storybook 的 [`play`](https://storybook.js.org/docs/writing-stories/play-function) 函数和 [`@storybook/addon-interactions`](https://storybook.js.org/docs/writing-tests/interaction-testing) 可以帮助我们实现这一点。一个 play 函数包含在 story 渲染后运行的小段代码。

play 函数帮助我们验证在任务更新时 UI 会发生什么。它使用与框架无关的 DOM API，这意味着无论使用何种前端框架，我们可以使用 play 函数编写 stories，以与 UI 交互并模拟人类行为。

`@storybook/addon-interactions` 帮助我们在 Storybook 中可视化测试，提供了循序渐进的流程。它还提供了一套方便的 UI 控件集，用于暂停、恢复、倒退和逐步查看每个交互。

让我们来看看它的实际效果！更新你新创建的 `InboxScreen` story，并通过添加以下内容来设置组件交互：

```diff:title=src/components/InboxScreen.stories.jsx
import InboxScreen from './InboxScreen';

import store from '../lib/store';

import { http, HttpResponse } from 'msw';

import { MockedState } from './TaskList.stories';

import { Provider } from 'react-redux';

+ import {
+  fireEvent,
+  waitFor,
+  within,
+  waitForElementToBeRemoved
+ } from '@storybook/test';

export default {
  component: InboxScreen,
  title: 'InboxScreen',
  decorators: [(story) => <Provider store={store}>{story()}</Provider>],
  tags: ['autodocs'],
};

export const Default = {
  parameters: {
    msw: {
      handlers: [
        http.get('https://jsonplaceholder.typicode.com/todos?userId=1', () => {
          return HttpResponse.json(MockedState.tasks);
        }),
      ],
    },
  },
+ play: async ({ canvasElement }) => {
+   const canvas = within(canvasElement);
+   // Waits for the component to transition from the loading state
+   await waitForElementToBeRemoved(await canvas.findByTestId('loading'));
+   // Waits for the component to be updated based on the store
+   await waitFor(async () => {
+     // Simulates pinning the first task
+     await fireEvent.click(canvas.getByLabelText('pinTask-1'));
+     // Simulates pinning the third task
+     await fireEvent.click(canvas.getByLabelText('pinTask-3'));
+   });
+ },
};

export const Error = {
  parameters: {
    msw: {
      handlers: [
        http.get('https://jsonplaceholder.typicode.com/todos?userId=1', () => {
          return new HttpResponse(null, {
            status: 403,
          });
        }),
      ],
    },
  },
};
```

<div class="aside">

💡 `@storybook/test` 包替代了 `@storybook/jest` 和 `@storybook/testing-library` 测试包，提供了更小的包体积和更简洁的 API，基于 [Vitest](https://vitest.dev/) 包。

</div>

检查 `Default` story。点击 `Interactions` 面板，以查看 story 的 play 函数中的交互列表。

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/storybook-interactive-stories-play-function-7-0.mp4"
    type="video/mp4"
  />
</video>

### 使用测试运行器自动化测试

通过使用 Storybook 的 play 函数，我们能够绕过问题，允许我们与 UI 进行交互，并快速检查它如何响应如果我们更新任务时——这样可以保持 UI 的一致性，而无需额外的手动工作。

但是，如果我们仔细查看 Storybook，会发现它仅在查看故事时运行交互测试。因此，如果我们做了更改，仍然需要逐一查看每个 story 来运行所有检查。我们可以自动化这个过程吗？

好消息是我们可以做到！Storybook 的 [测试运行器](https://storybook.js.org/docs/writing-tests/test-runner) 允许我们这么做。它是一个独立的工具，由 [Playwright](https://playwright.dev/) 提供支持，可以运行我们所有的交互测试并捕捉出问题的 stories。

让我们看看它如何工作！运行以下命令来安装它：

```shell
yarn add --dev @storybook/test-runner
```

接下来，更新你 `package.json` 文件中的 `scripts`，并添加一个新的测试任务：

```json:clipboard=false
{
  "scripts": {
    "test-storybook": "test-storybook"
  }
}
```

最后，在 Storybook 运行的情况下，打开一个新的终端窗口并运行以下命令：

```shell
yarn test-storybook --watch
```

<div class="aside">

💡 使用 play 函数进行交互测试是测试 UI 组件的绝佳方式。它的功能远不止我们在这里看到的这些；建议你阅读 [官方文档](https://storybook.js.org/docs/writing-tests/interaction-testing) 以了解更多内容。

如果想深入了解测试内容，可以查看 [测试手册](/ui-testing-handbook)。它涵盖了大规模前端团队使用的测试策略，可以大大提升你的开发流程。

</div>

![Storybook test runner successfully runs all tests](/intro-to-storybook/storybook-test-runner-execution.png)

成功了！现在我们有一个工具，可以帮助我们自动验证所有 stories 是否无错误地渲染，并且所有断言是否通过。此外，如果测试失败，它会提供一个链接，点击即可在浏览器中打开失败的 story。

## 组件驱动开发

我们从 `Task` 组件开始，然后发展到 `TaskList`，现在我们这里有了一个完整的屏幕 UI。我们的 `InboxScreen` 包含了连接的组件，并包括了相应的 stories。

<video autoPlay muted playsInline loop style="width:480px; height:auto; margin: 0 auto;">
  <source
    src="/intro-to-storybook/component-driven-development-optimized.mp4"
    type="video/mp4"
  />
</video>

[**组件驱动开发**](https://www.componentdriven.org/) 允许你随着组件层级的上升逐步扩展复杂性。其优势包括更专注的开发过程和所有可能的 UI 变体的覆盖范围。简而言之，CDD 帮助你构建更高质量、更复杂的用户界面。

我们还没完成——UI 构建完成后工作还没结束。我们还需要确保它能够随着时间的推移保持耐用。

<div class="aside">
💡 别忘了用 git 提交你的更改！
</div>
