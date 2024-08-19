---
title: '组装复合组件'
tocTitle: '复合组件'
description: '从较为简单的组件组装复合组件'
commit: '429780a'
---

上一章中，我们构建了第一个组件；本章将扩展我们学到的内容，创建一个 TaskList，任务列表。让我们将组件组合在一起，看看在引入更多复杂性时会发生什么。

## Tasklist 组件

Taskbox 通过将固定的任务置于默认任务之上来强调它们。这会产生 `TaskList` 的两种变体：默认任务和固定任务。你需要为这两种情况创建 stories。

![default and pinned tasks](/intro-to-storybook/tasklist-states-1.png)

由于 `Task` 数据可能会异步发送，我们**还**需要一个加载状态，以便在没有连接时渲染。此外，当没有任务时，我们还需要一个空状态。

![empty and loading tasks](/intro-to-storybook/tasklist-states-2.png)

## 做好准备

复合组件与其包含的基本组件并没有太大区别。创建一个 `TaskList` 组件以及一个对应的 story 文件：`src/components/TaskList.jsx` 和 `src/components/TaskList.stories.jsx`。

从粗略实现 `TaskList` 开始。你需要导入之前的 `Task` 组件，并将属性和操作作为输入传递给它。

```jsx:title=src/components/TaskList.jsx
import Task from './Task';

export default function TaskList({ loading, tasks, onPinTask, onArchiveTask }) {
  const events = {
    onPinTask,
    onArchiveTask,
  };

  if (loading) {
    return <div className="list-items">loading</div>;
  }

  if (tasks.length === 0) {
    return <div className="list-items">empty</div>;
  }

  return (
    <div className="list-items">
      {tasks.map(task => (
        <Task key={task.id} task={task} {...events} />
      ))}
    </div>
  );
}
```

接下来，在 story 文件中为 `TaskList` 创建测试状态。

```jsx:title=src/components/TaskList.stories.jsx
import TaskList from './TaskList';

import * as TaskStories from './Task.stories';

export default {
  component: TaskList,
  title: 'TaskList',
  decorators: [(story) => <div style={{ margin: '3rem' }}>{story()}</div>],
  tags: ['autodocs'],
  args: {
    ...TaskStories.ActionsData,
  },
};

export const Default = {
  args: {
    // Shaping the stories through args composition.
    // The data was inherited from the Default story in Task.stories.jsx.
    tasks: [
      { ...TaskStories.Default.args.task, id: '1', title: 'Task 1' },
      { ...TaskStories.Default.args.task, id: '2', title: 'Task 2' },
      { ...TaskStories.Default.args.task, id: '3', title: 'Task 3' },
      { ...TaskStories.Default.args.task, id: '4', title: 'Task 4' },
      { ...TaskStories.Default.args.task, id: '5', title: 'Task 5' },
      { ...TaskStories.Default.args.task, id: '6', title: 'Task 6' },
    ],
  },
};

export const WithPinnedTasks = {
  args: {
    tasks: [
      ...Default.args.tasks.slice(0, 5),
      { id: '6', title: 'Task 6 (pinned)', state: 'TASK_PINNED' },
    ],
  },
};

export const Loading = {
  args: {
    tasks: [],
    loading: true,
  },
};

export const Empty = {
  args: {
    // Shaping the stories through args composition.
    // Inherited data coming from the Loading story.
    ...Loading.args,
    loading: false,
  },
};
```

<div class="aside">

💡[**装饰器**](https://storybook.js.org/docs/writing-stories/decorators) 是为 stories 提供任意包装的一种方式。在本例中，我们使用默认导出的装饰器键来为渲染的组件周围添加一些 `margin`。装饰器还可以用于将 stories 包装在“提供者”中，即设置 React 上下文的库组件。

</div>

通过导入 `TaskStories`，我们能够轻松地[组合](https://storybook.js.org/docs/writing-stories/args#args-composition)stories 中的参数（简称 args）。这样，两个组件所期望的数据和操作（模拟回调）都得以保留。

现在检查 Storybook 是否有新的 `TaskList` stories。

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/inprogress-tasklist-states-7-0.mp4"
    type="video/mp4"
  />
</video>

## 构建状态

我们的组件仍然很粗糙，但现在我们对将要实现的 stories 有了一个概念。你可能会觉得 `.list-items` 包装器过于简单。没错——在大多数情况下，我们不会创建一个新组件，仅仅是添加一个包装器而已。但 `TaskList` 组件的**真正复杂性**体现在边界情况 `withPinnedTasks`、`loading` 和 `empty` 上。

```jsx:title=src/components/TaskList.jsx
import Task from './Task';

export default function TaskList({ loading, tasks, onPinTask, onArchiveTask }) {
  const events = {
    onPinTask,
    onArchiveTask,
  };
  const LoadingRow = (
    <div className="loading-item">
      <span className="glow-checkbox" />
      <span className="glow-text">
        <span>Loading</span> <span>cool</span> <span>state</span>
      </span>
    </div>
  );
  if (loading) {
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

  const tasksInOrder = [
    ...tasks.filter((t) => t.state === 'TASK_PINNED'),
    ...tasks.filter((t) => t.state !== 'TASK_PINNED'),
  ];
  return (
    <div className="list-items">
      {tasksInOrder.map((task) => (
        <Task key={task.id} task={task} {...events} />
      ))}
    </div>
  );
}
```

添加的标记生成了以下 UI：

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/finished-tasklist-states-7-0.mp4"
    type="video/mp4"
  />
</video>

注意列表中固定项的位置。我们希望固定项渲染在列表顶部，以便将其优先展示给用户。

## 数据要求和属性

随着组件的扩展，输入要求也会增加。定义 `TaskList` 的属性要求。由于 `Task` 是子组件，请确保提供正确的数据格式以便渲染它。为了节省时间和避免麻烦，重用你之前在 `Task` 中定义的 `propTypes`。

```diff:title=src/components/TaskList.jsx
+ import PropTypes from 'prop-types';

import Task from './Task';

export default function TaskList({ loading, tasks, onPinTask, onArchiveTask }) {
  const events = {
    onPinTask,
    onArchiveTask,
  };
  const LoadingRow = (
    <div className="loading-item">
      <span className="glow-checkbox" />
      <span className="glow-text">
        <span>Loading</span> <span>cool</span> <span>state</span>
      </span>
    </div>
  );
  if (loading) {
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

  const tasksInOrder = [
    ...tasks.filter((t) => t.state === 'TASK_PINNED'),
    ...tasks.filter((t) => t.state !== 'TASK_PINNED'),
  ];
  return (
    <div className="list-items">
      {tasksInOrder.map((task) => (
        <Task key={task.id} task={task} {...events} />
      ))}
    </div>
  );
}

+ TaskList.propTypes = {
+  /** Checks if it's in loading state */
+  loading: PropTypes.bool,
+  /** The list of tasks */
+  tasks: PropTypes.arrayOf(Task.propTypes.task).isRequired,
+  /** Event to change the task to pinned */
+  onPinTask: PropTypes.func,
+  /** Event to change the task to archived */
+  onArchiveTask: PropTypes.func,
+ };
+ TaskList.defaultProps = {
+  loading: false,
+ };
```

<div class="aside">
💡 别忘了用 git 提交你的更改！
</div>
