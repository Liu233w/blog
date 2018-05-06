---
layout: blog
date: 2018-05-06 22:10:18
tags:
- web
- 前端
- vue
- nuxt
title: 在 Nuxt 中修改 uglifyjs 的配置
---

假如需要修改 Nuxt 中 UglifyJs 的配置的话，可以直接在 `nuxt.config.js` 的 `build` 中添加一个 `uglify` 字段。如果其值是一个对象的话，对象的内容会跟 nuxt 默认的 uglify 配置进行合并。如果值是 false 的话，就不会使用 uglify plugin。

比如，如果想要在输出的文件中移除控制台输出，只需要使用如下设置：
```js
module.exports = {
  build: {
    uglify: {
      uglifyOptions: {
        compress: {
          drop_console: true,
        },
      },
    },
  },
}
```

控制此行为的代码在 https://github.com/nuxt/nuxt.js/blob/78494dac927681007ea6725b4548db8fd838ed16/lib/builder/webpack/client.config.js#L202 ，这个居然没有写进文档里，真是不靠谱\_(:3」∠)\_
