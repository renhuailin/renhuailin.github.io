---
title: 从报错中学习NodeJS的模块系统和TypeScript的配置
date: 2025-04-20T12:40:32+08:00
summary: 我发现我开发的chrome插件有个bug，找到这个bug后，我要用一个库来修复这个bug，结果我在引用这个库时一直报错，我不知道是不是库的作者在导出这个库时设置不对，还是我的环境有问题，于是我新建了一个ts项目，要自己编译一下看看，结果这个过程碰到一堆问题！
slug: learning-nodejs-typescript-from-compile-errors
categories:
  - 开发
tags:
  - 开发
---
今天，我发现我开发的chrome插件有个bug，找到这个bug后，我要用一个库来修复这个bug，结果我在引用这个库时一直报错，我不知道是库的作者在导出这个库时设置不对，还是我的开发环境有问题。

```
Error: Cannot find module '/Users/harley/workspaces/Typescript/save_to_notion_server/node_modules/html-to-notion-blocks/src/index.js'
    at createEsmNotFoundErr (node:internal/modules/cjs/loader:1262:15)
    at finalizeEsmResolution (node:internal/modules/cjs/loader:1250:15)
    at resolveExports (node:internal/modules/cjs/loader:634:14)
    at Function.Module._findPath (node:internal/modules/cjs/loader:724:31)
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1211:27)
    at Function.Module._load (node:internal/modules/cjs/loader:1051:27)
    at Module.require (node:internal/modules/cjs/loader:1311:19)
    at require (node:internal/modules/helpers:179:18)
    at Object.<anonymous> (/Users/harley/workspaces/Typescript/save_to_notion_server/src/services/NotionService.ts:3:1)
    at Module._compile (node:internal/modules/cjs/loader:1469:14)
```

下面是这个库的`package.json`

```json
  "exports": {
    ".": {
      "types": "./index.d.ts",
      "import": "./index.js",
      "require": "./src/index.js"
    }
  },
```

这里面有个`"require": "./src/index.js"`但是安装的文件夹里没有这个文件。导致我的项目编译总是报错。

于是我就想，我新建一个ts的项目，把这个项目里的源代码copy过来，看看能不能调用。

```
$ mkdir typescript-project
$ cd typescript-project
$ npm i typescript --save-dev
$ pnpm exec tsc --init
```

这样一个新的ts项目就创建好了。

接下来我直接运行ts文件 `ts-node src/index.ts`。结果报错：

```
TypeError: Unknown file extension ".ts" for /Users/harley/workspaces/Typescript/html_to_notion_test/src/index.ts
```

问了Gemini，发现可能是在我的代码里引用了其它的ts文件，而我又没有设置解析路径。

那tsconfig.json里moduleResolution是做什么的呢？

在 `tsconfig.json` 中，`moduleResolution` 选项用于告诉 TypeScript 编译器 **如何解析模块路径** 。

简单来说，当你写下这样的导入语句时：

```ts
import { someFunction } from 'some-package'; // 非相对导入
import { anotherThing } from './some-file'; // 相对导入
```

`moduleResolution` 就会指导 TypeScript 编译器去哪里以及如何查找这些模块对应的文件（特别是 `.ts`、`.tsx`、`.d.ts` 文件）。

它之所以重要，是因为不同的 JavaScript 环境（例如 Node.js、浏览器、不同的打包工具如 Webpack、Rollup）有不同的模块解析规则：

1. **Node.js 的解析规则：** Node.js 在处理 `require()` 或 `import` 时，会按照一套特定的算法查找文件，包括：
    
    - 查找内置模块。
    - 对于非相对路径（如 `'some-package'`），会在当前目录及父目录的 `node_modules` 文件夹中查找同名文件夹或文件。
    - 在找到的文件夹中，会根据 `package.json` 的 `main` 字段（CommonJS 时代）或 **`exports` 字段（现代 Node.js）** 来确定入口文件。
    - 对于相对路径（如 `'./some-file'`），会尝试 `.js`、`.json`、`.node` 等扩展名，或者如果找到同名文件夹，则查找该文件夹下的 `index.js` 等。
    - 在 ES Module 环境下（`.mjs` 文件或 `package.json` 中设置 `"type": "module"`），Node.js 的解析规则会有所不同，例如强制要求相对导入带文件扩展名，并且对 `exports` 字段的处理更加严格和精确。
2. **打包工具的解析规则：** Webpack、Rollup 等打包工具通常有自己的解析插件和配置，它们可能支持路径别名、自动处理某些扩展名、以及更复杂的查找逻辑。

**`moduleResolution` 选项的作用就是让 TypeScript 模拟这些不同的解析行为。** TypeScript 需要在编译时正确地找到导入的模块对应的类型定义文件（`.d.ts`），以便进行类型检查。如果 `moduleResolution` 设置不正确，TypeScript 可能找不到导入的模块，或者找到错误的类型定义，从而导致编译错误（比如 "Cannot find module 'some-package'"）或错误的类型检查结果。

`moduleResolution` 常见的选项值有：

- `classic`: 旧的、不推荐使用的解析策略。
- `node` (或 `node10`): 模拟 Node.js 传统的 CommonJS 模块解析算法。它主要依赖 `main` 字段，**对 `package.json` 的 `exports` 字段支持不佳或不支持**。
- `node16` / `nodenext`: 模拟 Node.js 16 及更高版本的模块解析算法。这些策略**完全支持 `package.json` 的 `exports` 字段**，并且能够正确处理 CommonJS 和 ES Modules 之间的交互和区分。它们是最能反映现代 Node.js 模块解析行为的选项。
- `bundler`: 为配合打包工具而设计的策略。它在支持 `exports` 的同时，也包含一些打包工具常用的解析特性（例如不强制要求相对导入带文件扩展名，即使在 ESM 环境下）。

我意识到直接用ts-node来运行可能会有一些问题，于是我直接使用`pnpm exec tsc src/index.ts`来编译。这回的错误如下：

```
node_modules/.pnpm/hast-util-to-mdast@9.0.0/node_modules/hast-util-to-mdast/lib/state.d.ts:74:16 - error TS2583: Cannot find name 'Map'. Do you need to change your target library? Try changing the 'lib' compiler option to 'es2015' or later.

74   elementById: Map<string, Element>
                  ~~~

node_modules/.pnpm/parse5@7.2.1/node_modules/parse5/dist/common/foreign-content.d.ts:3:52 - error TS2583: Cannot find name 'Map'. Do you need to change your target library? Try changing the 'lib' compiler option to 'es2015' or later.

3 export declare const SVG_TAG_NAMES_ADJUSTMENT_MAP: Map<string, string>;
                                                     ~~~

node_modules/.pnpm/parse5@7.2.1/node_modules/parse5/dist/common/html.d.ts:287:51 - error TS2583: Cannot find name 'Set'. Do you need to change your target library? Try changing the 'lib' compiler option to 'es2015' or later.

287 export declare const SPECIAL_ELEMENTS: Record<NS, Set<TAG_ID>>;
                                                      ~~~

node_modules/.pnpm/parse5@7.2.1/node_modules/parse5/dist/common/html.d.ts:288:40 - error TS2583: Cannot find name 'Set'. Do you need to change your target library? Try changing the 'lib' compiler option to 'es2015' or later.

288 export declare const NUMBERED_HEADERS: Set<TAG_ID>;
                                           ~~~

src/index.ts:1:10 - error TS2614: Module '"rehype-notion"' has no exported member 'rehypeNotion'. Did you mean to use 'import rehypeNotion from "rehype-notion"' instead?

1 import { rehypeNotion } from 'rehype-notion';
           ~~~~~~~~~~~~
```

看来是这个库是使用es6的版本编写的，错误信息提示我必须指定lib为`es6`或`es2015`。

我在`tsconfig.json`中修改了lib和target
```json
"target": "es2016", /* Set the JavaScript language version for emitted JavaScript and include compatible library declarations. */

"lib": ["ES6"],
```
我再运行编译命令，结果可是还是报上面的错误。

这太奇怪了！一顿Google大法后，我看到sof上的帖子，原来是忘记了安装`@types/node`这个包。

**`@types/node` 通常包含了 Node.js 环境所需的标准库定义：** 在 Node.js 环境下，像 `Map`, `Set`, `Promise` 等 ES6+ 的全局对象都是可用的。为了在 TypeScript 中正确地为这些 Node.js 环境下的标准全局对象提供类型，`@types/node` 这个包**通常会在其内部依赖或包含**了对应的标准库声明文件（比如等同于 `lib: ["es2015", "es2016", ...]` 的部分或全部）

明白了原因后，安装它。
```
pnpm add -D @types/node
```
安装完以后，上面的错误确实不见了。但是新错误出来了：

```ts
import {rehypeNotion} from "rehype-notion"
```
```
Module '"rehype-notion"' has no exported member 'rehypeNotion'. Did you mean to use 'import rehypeNotion from "rehype-notion"' instead?
```

错误的意思直译过来就是：**模块 `"rehype-notion"` 没有一个叫做 `rehypeNotion` 的导出成员。你是不是想使用 `import rehypeNotion from "rehype-notion"` 这种方式来导入？**

那么这两种导入方式有什么区别呢？
```ts
import {rehypeNotion} from "rehype-notion"
import rehypeNotion from "rehype-notion"
```

要想知道这两种导入方式的区别，我们要首先了解模块的导出方式。

一个模块可以通过以下方式导出内容：

1. **命名导出 (Named Export):** 使用 `export const name = ...` 或 `export function func() { ... }`。导入时使用花括号 `{ name, func }`。
2. **默认导出 (Default Export):** 使用 `export default ...`。一个模块只能有一个默认导出。导入时直接给导出的内容起一个名字，不使用花括号，例如 `import myDefaultExport from 'module'`。

如果模块使用的是**命名导出**，你在导入的时候当然是要用**命名导入**；如果模块使用的是**默认导出**，你在导入的时候当然是要用**默认导入**。

而上面的错误说明我尝试使用的**命名导入 (Named Import)** 方式 `{ rehypeNotion }` 在这个 `"rehype-notion"` 包的类型定义 (`.d.ts` 文件) 中并没有找到对应的导出。

而错误信息给出的建议 `import rehypeNotion from "rehype-notion"` 则暗示着，`rehypeNotion` 可能是这个包的 **默认导出 (Default Export)**。


于是我查看了"rehype-notion"的index.d.ts，它里面确实有这样一行代码：
```ts
export { default } from './lib/rehype-notion.js';
```

这正是 ES Module 中用于**将另一个模块的默认导出，作为当前模块的默认导出**的语法。

它的含义是：

- 从 `./lib/rehype-notion.js` 这个文件中获取它的 **默认导出 (default export)**。
- 然后把这个默认导出，**作为当前 `index.d.ts` 文件所在的模块 (即 `"rehype-notion"` 包)** 的 **默认导出** 暴露出去。

所以，这行代码明确表明，`"rehype-notion"` 包的顶级导出就是一个 **默认导出**。

这就完全解释了为什么之前使用 **命名导入** `import { rehypeNotion } from "rehype-notion"` 时会收到错误 `Module '"rehype-notion"' has no exported member 'rehypeNotion'`。因为这个包在顶层并没有一个叫做 `rehypeNotion` 的**命名导出**，它只有一个 **默认导出**。

因此，根据这个 `.d.ts` 文件，正确的导入方式确实应该是使用 **默认导入**，于是我修改了代码。

```ts
import rehypeNotion from "rehype-notion"; // 没有花括号
```


现在我运行pnpm exec tsc src/index.ts已经能成功编译了，但是在执行node src/index.js时报错了，

```
file:///Users/harley/workspaces/Typescript/html_to_notion_test/src/index.js:2

Object.defineProperty(exports, "__esModule", { value: true });
                      ^
ReferenceError: exports is not defined in ES module scope

This file is being treated as an ES module because it has a '.js' file extension and '/Users/harley/workspaces/Typescript/html_to_notion_test/package.json' contains "type": "module". To treat it as a CommonJS script, rename it to use the '.cjs' file extension.

    at file:///Users/harley/workspaces/Typescript/html_to_notion_test/src/index.js:2:23

    at ModuleJob.run (node:internal/modules/esm/module_job:234:25)

    at async ModuleLoader.import (node:internal/modules/esm/loader:473:24)

    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:123:5)

  

Node.js v20.18.0
```

好的，现在我们遇到了一个新的、非常明确的错误：

```
ReferenceError: exports is not defined in ES module scope
This file is being treated as an ES module because it has a '.js' file extension and '/Users/harley/workspaces/Typescript/html_to_notion_test/package.json' contains "type": "module". To treat it as a CommonJS script, rename it to use the '.cjs' file extension.
```

这个错误信息告诉我：

1. **Node.js 正在将 `src/index.js` 文件视为 ES Module (`.js` 扩展名 + `package.json` 中有 `"type": "module"`)。**
2. **但是，编译出来的 `src/index.js` 文件中包含了 CommonJS 的语法 (`exports`)。**
3. **在 ES Module 的顶层作用域中，`exports` 是没有定义的，因此 Node.js 运行时报错 `ReferenceError: exports is not defined in ES module scope`。**

在解决这个问题前，我要先弄清什么是`CommonJS`，什么是`ES Module`。

由于历史的原因，在ES Module出现之前，JavaScript有过多种模块机制，如`commonjs`、`amd`、 `umd`等等。这些都是 JavaScript 中用于组织和加载代码的**模块系统 (Module Systems)**。在早期 JavaScript 没有官方的模块化规范时，社区为了解决代码组织、依赖管理和避免命名冲突的问题，发展出了多种不同的模块化方案。ES Module 则是 ECMAScript 标准委员会推出的官方模块系统。

下面是它们各自的介绍：

1. **CommonJS**
    
    - **起源与目的：** 主要用于服务器端 JavaScript，特别是 **Node.js** 环境。Node.js 在诞生之初并没有官方的模块系统，于是采用了 CommonJS 规范。
        
    - **语法：** 使用 `require()` 函数来导入（加载）模块，使用 `module.exports` 或简写 `exports` 对象来导出模块中的内容。
        
    - **加载方式：** **同步加载**。当使用 `require()` 导入一个模块时，Node.js 会立即加载并执行该模块，然后返回导出的内容。这在服务器端文件系统访问速度快的情况下不是问题。
        
    - **特点：** 简单直观，成为 Node.js 生态系统的基石。但不适合直接用于浏览器环境，因为同步加载会阻塞页面渲染（除非经过打包工具处理）。
        
    - **示例：**
        
        ```js
        // greet.js (模块文件)
        const name = 'World';
        function sayHello() {
          console.log('Hello, ' + name);
        }
        module.exports = { // 导出对象或单个值
          sayHello: sayHello,
          name: name
        };
        
        // main.js (使用模块的文件)
        const greet = require('./greet'); // 同步导入模块
        greet.sayHello(); // 输出: Hello, World
        console.log(greet.name); // 输出: World
        ```
        
2. **AMD (Asynchronous Module Definition)**
    
    - **起源与目的：** 主要用于客户端 JavaScript（浏览器环境）。为了解决 CommonJS 同步加载不适应浏览器网络环境的问题，AMD 规范被提出，旨在实现模块的**异步加载**。
        
    - **语法：** 主要依赖全局的 `define()` 函数来定义模块，以及 `require()` 函数来异步加载依赖和执行回调函数。常见的实现库是 RequireJS。
        
    - **加载方式：** **异步加载**。模块及其依赖通过网络异步获取，不会阻塞主线程。所有依赖都加载完成后，再执行定义模块的回调函数。
        
    - **特点：** 解决了浏览器环境下模块依赖和加载顺序的问题，避免了全局变量污染。但语法相对复杂。
        
    - **示例 (使用 RequireJS 风格)：**        
        ```js
        // greet.js (模块文件)
        define(['require'], function(require) { // 使用 define 定义模块，声明依赖
          const name = 'World';
          function sayHello() {
            console.log('Hello, ' + name);
          }
          return { // 通过 return 导出内容
            sayHello: sayHello,
            name: name
          };
        });
        
        // main.js (使用模块的文件)
        require(['greet'], function(greet) { // 使用 require 异步加载模块，并在加载完成后执行回调
          greet.sayHello(); // 输出: Hello, World
          console.log(greet.name); // 输出: World
        });
        ```
        
3. **UMD (Universal Module Definition)**
    
    - **起源与目的：** UMD 不是一个新的模块系统，而是一种**模块定义模式（模式）**。它旨在创建一个能**兼容多种环境**（包括 CommonJS 环境、AMD 环境以及直接在浏览器中通过全局变量访问的环境）的 JavaScript 模块。
        
    - **语法：** 通常是一个包裹函数（Immediately Invoked Function Expression, IIFE），通过检测当前运行环境中是否存在 `define` 函数（判断是否是 AMD 环境）或 `module.exports` 对象（判断是否是 CommonJS 环境）来决定采用哪种方式导出和引入模块。如果两者都不存在，则将模块暴露为全局变量。
        
    - **加载方式：** 根据检测到的环境采取对应的加载方式（同步或异步）。
        
    - **特点：** 提供最大的兼容性，一份代码理论上可以在任何地方使用。但代码包裹层比较复杂。许多流行的 JavaScript 库为了兼容性会采用 UMD 模式打包。
        
    - **示例 (简化版)：**
        ```js
        (function (root, factory) { // root 指代全局对象 (如浏览器中的 window, Node.js 中的 global)
            if (typeof define === 'function' && define.amd) {
                // AMD 环境
                define([], factory); // 如果有依赖，放在第一个数组里
            } else if (typeof module === 'object' && module.exports) {
                // CommonJS 环境 (Node.js)
                module.exports = factory(); // 如果有依赖，使用 require 导入
            } else {
                // 浏览器全局环境
                root.MyModule = factory();
            }
        }(typeof self !== 'undefined' ? self : this, function () {
            // 这是你的模块代码，定义要导出的内容
            const foo = 'bar';
            return { // 返回要导出的对象或值
              foo: foo
            };
        }));
        ```
        
4. **ES Module (ECMAScript Module)**
    
    - **起源与目的：** ECMAScript 标准委员会在 ES6 (ECMAScript 2015) 中引入的**官方、原生的模块系统**。目标是成为浏览器和服务器端通用的标准。
        
    - **语法：** 使用 `import` 语句导入模块，使用 `export` 语句导出模块中的变量、函数、类等。支持命名导出和默认导出。
        
    - **加载方式：** 标准规定是**异步加载**，但 Node.js 在实现文件系统模块加载时是同步的。它具有**静态结构**，意味着导入和导出的关系在代码执行前就能确定，这使得静态分析（如 Tree Shaking 移除未使用代码）成为可能。
        
    - **特点：** 现代、官方的标准，语法简洁优雅。得到现代浏览器和 Node.js 的原生支持（在 Node.js 中使用通常需要在 `package.json` 中设置 `"type": "module"` 或使用 `.mjs` 文件扩展名）。是未来 JavaScript 模块化的方向。
        
    - **示例：**
        ```js
        // greet.js (模块文件)
        export const name = 'World'; // 命名导出
        export function sayHello() { // 命名导出
          console.log('Hello, ' + name);
        }
        // export default ...; // 默认导出 (一个模块只能有一个)
        
        // main.js (使用模块的文件)
        import { sayHello, name } from './greet.js'; // 使用 import 导入 (注意文件扩展名通常需要)
        // import anyName from './some-other-module.js'; // 导入默认导出
        sayHello(); // 输出: Hello, World
        console.log(name); // 输出: World
        ```
        

明白了上面这些模块系统后，我们再来看看是什么原因导致上面的错误。

首先node在运行一个js文件时，它是不知道这个文件使用的是哪个模块系统的，那它是怎么决定使用哪个模块系统的呢？

根据[官方文档](https://nodejs.org/api/packages.html#determining-module-system)中关于模块部分的说明，我做了简单的总结。

1. 根据文件扩展名。  如果扩展名是`.mjs`node会把它当成ES modules；如果扩展名是`.cjs`,node则认为这是CommonJS模块。
2. 如果扩展名是`.js`，就会查找最近的`package.json`，看看它的type这个字段的值是什么。如果它的值是`"module"`，就会使用ES modules； 如果是`commonjs`则会使用CommonJS。
3. 由命令行参数`--input-type`来决定。

我看了项目的`package.json`，type是设置为"module"的： `"type": "module"`。

在上面的错误说，nodejs的执行的时候使用的是es module，但是tsc编译时指定的模块系统是commonjs.

TypeScript 编译器根据 `tsconfig.json` 中的 `module` 选项来决定生成哪种模块格式的代码：

- 如果 `module` 设置为 `commonjs` 或某些旧值，`tsc` 会生成使用 `require()` 和 `exports`/`module.exports` 的 CommonJS 代码。
- 如果 `module` 设置为 `es2015` 或更高版本（如 `esnext`）或针对 Node.js ES Module 的值（如 `node16`, `nodenext`），`tsc` 会生成使用 `import` 和 `export` 语法的 ES Module 代码。

但是我的`tsconfig.json`中module设置是`nodenext`,也是ES Module呀。

难道`pnpm exec tsc`时使用的配置不对？

但是我通过`pnpm exec tsc --showConfig`这个命令查看tsc的配置时，我发现它的module的配置是"module": "nodenext"，也就是ts在编译的时候，用的是es module啊，这是怎么回事呢?

在问了gemini后，它提示我，有可能`pnpm exec tsc`在执行时没有使用当前目录下的 `tsconfig.json`中的配置。它让我使用`pnpm exec tsc --project ./tsconfig.json`这种指定配置文件的方式来编译试试。

于是我用`pnpm exec tsc --project ./tsconfig.json src/index.ts`这条命令来运行，发现不能同时指定源文件和配置文件。也就是说你指定了配置文件，就不能再指定源文件了，你应该在配置文件是指明源文件和编译输出路径。

于是我修改了tsconfig.json，指定了源文件目录和输出目录。然后`pnpm exec tsc --project ./tsconfig.json`来编译，这样生成的index.js，运行就成功了。


总结一下：
1. tsc init创建的配置`tsconfig.json`默认使用`CommonJS`模块系统。现在已经是2025年了，大部分的包已经都改成了ES6的标准模块系统，所以这块要手动修改为使用es module.
2. 要注意区分`命名导入`和`默认导入`。
3. 在NodeJS环境下编译ts,一定要加上`@types/node `,对应的pnpm命令是：`pnpm add -D @types/node`
4. 学习了Javascript模块系统的历史，以及为什么会有`commonjs`、`amd`、 `umd`和`es module`等。
5. 要想正确地编译ts，最好使用`tsc --project ./tsconfig.json`，不要编译单个ts文件。

