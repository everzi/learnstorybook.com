---
title: '连接数据'
tocTitle: '数据'
description: '了解如何将数据连接到你的UI组件'
commit: 'c70ec15'
---

到目前为止，我们已经创建了独立的无状态组件——这对于 Storybook 很有用，但最终要在应用中赋予它们一些数据才有用。

这篇教程不专注于构建应用程序的细节，因此我们不会深入探讨这些内容。但我们会花一点时间来看一下将数据接入到连接的组件中的一种常见模式。

## 连接的组件

我们当前编写的 `TaskList` 组件是“展示型”的，因为它不与自身实现之外的任何外部内容交互。我们需要将其连接到数据提供者，以获取数据。

本示例使用 [Redux Toolkit](https://redux-toolkit.js.org/)，这是使用 [Redux](https://redux.js.org/) 开发用于存储数据的应用程序最有效的工具集，来为我们的应用构建一个简单的数据模型。不过，这里使用的模式同样适用于其他数据管理库，如 [Apollo](https://www.apollographql.com/client/) 和 [MobX](https://mobx.js.org/)。

将必要的依赖项添加到你的项目中：

```shell
yarn add @reduxjs/toolkit react-redux
```

首先，我们将在 `src/lib` 目录中的一个名为 `store.js` 的文件中构建一个简单的 Redux store，它响应更改任务状态的操作（为了保持简洁）：

```js:title=src/lib/store.js
/* A simple redux store/actions/reducer implementation.
 * A true app would be more complex and separated into different files.
 */
import { configureStore, createSlice } from '@reduxjs/toolkit';

/*
 * The initial state of our store when the app loads.
 * Usually, you would fetch this from a server. Let's not worry about that now
 */
const defaultTasks = [
  { id: '1', title: 'Something', state: 'TASK_INBOX' },
  { id: '2', title: 'Something more', state: 'TASK_INBOX' },
  { id: '3', title: 'Something else', state: 'TASK_INBOX' },
  { id: '4', title: 'Something again', state: 'TASK_INBOX' },
];
const TaskBoxData = {
  tasks: defaultTasks,
  status: 'idle',
  error: null,
};

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

然后，我们将更新 `TaskList` 组件，使其连接到 Redux store 并渲染我们感兴趣的任务：

```jsx:title=src/components/TaskList.jsx
import Task from './Task';

import { useDispatch, useSelector } from 'react-redux';

import { updateTaskState } from '../lib/store';

export default function TaskList() {
  // We're retrieving our state from the store
  const tasks = useSelector((state) => {
    const tasksInOrder = [
      ...state.taskbox.tasks.filter((t) => t.state === 'TASK_PINNED'),
      ...state.taskbox.tasks.filter((t) => t.state !== 'TASK_PINNED'),
    ];
    const filteredTasks = tasksInOrder.filter(
      (t) => t.state === 'TASK_INBOX' || t.state === 'TASK_PINNED'
    );
    return filteredTasks;
  });

  const { status } = useSelector((state) => state.taskbox);

  const dispatch = useDispatch();

  const pinTask = (value) => {
    // We're dispatching the Pinned event back to our store
    dispatch(updateTaskState({ id: value, newTaskState: 'TASK_PINNED' }));
  };
  const archiveTask = (value) => {
    // We're dispatching the Archive event back to our store
    dispatch(updateTaskState({ id: value, newTaskState: 'TASK_ARCHIVED' }));
  };
  const LoadingRow = (
    <div className="loading-item">
      <span className="glow-checkbox" />
      <span className="glow-text">
        <span>Loading</span> <span>cool</span> <span>state</span>
      </span>
    </div>
  );
  if (status === 'loading') {
    return (
      <div className="list-items" data-testid="loading" key={"loading"}>
        {LoadingRow}
        {LoadingRow}
        {LoadingRow}
        {LoadingRow}
        {LoadingRow}
        {LoadingRow}
      </div>
    );
  }
  if (tasks.length === 0) {
    return (
      <div className="list-items" key={"empty"} data-testid="empty">
        <div className="wrapper-message">
          <span className="icon-check" />
          <p className="title-message">You have no tasks</p>
          <p className="subtitle-message">Sit back and relax</p>
        </div>
      </div>
    );
  }

  return (
    <div className="list-items" data-testid="success" key={"success"}>
      {tasks.map((task) => (
        <Task
          key={task.id}
          task={task}
          onPinTask={(task) => pinTask(task)}
          onArchiveTask={(task) => archiveTask(task)}
        />
      ))}
    </div>
  );
}
```

现在，我们已经从 Redux store 获取了一些实际数据来填充我们的组件，我们可以将其连接到 `src/App.js` 并在其中渲染组件。但目前先不这样做，继续我们的组件驱动之旅。

不用担心，我们会在下一章处理这个问题。

## 使用装饰器提供上下文

由于我们的`TaskList` 现在是一个连接组件，依赖于 Redux store 来获取和更新任务，因此我们的 Storybook stories 在这个更改后停止工作了。

![Broken tasklist](/intro-to-storybook/broken-tasklist-7-0-optimized.png)

我们可以使用多种方法来解决这个问题。尽管如此，由于我们的应用程序相对简单，我们可以依赖装饰器，类似于我们在[上一章](/intro-to-storybook/react/en/composite-component)中所做的，在 Storybook stories 中提供一个模拟的 store：

```jsx:title=src/components/TaskList.stories.jsx
import TaskList from './TaskList';

import * as TaskStories from './Task.stories';

import { Provider } from 'react-redux';

import { configureStore, createSlice } from '@reduxjs/toolkit';

// A super-simple mock of the state of the store
export const MockedState = {
  tasks: [
    { ...TaskStories.Default.args.task, id: '1', title: 'Task 1' },
    { ...TaskStories.Default.args.task, id: '2', title: 'Task 2' },
    { ...TaskStories.Default.args.task, id: '3', title: 'Task 3' },
    { ...TaskStories.Default.args.task, id: '4', title: 'Task 4' },
    { ...TaskStories.Default.args.task, id: '5', title: 'Task 5' },
    { ...TaskStories.Default.args.task, id: '6', title: 'Task 6' },
  ],
  status: 'idle',
  error: null,
};

// A super-simple mock of a redux store
const Mockstore = ({ taskboxState, children }) => (
  <Provider
    store={configureStore({
      reducer: {
        taskbox: createSlice({
          name: 'taskbox',
          initialState: taskboxState,
          reducers: {
            updateTaskState: (state, action) => {
              const { id, newTaskState } = action.payload;
              const task = state.tasks.findIndex((task) => task.id === id);
              if (task >= 0) {
                state.tasks[task].state = newTaskState;
              }
            },
          },
        }).reducer,
      },
    })}
  >
    {children}
  </Provider>
);

export default {
  component: TaskList,
  title: 'TaskList',
  decorators: [(story) => <div style={{ margin: '3rem' }}>{story()}</div>],
  tags: ['autodocs'],
  excludeStories: /.*MockedState$/,
};

export const Default = {
  decorators: [
    (story) => <Mockstore taskboxState={MockedState}>{story()}</Mockstore>,
  ],
};

export const WithPinnedTasks = {
  decorators: [
    (story) => {
      const pinnedtasks = [
        ...MockedState.tasks.slice(0, 5),
        { id: '6', title: 'Task 6 (pinned)', state: 'TASK_PINNED' },
      ];

      return (
        <Mockstore
          taskboxState={{
            ...MockedState,
            tasks: pinnedtasks,
          }}
        >
          {story()}
        </Mockstore>
      );
    },
  ],
};

export const Loading = {
  decorators: [
    (story) => (
      <Mockstore
        taskboxState={{
          ...MockedState,
          status: 'loading',
        }}
      >
        {story()}
      </Mockstore>
    ),
  ],
};

export const Empty = {
  decorators: [
    (story) => (
      <Mockstore
        taskboxState={{
          ...MockedState,
          tasks: [],
        }}
      >
        {story()}
      </Mockstore>
    ),
  ],
};
```

<div class="aside">

💡 `excludeStories` 是一个 Storybook 配置字段，用于防止我们的模拟状态被当作一个 story 处理。你可以在 [Storybook 文档](https://storybook.js.org/docs/api/csf) 中了解更多关于这个字段的信息。

</div>

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/finished-tasklist-states-7-0-optimized.mp4"
    type="video/mp4"
  />
</video>

<div class="aside">
💡 别忘了用 git 提交你的更改！
</div>

成功！我们回到了最初的位置，我们的 Storybook 现在可以正常工作，并且我们能够看到如何将数据提供给连接组件。在下一章中，我们将利用这里学到的知识和把它应用到一个屏幕上。
