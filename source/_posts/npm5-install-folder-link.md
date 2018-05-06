---
categories:
  - blog
date: 2018-04-24 23:51:41
tags:
  - 前端
  - npm
title: npm5中本地间模块引用的最好方式（附带引用方法总结）
---

## 前言

对于某些比较大的项目，我们可能会希望将其划分成多个模块：

- module1
  - package.json
  - libs
    - file1.js
- module2
  - package.json
  - index.js

假如我们要在 `module2/index.js` 中直接引用 `file1.js`，可能就会写成这样 `require('../module1/libs/file1')`。这样写并不好，因为：
1. 有时候引用方的文件处在很深的路径下，那么引用地址就得写成 `../../../module1/libs/file1`，体验很不好。
2. 既然我们将项目划分成了多个模块，肯定是想要降低模块之间的偶合度。这样直接引用文件的做法相当于直接深入到了模块的内部，没有将模块之间隔离出来，低耦合更无从谈起了。

解决2号问题最好的做法是提供一个单独的文件作为模块的接口。

比如上文的文件结构，我们可以变成这样：

- module1
  - package.json
  - index.js
  - libs
    - file1.js
- module2
  - package.json
  - index.js

在 `module1/index.js` 中把我们希望在本模块中暴露出来的函数或类export出来，这样的话就相当于给模块提供了一个接口。其他模块只能通过 index.js 提供的内容来访问。假如以后需要重构 module1，将 file1.js 拆成两个文件，只需要改动一下 `module1/index.js`即可，不需要修改其他的模块。

现在我们的引用可以写成 `require('../module1')` 了，本文章的剩余内容就来讨论一下怎样解决1号问题——去掉前面的`..`。

## 1. 使用 `npm install ../module1`
我们可以把 module1 作为依赖安装给 module2。这样的话，module2在运行的时候就可以把 module1 当成一个普通的依赖，使用 `require('module1')` 来引入了。这个方法的问题在于要安装的module1必须在package.json文件中有version字段，否则无法安装成功。对于一个应用程序（而不是类库）而言，每次修改都需要更新版本还是太麻烦了。而且npm5的安装会直接创建一个软链接，会引发更多的问题（我下文要说）。

## 2. 直接修改 package.json 字段，增加依赖
这种方法也很有意思。我们可以直接在 `module2/package.json` 的 dependencies 中增加一个 `"module1": "file:../module1"` 。这样npm5在安装依赖的时候会自动创建相应的软链接，自动给module1安装依赖，而且不需要版本号。但这个特性跟npm5的package-lock结合起来就出问题了。

对于进行过这些修改的module2，在安装依赖的时候会把 module1 的 package-lock 也合并进 `module2/package-lock.json`，再一次使用 `npm install` 的时候就会提示 `enoent ENOENT: no such file or directory`。我也不知道这是为什么。不过npm5已经准备[放弃直接安装文件夹的功能而改用 npm link](https://github.com/npm/npm/pull/15900)了，这些问题以后肯定也不会修复了。 

## 3. 使用 [install-local](https://www.npmjs.com/package/install-local)
这玩意跟解决方法1类似，只不过在npm5里面也能直接拷贝文件夹而不是创建软链接。这么做的话还是需要version字段，而且对于module1的改动不会直接反应到module2上，还需要像更新其他模块那样手动更新。

## 4. 使用 npm link
这个比较神奇。我们可以在module1目录下运行 `npm link`，然后在`module2`目录下运行`npm link module1`。这样的话，npm就会自动在module2的node\_modules目录下创建一个软链接，我们拥有了类似于方法2的效果，并且不会修改 package-lock。

但是它也不会修改 package.json。也就是说我们拿到项目源代码之后不能直接用 npm i 来安装所有依赖，还必须得加一步 npm link 才行，还是太麻烦了。

## 5. 使用 [require-rewrite](https://www.npmjs.com/package/require-rewrite)
这个方法解决了上述的所有问题。我们只需要配置一下module2的package.json，加入module1的路径，这样就能使用`require('module1')`来引入模块了。

不过，这个方法又引入了两个新的问题：
1. 引入的时候得多写一步。我们必须要在 `require('module1')` 前面写上 `require('require-rewrite')(__dirname)` 才能在本文件中引入此模块。
2. 静态代码分析失效了。如果你用 webstorm 的话，它显然是没法通过重写过的路径来找到module1的实际路径的，那基于此的代码补全也就失效了。

## 6. 使用 postinstall 钩子手动创建软链接
我个人比较推荐这种做法。

我们可以在module2的package.json的script字段中新建一个：
```json
"postinstall": "node -e \"var s='../module1',d='./node_modules/module1',fs=require('fs'), r=require('path').resolve;fs.exists(d,function(e){e||fs.symlinkSync(r(s),r(d),'junction')});\""
```

它的命令名称为 postinstall，使得这个命令会在 npm install 安装结束之后自动执行。它执行一段node脚本来自动创建链接，起到了跟npm link相似的效果。只要把其中的 module1 更换成你自己的模块路径即可。注意其中的 `junction`，这个参数只在 windows 下才有效果，\*Unix 下这个参数是什么都没有关系。junction代表创建目录链接。

其实这个解决办法还是有两个小问题，不过对于我而言还能接受。

### 不会自动安装依赖的依赖
在 module2 下执行 npm install 并不会自动给 module1 安装依赖。

### 在 root 权限下执行会有问题
默认情况下，在 root 权限下执行 npm install 会阻止 postinstall 脚本运行。我们可以使用普通权限来安装或者使用 `npm install --unsafe-perm` 来安装。

## 参考资料
- https://juejin.im/post/5ab3f77df265da2392364341
- https://github.com/npm/npm/issues/3497
- https://github.com/SamHwang1990/blog/issues/5
