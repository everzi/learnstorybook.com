---
title: '构建一个简单的组件'
tocTitle: '简单组件'
description: '构建一个简单的独立的组件'
commit: '9b36e1a'
---

我们将采用[组件驱动开发](https://www.componentdriven.org/)（CDD）的方法来构建我们的 UI。这种方法“自下而上”的构建 UI，先从组件开始，最后形成完整页面。CDD 有助于你在构建 UI 时逐步应对扩展复杂性。

## 任务（Task）

![Task component in three states](/intro-to-storybook/task-states-learnstorybook.png)

`Task` 是我们应用程序的核心组件。每个任务的显示会有所不同，具体取决于他所处的状态。我们会显示一个选中（或未选中）的复选框、任务的一些信息以及一个“pin”按钮，并允许我们在列表中上下移动任务。结合这些需求，我们需要以下 props：

- `title` – 一个描述任务的字符串
- `state` – 判断当前任务在哪个列表中，以及是否已完成？

在开始构建 `Task` 时，我们首先编写对应于上面勾画的不同任务类型的测试状态。然后我们使用 Storybook 使用模拟数据创建独立的组件。在进行过程中，我们将在每种状态下手动测试组件的外观。

## 做好准备

首先，让我们创建任务组件及其附带的 story 文件：`src/components/Task.jsx` 和 `src/components/Task.stories.jsx`。

我们将从 `Task` 的基础实现开始，只接收我们已知需要的属性和你可以在任务中执行的两个操作（在列表之间移动任务）：

```jsx:title=src/components/Task.jsx
export default function Task({ task: { id, title, state }, onArchiveTask, onPinTask }) {
  return (
    <div className="list-item">
      <label htmlFor={`title-${id}`} aria-label={title}>
        <input type="text" value={title} readOnly={true} name="title" id={`title-${id}`} />
      </label>
    </div>
  );
}
```

上面，我们基于 Todos 应用的现有 HTML 结构直接渲染了 `Task` 的标记文本。

下面，我们在 story 文件中构建 Task 的三个测试状态：

```jsx:title=src/components/Task.stories.jsx
import { fn } from "@storybook/test";

import Task from './Task';

export const ActionsData = {
  onArchiveTask: fn(),
  onPinTask: fn(),
};

export default {
  component: Task,
  title: 'Task',
  tags: ['autodocs'],
  //👇 以“Data”结尾的导出不是stories。
  excludeStories: /.*Data$/,
  args: {
    ...ActionsData,
  },
};

export const Default = {
  args: {
    task: {
      id: '1',
      title: 'Test Task',
      state: 'TASK_INBOX',
    },
  },
};

export const Pinned = {
  args: {
    task: {
      ...Default.args.task,
      state: 'TASK_PINNED',
    },
  },
};

export const Archived = {
  args: {
    task: {
      ...Default.args.task,
      state: 'TASK_ARCHIVED',
    },
  },
};
```

<div class="aside">

💡 [**Actions**](https://storybook.js.org/docs/essentials/actions) 帮助你在构建独立的 UI 组件时验证交互。通常，你无法在应用上下文中访问你拥有的函数和状态。使用 `fn()` 来接入它们。

</div>

Storybook 中有两个基本的组织级别：组件和其 child stories。可以将每个 story 视为组件的一个变体。你可以根据需要为每个组件添加任意数量的 stories。

- **Component**
  - Story
  - Story
  - Story

为了告诉 Storybook 关于我们正在文档化和测试的组件，我们创建一个 `default` 导出，其中包含：

- `component` -- 组件本身
- `title` -- 如何在 Storybook 侧边栏中分组或分类组件
- `tags` -- 用于自动生成我们的组件文档
- `excludeStories` -- story 所需的额外信息，但不应在 Storybook 中渲染
- `args` -- 定义组件期望模拟自定义事件的操作[参数](https://storybook.js.org/docs/essentials/actions#action-args)

为了定义我们的 stories，我们将使用 Component Story Format 3（也称为 [CSF3](https://storybook.js.org/docs/api/csf)）来构建每个测试用例。这种格式旨在以简洁的方式构建每个测试用例。通过导出包含每个组件状态的对象，我们可以更直观地定义测试和更高效地编写和重用 stories。

参数或 [`args`](https://storybook.js.org/docs/writing-stories/args) 允许我们通过控制插件实时编辑组件，而无需重启 Storybook。一旦 [`args`](https://storybook.js.org/docs/writing-stories/args) 的值发生变化，组件也会随之更新。

`fn()` 允许我们创建一个在被点击时会出现在 Storybook UI 的 **Actions** 面板中的回调。因此，当我们构建一个固定按钮时，我们可以确定按钮点击是否在 UI 中成功。

由于我们需要将相同的操作集传递给组件的所有变体，将它们打包成一个 `ActionsData` 变量并每次传递到我们的 story 定义中是方便的。将组件需要的`ActionsData` 打包后的另一个优点是，你可以 `export` 它们，并在在复用该组件的组件 stories 中使用它们，如我们稍后将看到的那样。

在创建 story 时，我们使用一个基础 `task` 参数来构建组件期望的任务外形。通常根据实际数据的样子来建模。同样，导出此形状将使我们能够在后续 stories 中重用它，如我们将看到的那样。

## 配置

我们需要对 Storybook 的配置文件做几处更改，以便它能识别我们最近创建的 stories，并允许我们使用应用的 CSS 文件（位于 `src/index.css`）。

首先，将你的 Storybook 配置文件 (`.storybook/main.js`) 更改为以下内容：

```diff:title=.storybook/main.js
/** @type { import('@storybook/react-vite').StorybookConfig } */
const config = {
- stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
+ stories: ['../src/components/**/*.stories.@(js|jsx)'],
  staticDirs: ['../public'],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
};
export default config;
```

完成上述更改后，在 `.storybook` 文件夹内，将 `preview.js` 更改为以下内容：

```diff:title=.storybook/preview.js
+ import '../src/index.css';

// 配置 Storybook 在 UI 中记录操作（`onArchiveTask` 和 `onPinTask`）。
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
};

export default preview;
```

[`parameters`](https://storybook.js.org/docs/writing-stories/parameters) 通常用于控制 Storybook 功能和插件的行为。在我们的例子中，我们不会将它们用于此目的。相反，我们将导入应用程序的 CSS 文件。

完成这一步后，重新启动 Storybook 服务器，应该会生成三个 Task 状态的测试用例：

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/inprogress-task-states-7-0.mp4"
    type="video/mp4"
  />
</video>

## 构建状态

现在，我们已经设置了 Storybook，导入了样式，并构建了测试用例，我们可以迅速开始实现组件的 HTML，以匹配设计。

目前组件仍然是基础的。首先，编写实现设计的代码，但不需要涉及太多细节：

```jsx:title=src/components/Task.jsx
export default function Task({ task: { id, title, state }, onArchiveTask, onPinTask }) {
  return (
    <div className={`list-item ${state}`}>
      <label
        htmlFor={`archiveTask-${id}`}
        aria-label={`archiveTask-${id}`}
        className="checkbox"
      >
        <input
          type="checkbox"
          disabled={true}
          name="checked"
          id={`archiveTask-${id}`}
          checked={state === "TASK_ARCHIVED"}
        />
        <span
          className="checkbox-custom"
          onClick={() => onArchiveTask(id)}
        />
      </label>

      <label htmlFor={`title-${id}`} aria-label={title} className="title">
        <input
          type="text"
          value={title}
          readOnly={true}
          name="title"
          id={`title-${id}`}
          placeholder="Input title"
        />
      </label>
      {state !== "TASK_ARCHIVED" && (
        <button
          className="pin-button"
          onClick={() => onPinTask(id)}
          id={`pinTask-${id}`}
          aria-label={`pinTask-${id}`}
          key={`pinTask-${id}`}
        >
          <span className={`icon-star`} />
        </button>
      )}
    </div>
  );
}
```

上面的额外标记与我们之前导入的 CSS 结合后，生成了以下 UI：

<video autoPlay muted playsInline loop>
  <source
    src="/intro-to-storybook/finished-task-states-7-0.mp4"
    type="video/mp4"
  />
</video>

## 指定数据必要条件

在 React 中，使用 `propTypes` 来指定组件所期望的数据结构是一种最佳实践。它不仅具有自我文档化的作用，还可以帮助及早发现问题。

```diff:title=src/components/Task.jsx
+ import PropTypes from 'prop-types';

export default function Task({ task: { id, title, state }, onArchiveTask, onPinTask }) {
  return (
    <div className={`list-item ${state}`}>
      <label
        htmlFor={`archiveTask-${id}`}
        aria-label={`archiveTask-${id}`}
        className="checkbox"
      >
        <input
          type="checkbox"
          disabled={true}
          name="checked"
          id={`archiveTask-${id}`}
          checked={state === "TASK_ARCHIVED"}
        />
        <span
          className="checkbox-custom"
          onClick={() => onArchiveTask(id)}
        />
      </label>

      <label htmlFor={`title-${id}`} aria-label={title} className="title">
        <input
          type="text"
          value={title}
          readOnly={true}
          name="title"
          id={`title-${id}`}
          placeholder="Input title"
        />
      </label>
      {state !== "TASK_ARCHIVED" && (
        <button
          className="pin-button"
          onClick={() => onPinTask(id)}
          id={`pinTask-${id}`}
          aria-label={`pinTask-${id}`}
          key={`pinTask-${id}`}
        >
          <span className={`icon-star`} />
        </button>
      )}
    </div>
  );
}
+ Task.propTypes = {
+  /** Composition of the task */
+  task: PropTypes.shape({
+    /** 任务的Id */
+    id: PropTypes.string.isRequired,
+    /** 任务标题 */
+    title: PropTypes.string.isRequired,
+    /** 当前任务状态 */
+    state: PropTypes.string.isRequired,
+  }),
+  /** 将任务更改为已归档状态的事件 */
+  onArchiveTask: PropTypes.func,
+  /** 将任务更改为已固定状态的事件 */
+  onPinTask: PropTypes.func,
+ };
```

现在，如果误用 Task 组件，在开发环境中会出现警告。

<div class="aside">
💡 实现相同目的的另一种方法是使用像 TypeScript 这样的 JavaScript 类型系统，为组件属性创建类型。
</div>

## 组件构建!

我们现在已经成功构建了一个组件，无需服务器或运行整个前端应用程序。下一步是以类似的方式逐个构建其余的 Taskbox 组件。

正如你所看到的，开始构建独立组件的过程既简单又快速。由于可以深入测试每种可能的状态，我们可以期望生产出更少漏洞和更完善的高质量 UI。

## 发现可访问性问题

可访问性测试是指使用自动化工具根据一组基于[WCAG](https://www.w3.org/WAI/standards-guidelines/wcag/) 规则和其他行业公认的最佳实践的启发式标准来审计渲染后 DOM 的实。它们作为质量保证的第一道防线，用于发现明显的可访问性违规问题，确保应用程序对尽可能多的人都可用，包括有视力障碍、听力问题和认知障碍的用户。

Storybook 包含一个官方的 [可访问性插件](https://storybook.js.org/addons/@storybook/addon-a11y)。该插件由 Deque 的 [axe-core](https://github.com/dequelabs/axe-core) 提供支持，能够发现高达 [57% 的 WCAG 问题](https://www.deque.com/blog/automated-testing-study-identifies-57-percent-of-digital-accessibility-issues/)。

让我们看看它是如何工作的！运行以下命令来安装该插件：

```shell
yarn add --dev @storybook/addon-a11y
```

然后，更新你的 Storybook 配置文件（`.storybook/main.js`）以启用它：

```diff:title=.storybook/main.js
/** @type { import('@storybook/react-vite').StorybookConfig } */
const config = {
  stories: ['../src/components/**/*.stories.@(js|jsx)'],
  staticDirs: ['../public'],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
+   '@storybook/addon-a11y',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
};
export default config;
```

最后，重新启动你的 Storybook，以查看 UI 中启用的新插件。

![Task accessibility issue in Storybook](/intro-to-storybook/finished-task-states-accessibility-issue-7-0.png)

在浏览我们的 stories 时，我们可以看到插件在我们一个测试状态中发现了一个可访问性问题。消息 [**“元素必须具有足够的颜色对比度”**](https://dequeuniversity.com/rules/axe/4.4/color-contrast?application=axeAPI) 基本上意味着任务标题与背景之间的对比度不足。我们可以通过在应用程序的 CSS 文件（位于 `src/index.css`）中将文本颜色更改为更深的灰色来快速修复这个问题。

```diff:title=src/index.css
.list-item.TASK_ARCHIVED input[type="text"] {
- color: #a0aec0;
+ color: #4a5568;
  text-decoration: line-through;
}
```

就是这样！我们已经迈出了确保 UI 可访问性的第一步。随着我们继续增加应用程序的复杂性，我们可以对所有其他组件重复这一过程，而无需启动额外的工具或测试环境。

<div class="aside">
💡 别忘了用 git 提交你的更改！
</div>
