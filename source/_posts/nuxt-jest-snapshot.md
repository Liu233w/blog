---
date: 2018/03/25 00:00:00
tags:
- vue
- jest
- 前端
categories:
- blog
title: Nuxt+Jest 构建页面级的快照测试
---

# 前言

我最近的一个网站用到了 Vue 的 nuxtjs 框架，这个框架的[示例](https://nuxtjs.org/guide/development-tools) 中用的是 ava。很不巧，WebStorm 不支持 ava，所以我打算用 jest 来进行快照测试。在折腾的过程中，我发现了一种神奇的方法，可以 将 nuxt 的构建过程同测试框架完美配合起来，这样在进行快照测试的时候不需要配置 jest 的 babel， 只需要让 jest 来比较生成出的 html 即可。不过缺点就是只能进行页面级的快照测试，没法单独测试 某个组件的快照了。

# 生成 html

我们可以用 nuxt 的构建工具直接生成 html，这样就可以使用 nuxt 的构建设置了。 首先我们在项目的根目录下建立 `/__test__/pages` 文件夹，当然你也可以不用这个路径，只是注意改一下 之后代码里的路径就好。然后在这个路径里建立一个 `pages_snapshot.test.js` ，写入：

```javascript
const {Nuxt, Builder} = require('nuxt')
const {resolve} = require('path')
const cheerio = require('cheerio')

// We keep a reference to Nuxt so we can close
// the server at the end of the test
let nuxt = null

// Init Nuxt.js
beforeAll(async () => {
  const rootDir = resolve(__dirname, '../..')
  const config = require(resolve(rootDir, 'nuxt.config.js'))
  config.rootDir = rootDir // project folder
  config.dev = false // production build
  nuxt = new Nuxt(config)
  await new Builder(nuxt).build()
}, 1200000) // 用两分钟的时间启动 nuxt
```

这段代码是直接启动 nuxt 的构建系统来编译前端文件，这段代码我是从 <https://nuxtjs.org/guide/development-tools> 里面拿来的，但是去掉了最后的 `listen` 。注意这里必须要用 `require` 而不是 es6 的 `import` 来 引入组件，因为这个文件还是在 node 里运行的，且没有经过 babel 编译。

然后就可以用下面的代码来生成 html 并测试快照了：

```javascript
test('page /', async () => {
  const context = {}
  const {html} = await nuxt.renderRoute(path, context)

  expect(html).toMatchSnapshot()
})
```

要运行这个测试用例，只需要 `npm i -D jest` 然后在 `package.json` 里面加上一个 script 即可：

```json
{
  "scripts": {
    "test": "jest __test__/"
  }
}
```

然后运行 `npm test` 即可得到一堆错误。接下来我来讲解一下解决办法。

# `timer.unref is not a function`

我在 <https://github.com/facebook/jest/issues/1909> 找到了它的解决办法，修改一下 package.json， 增加一个配置项即可：

```json
{
  "jest": {
    "testEnvironment": "node"
  }
}
```

# 生成的快照糊到了一起

由于 `render` 得到的 html 是 production 状态的，经过了压缩。而 jest 的 toMatchSnapshot 是按行比较的。因此我们需要先把得到的 html 美化一下，再生成快照，我们可以直接用 jest 的 `jest-serializer-html` 来做到。先安装： `npm i -D jest-serializer-html` ，然后在 package 的 jest 字段里面加一个 snapshotSerializers，配置项就变成了这样：

```json
{
  "jest": {
    "snapshotSerializers": [
      "jest-serializer-html"
    ],
    "testEnvironment": "node"
  }
}
```

这样 jest 就会用这个工具来自动格式化生成的 html 了。

# 每次修改 js 快照都会改变

因为快照的 html 中还包括 webpack 打包出来的 css 和 js 文件，每次修改这些文件的时候 hash 值 都会被改变，然后快照里的相应文件名就改变了。由于我们只需要网页 dom 的快照，不需要这些东西， 我们可以直接把相应的 tag 删掉。我是用 cheerio 做到的，如果你喜欢的话，也可以用其他的工具。

首先安装它 `npm i -D cheerio` 。

然后修改一下刚刚的测试文件，在获取到 html 的时候进行一些奇妙的操作：

```javascript
test('page /', async () => {
  const context = {}
  const {html} = await nuxt.renderRoute(path, context)
  const $ = cheerio.load(html)

  $('link[href^="/_nuxt/"]').remove()
  $('script[src^="/_nuxt/"]').remove()

  expect($.html()).toMatchSnapshot()
})
```

我们移除了所有包含以 `/_nuxt/` 打头的文件的script和link。

# 快照中会出现 `data-v-xxx` 或者 `data-vue-ssr-id` 之类的奇怪东西

这两个是 vue 在生成页面的时候生成的，在不同的系统中结果也不一样，而且在修改单文件组件的 `style scoped` 时也会改变。我们当然也不需要这些东西。

再魔改一下刚刚的代码：

```javascript
test('page /', () => {
  const context = {}
  const {html} = await nuxt.renderRoute(path, context)
  const $ = cheerio.load(html)

  $('link[href^="/_nuxt/"]').remove()
  $('script[src^="/_nuxt/"]').remove()
  // 移除 data-v- 开头的属性和 data-vue-ssr-id 属性
  $('*').each((i, el) => {
    $(el).removeAttr('data-vue-ssr-id')
    for (let key in $(el).attr()) {
      if (key.startsWith('data-v-')) {
        $(el).removeAttr(key)
      }
    }
  })

  expect($.html()).toMatchSnapshot()
})
```

这样我们的测试就已经差不多能用了。

整个的修改过程可以在 <https://github.com/Liu233w/acm-statistics/compare/df3a685bb307f76193ce88f61bda49497bedf383...ba09a04e9ad2e478f63cde8578ae5594ddd2656d> 看到，欢迎大家捧场。
