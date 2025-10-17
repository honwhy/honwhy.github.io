---
title: "如何在浏览器插件中预估公众号文章阅读时间"
description: "在我开发的 【公众号阅读增强器】 浏览器插件中，使用reading-time 对文章阅读时长进行预估。"
publishDate: "15 Oct 2025"
tags: ["微信", "公众号", "阅读", "插件"]
---
# 如何在浏览器插件中预估公众号文章阅读时间

## 使用reading-time

在我开发的 【公众号阅读增强器】 浏览器插件中，使用reading-time 对文章阅读时长进行预估。

项目主页：[wxreader](https://wxreader.honwhy.wang/)

开源地址：[honwhy/WeChatReaderEnhancer](https://github.com/honwhy/WeChatReaderEnhancer)， 

### 如何使用
安装依赖
```
pnpm install reading-time
```
使用方式一，
```ts
import readingTime from 'reading-time/lib/reading-time'

const { minutes } = readingTime(getArticleContent())
```

使用方式二，
```ts
import readingTime from 'reading-time'

const { minutes } = readingTime(getArticleContent())
```

这两种方式都不完美。

方式一，`import readingTime from 'reading-time/lib/reading-time'` 一直会有恼人的typescript warning

方式二，需要配合`nodepolyfills` 才能正常使用。

```ts
// file vite.config.ts
import { defineConfig } from 'vite'
import { nodePolyfills } from 'vite-plugin-node-polyfills'

export default defineConfig({
  plugins: [
    nodePolyfills({
        include: [`path`, `util`, `timers`, `stream`, `fs`],
        overrides: {
        // Since `fs` is not supported in browsers, we can use the `memfs` package to polyfill it.
        // fs: 'memfs',
        },
      }),
  ]
})
```

### 效果
比如阅读这篇文章预估是3分钟，

https://mp.weixin.qq.com/s/TNN35EJYWmkomMXmwLLcFg

![](https://fastly.jsdelivr.net/gh/bucketio/img0@main/2025/10/15/1760459834440-a40a9367-a9fe-43bd-aabc-4510fe28aba8.png)

### API 简要说明

参考文档：[reading-time](https://www.npmjs.com/package/reading-time)

默认是每分钟阅读200个单词。

## 使用reading-time-estimator

本人对`nodepolyfills` 稍微有点成见，在`cloudflare workers` 穷鬼套餐中， 使用`@cloudflare/vite-plugin` 在dev 模式也会因为`nodepolyfills`的问题导致运行中报错。（因为cloudflare workerd 不是真正的nodejs）

关于[workerd](https://github.com/cloudflare/workerd)

### 如何使用
安装依赖
```
pnpm install reading-time-estimator
```
预估阅读时长，
```ts
import { readingTime } from 'reading-time-estimator'

const { minutes } = readingTime(getArticleContent(), 200, `zh-cn`)
```

### 依然存在问题

问题表现为编译成浏览器插件后的content-scripts内容不是utf-8编码的，导致无法启用，已经给原作者提了issue

[reading-time-estimator#2049](https://github.com/lbenie/reading-time-estimator/issues/2049)


![](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/10/15/1760460690356-c54d277c-6086-4571-b1f0-542c8705ba27.png)

### 解决办法
// file: plugins/vite-plugin-to-utf8.ts
```ts
import type { PluginOption } from 'vite'

function strToUtf8(str: string) {
  return str.split(``).map(ch =>
    ch.charCodeAt(0) <= 0x7F
      ? ch
      : `\\u${(`0000${ch.charCodeAt(0).toString(16)}`).slice(-4)}`).join(``)
}

export default function toUtf8(): PluginOption {
  return {
    name: `to-utf8`,
    enforce: `post`,
    generateBundle(options: any, bundle: any) {
      for (const fileName in bundle) {
        const file = bundle[fileName]
        if (file.type === `chunk`) {
          const originCode = file.code
          const modifiedCode = strToUtf8(originCode)
          file.code = modifiedCode
        }
      }
    },
  }
}

```

这段魔法代码忘记从哪里找到的，有点遗憾。

```ts
import toUtf8 from './plugins/vite-plugin-to-utf8'

export default defineConfig({
  plugins: [
    toUtf8(),
  ]
})
```
### API 简要说明

参考文档：[reading-time-estimator](https://www.npmjs.com/package/reading-time-estimator)

方法需要3个参数，(text, 200, 'zh-cn')

text: 需要预估的文章内容

200: 每分钟阅读的单词数

'zh-cn': 指定文字语言

## 引用链接

1. wxreader: https://wxreader.honwhy.wang/
2. honwhy/WeChatReaderEnhancer: https://github.com/honwhy/WeChatReaderEnhancer
3. reading-time: https://www.npmjs.com/package/reading-time
4. workerd: https://github.com/cloudflare/workerd
5. reading-time-estimator#2049: https://github.com/lbenie/reading-time-estimator/issues/2049
6. reading-time-estimator: https://www.npmjs.com/package/reading-time-estimator