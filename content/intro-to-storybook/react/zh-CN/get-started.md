---
title: 'Storybook React 教程'
tocTitle: '开始吧'
description: '在你的开发环境下设置 Storybook'
commit: 'bf3514f'
---

Storybook 是在开发模式下与您的应用程序一起运行的。它帮助您在与应用程序的业务逻辑和上下文隔离的情况下构建 UI 组件。这是 Storybook 入门教程的 React 版本。其他版本还包括[React Native](/intro-to-storybook/react-native/en/get-started), [Vue](/intro-to-storybook/vue/en/get-started), [Angular](/intro-to-storybook/angular/en/get-started), [Svelte](/intro-to-storybook/svelte/en/get-started) and [Ember](/intro-to-storybook/ember/en/get-started).

![Storybook and your app](/intro-to-storybook/storybook-relationship.jpg)

## 设置 React Storybook

我们需要遵循一些步骤来在我们的环境中设置构建流程。首先，我们需要使用 [degit](https://github.com/Rich-Harris/degit) 来设置我们的构建系统。通过使用这个工具包，您可以下载“模板”（部分构建完成的且带有一些默认配置的应用程序，）来帮助您加快开发流程。

执行以下指令：

```shell:clipboard=false
# 克隆模版
npx degit chromaui/intro-storybook-react-template taskbox

cd taskbox

# 安装依赖
yarn
```

<div class="aside">
💡 这个模板包含了本教程版本所需的样式、资源和基本的必要配置。
</div>

现在我们可以快速检查我们应用程序的各个环境是否正确运行：

```shell:clipboard=false
# 在端口 6006 上启动组件管理器：
yarn storybook

# 在端口 5173 上正确地运行前端应用程序：
yarn dev
```

我们的主要前端应用形式包括：组件开发（Storybook）和应用程序本身。

![Main modalities](/intro-to-storybook/app-main-modalities-react.png)

根据你正在处理应用程序的哪个部分，你可能想要同时运行其中一个或多个。由于我们当前的重点是创建一个单一的 UI 组件，我们将继续运行 Storybook。

## 提交更改

在这个阶段，可以安全地将我们的文件添加到本地仓库。运行以下命令来初始化本地仓库，添加并提交我们迄今为止所做的更改。

```shell
git init
```

接着：

```shell
git add .
```

然后：

```shell
git commit -m "first commit"
```

最后：

```shell
git branch -M main
```

让我们开始构建第一个组件吧！
