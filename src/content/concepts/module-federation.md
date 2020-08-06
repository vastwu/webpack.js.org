---
title: 模块联合（Module Federation）
sort: 8
contributors:
  - sokra
  - chenxsan
  - EugeneHlushko
  - jamesgeorge007
  - ScriptedAlchemy
  - vastwu
related:
  - title: 'Webpack 5 Module Federation: A game-changer in JavaScript architecture'
    url: https://medium.com/swlh/webpack-5-module-federation-a-game-changer-to-javascript-architecture-bcdd30e02669
  - title: 'Explanations and Examples'
    url: https://github.com/module-federation/module-federation-examples
  - title: 'Module Federation YouTube Playlist'
    url: https://www.youtube.com/playlist?list=PLWSiF9YHHK-DqsFHGYbeAMwbd9xcZbEWJ
---

## 原因 {#motivation}

一个应用可能会被分成多个独立编译的部分，这些独立编译的部分之并没有互相依赖，他们应该可以被独立的开发和部署

这就是常说的微前端（Micro-Frontends），但并不局限于此

## 底层概念 {#low-level-concepts}

首先我们区分一下本地模块和远程模块。本地模块是本地构建的一部分；远程模块不属于本地构建，而是在运行时从所谓的`container`中加载。

加载远程模块是一组异步操作，当使用远程模块时，会将这个远程模块和入口文件一起异步加载。所以使用远程模块必须依赖chunk加载。

chunk加载经常使用`import()`调用，像`require.ensure`或者`require[(...)]`这些更早的模式中也可以工作

`container`通过异步方式公开一些特定的模块作为入口，公开入口的行为分为以下步骤：
1. 加载模块（异步的）
2. 执行模块（同步的）

步骤1在chunk加载过程中完成。步骤二将在其他模块（本地和远程的）执行前完成。这样，执行顺序就不回受那些从本地转为远程（或者反过来）的模块的影响

也可以可以嵌套使用，`container` 可以使用另一些 `container` 中的模块，在 `container` 之间的循环依赖也是可行的。

### Overriding {#overriding}

`container`可以选择一些本地模块，标记为**可复写的**，`container`的使用者能够设定替换哪些`container`中的某一些被标记为**可复写的**的模块。`container`中的所有模块都会使用被替换了之后的模块，而不会再使用本地模块。当使用者没有设定任何替换时，`container`将会使用本地模块


`container`不需要下载这些被使用者复写的模块，一般是通过独立chunk打包来实现。

另一方面，替换模块的提供方，需要支持异步加载函数。将会允许`container`只在需要的时候加载被替换的模块。同时`container`也不会加载不需要的模块，一般是通过独立chunk打包来实现。

需要一个"name"属性标记`container`中的可重写模块

复写模块与`container`
模块的过程类似，分为以下两部
1. 加载（异步的）
2. 执行（异步的）

W> 当发生`container`嵌套时，提供了复写模块的`container`会自动复写被嵌套`container`中的同"name"模块

复写必须在`container`加载前就被设定好，在初始化chunk中模块，只能被不使用Promise的同步模块复写，一旦被执行，就不能在复写了

## 高层概念 {#high-level-concepts}
每个构建的`container`都可以消费其他构建的`container`。通过这种方式，任何构建都能够加载其他`container`来使用和访问其公开的模块。

**Shared**模块即是可被复写的，也可以复写被嵌套的其他`container`的模块，它们通常指向每个构建中的相同模块，例如相同的库。

`packageName`选项允许设置一个包的名称来寻找一个`requiredVersion`。默认情况下会自动推判断块请求，将`requiredVersion`设置为`false`可以禁用自动判断。

## Building blocks {#building-blocks}

### `OverridablesPlugin` (底层) {#overridablesplugin-low-level}

该插件定义了哪些模块可以被复写，通过一个运行时接口(`__webpack_override__`)可以进行复写

__webpack.config.js__

```js
const OverridablesPlugin = require('webpack/lib/container/OverridablesPlugin');
module.exports = {
  plugins: [
    new OverridablesPlugin([
      {
        // we define an overridable module with OverridablesPlugin
        test1: './src/test1.js',
      },
    ]),
  ],
};
```

__src/index.js__

```js
__webpack_override__({
  // here we override test1 module
  test1: () => 'I will override test1 module under src',
});
```

### `ContainerPlugin` (底层) {#containerplugin-low-level}

该插件创建了一个`container`入口，包含一些指定公开的模块。它还在内部使用`OverridablesPlugin`，并向`container`公开了`override`接口。

### `ContainerReferencePlugin` (底层) {#containerreferenceplugin-low-level}

该插件将特定的`container`引用排除到外部，并允许从这些`container`中加载远程模块。它还调用这些容器的`override`接口来为它们进行复写。本地复写(通过`__webpack_override__`接口，当构建也是一个`container`时使用`override`接口)和指定的复写模块，会被提供给所有被引用的`container`。

### `ModuleFederationPlugin` (高级) {#modulefederationplugin-high-level}

该插件合并了`ContainerPlugin` 和 `ContainerReferencePlug`。复写和可被复写的模块被合并到一个特定的`shared`模块列表中

## 目标概念 {#concept-goals}

- 应该可以公开和使用任何webpack支持的模块类型
- 加载Chunk应该可以并行加载需要的内容（web:从服务端一次性加载）
- 消费者对`container`的控制
  - 覆盖模块是单向操作
  - 同级`container`不能覆盖彼此的模块。
- 概念是独立于运行环境的
  - 可用于web,node.js 等等
- 在`shared`中的相对和绝对请求
  - 即使没有被使用也要被支持
  - 可以被`config.context`解析成相对请求
  - 默认情况下不使用`requiredVersion`。
- `shared`中的模块请求
  - 只有在被使用时才提供
  - 将在构建中匹配所有相同的模块请求
  - 将提供所有的匹配模块
  - 将从package.json中提取`requiredVersion`
  - 当嵌套`node_modules`时，可以提供和消费多个不同的版本
- 在`shared`中，以`/`结尾的模块请求，将匹配所有带有这个前缀的模块请求。

## 使用案例 {#use-cases}

### 单页分拆构建 {#separate-builds-per-page}

单页应用中的每一个页面，从`container`构建中分出，作为一个独立的构建。应用外壳也是一个独立的构建，以远程模块的方式引用所有的页面。通过这种方式，可以单独部署每个页面，在更新路由或添加新路由时才部署应用外壳，应用外壳通常将使用的库作为`shared`模块，以避免在页面构建中出现重复

### 组件库Container {#components-library-as-container}

许多应用共享一个通用的组件库，可以将其构建为一个容器，公开包含的组件。其他应用使用来自组件库`container`的组件。可以单独部署组件库的更新，而不需要重新部署所有应用程序。应用程序自动使用组件库的最新版本

## 远程动态Containers {#dynamic-remote-containers}
`container`接口支持`get`和`init`方法。
`init`是一个兼容异步的方法，只有一个参数：共享范围对象。该对象在远程容器中用作共享作用于，并由主应用填充模块。可以利用它在运行时动态地将远程容器连接到主机容器。

__init.js__

```js
(async () => {
  // Initializes the shared scope. Fills it with known provided modules from this build and all remotes
  await __webpack_init_sharing__('default');
  const container = window.someContainer; // or get the container somewhere else
  // Initialize the container, it may provide shared modules
  await container.init(__webpack_share_scopes__.default);
  const module = await container.get('./module');
})();
```

`container`尝试提供共享模块，但是如果共享模块已经被使用，将忽略警告和提供的共享模块。`container`可能仍然使用它作为降级方案。

以这种方式可以动态的加载不同版本的`shared`模块，作为A/B test。
T> 在尝试动态连接远程`container`之前，确保已加载`container`;


Example:

__init.js__

```js
function loadComponent(scope, module) {
  return async () => {
    // Initializes the shared scope. Fills it with known provided modules from this build and all remotes
    await __webpack_init_sharing__('default');
    const container = window[scope]; // or get the container somewhere else
    // Initialize the container, it may provide shared modules
    await container.init(__webpack_share_scopes__.default);
    const factory = await window[scope].get(module);
    const Module = factory();
    return Module;
  };
}

loadComponent('abtests', 'test123');
```

[查看完整实现](https://github.com/module-federation/module-federation-examples/tree/master/advanced-api/dynamic-remotes)

## 问题处理 {#troubleshooting}

__`Uncaught Error: Shared module is not available for eager consumption`__


？？应用正在急切地执行一个作为全向主机运行的应用。有两个方案

可以在模块联合的高级API中将依赖设置为即时依赖，该API不会将模块放在异步块中，而是同步地提供它们。这允许我们在初始chunk中使用这些共享模块。但是要小心，因为所有提供的和降级模块总是要下载的。建议只在应用程序的某个地方提供它，例如外壳。

我们强烈建议使用异步边界。它将把初始化代码分割成更大的chunk，以避免任何额外的冗余，并在总体上提高性能。

举个例子，你的入口文件大概是这样

__index.js__

```js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
ReactDOM.render(<App />, document.getElementById('root'));
```

我们创建一个`bootstrap`文件，并将入口文件的内容放在里面，然后在入口文件中 import 这个 bootstrap:

__index.js__

```diff
+ import('./bootstrap');
- import React from 'react';
- import ReactDOM from 'react-dom';
- import App from './App';
- ReactDOM.render(<App />, document.getElementById('root'));
```

__bootstrap.js__

```diff
+ import React from 'react';
+ import ReactDOM from 'react-dom';
+ import App from './App';
+ ReactDOM.render(<App />, document.getElementById('root'));
```

这种方法有效，但也有局限性或缺点。

在`ModuleFederationPlugin`中设置 `eager: true`  

__webpack.config.js__

```js
// ...
new ModuleFederationPlugin({
  shared: {
    ...deps,
    react: {
      eager: true,
    }
  }
});
```

__`Uncaught Error: Module "./Button" does not exist in container.`__

看起来不像`"./Button"`，但是错误信息看起来类似。这是个从webpack beta.16 升级到 bata.17 的代表性问题。

在`ModuleFederationPlugin`中，修改`exposes`

```diff
new ModuleFederationPlugin({
  exposes: {
-   'Button': './src/Button'
+   './Button':'./src/Button'
  }
});
```

__`Uncaught TypeError: fn is not a function`__

您可能丢失了远程`container`，请确保添加了它。
如果已为试图使用的远程服务器加载了`container`，但仍然看到此错误，则还将主机`container`的远程`container`文件添加到HTML中。

