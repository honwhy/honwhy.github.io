---
title: "将Web 项目打包成浏览器插件"
description: "如何使用WXT框架+vite plugin方式将一个Web项目打包成浏览器插件，如何解决cdn js资源引用问题，解决VueDevTools在插件中无法使用问题"
publishDate: "8 Dec 2024"
tags: ["wxt", "vite", "公众号文章编辑器", "markdown"]
---

# 将Web 项目打包成浏览器插件

最近重新思考如何将一个Web项目打包成浏览器插件，利用浏览器插件的options页面来作为Web项目的入口`index.html`。具体就是，如何不改动[doocs/md](https://github.com/doocs/md) 项目源码，仅通过`WXT` 框架及打包命令完成改造。这是一款**微信 Markdown 编辑器**在线项目，功能非常丰富，为了尽最大的可能迁移所有功能，改造过程比较曲折，这里将遇到的主要问题及解决方案记录如下，那就让我们开始吧！


## 项目初始化与配置

### 1.1 项目初始化

采用From Scratch 方式，为已有的项目增加`wxt.config.ts` 等配置，这一步可以参考[官方指引](https://wxt.dev/guide/installation.html#from-scratch)

```markdown
cd md
npm i -D wxt
```
为了插件能够基本启动起来，需要至少配置一个`WXT` 要求的入口点，那么可以学习官方的demo 做法，
```ts
// <srcDir>/entrypoints/background.ts
export default defineBackground(() => {
  console.log('Hello world!');
});
```
`WXT` 的项目结构有点特殊，所以结合[doocs/md](https://github.com/doocs/md) 项目原有结构调整 `wxt.config.ts` 配置，主要是调整 `srcDir` `publicDir` `base` 等属性，

```ts
import { defineConfig } from 'wxt'
import ViteConfig from './vite.config'

export default defineConfig({
  srcDir: `src`,
  publicDir: `../public`,
  extensionApi: `chrome`,
  manifest: {
    name: `公众号文章编辑器`,
    version: `0.0.6`,
    icons: {
      256: `/mpmd/icon-256.png`,
    },
    permissions: [`storage`],
    host_permissions: [],
    web_accessible_resources: [
      {
        resources: [`*.png`, `*.svg`],
        matches: [`<all_urls>`],
      },
    ],
  },
  analysis: {
    open: true,
  },
  vite: () => ({
    ...ViteConfig,
    base: `/`,
  }),
})

```
同时也可以看到，在 `wxt.config.ts` 可以沿用原来项目中的 `vite.config.ts` 配置。

### 1.2 配置增加入口点 (EntryPoint)

由于很难同时满足两种项目的结构形式，此时`index.html` 并不是 `WXT` 的一个入口点，所以需要动态方式配置，这一个环节需要参考 `WXT` 框架的 [hooks](https://wxt.dev/guide/essentials/config/hooks.html) 相关说明。 同时也要了解下 `WXT Modules`，到时候会用到。

解决办法如下，在 `srcDir/modules`目录下新增一个`WXT Modules` 定义，在`WXT` 解析完整个 `wxt.config.ts` 后会调用这个模块的方法。

```ts
// <srcDir>/modules/build-extension.ts
export default defineWxtModule({
  async setup(wxt) {
    wxt.config.alias[`/src/main.ts`] = `./src/main.ts`
    wxt.config.manifest.options_page = `options.html`
    wxt.hook(`entrypoints:grouped`, (_, groups) => {
      groups.push([{
        type: `options`,
        name: `options`,
        options: { openInTab: true },
        inputPath: path.resolve(wxt.config.root, `./index.html`),
        outputDir: wxt.config.outDir,
        skipped: false,
      }])
    })
  },
})
```

## CDN资源本地化处理

由于原来项目中的Tex数学公式转svg、一键发布功能，引用了外部CDN js资源，而这个行为是被浏览器插件禁止的。其实包括内联的script内容也是不被允许的。

这一步我通过阅读 `WXT` 框架自己实现的 `devHtmlPrerender` vite 插件源码进行模仿解决的。从它的源码中可以找到相关介绍说明

```markdown
// Replace inline script with virtual module served via dev server.
// Extension CSP blocks inline scripts, so that's why we're pulling them out.
```
### 2.1 将CDN JS转换为虚拟文件

这一步的做法是将 CDN js资源下载下来，保存到内存中，然后修改原始CDN js链接，改为引用请求虚拟文件地址。 参考：[虚拟模块相关说明](https://cn.vite.dev/guide/api-plugin.html#virtual-modules-convention)

大部分的代码思路是参考了 `devHtmlPrerender` 插件的。

```ts
// Stored outside the plugin to effect all instances of the htmlScriptToVirtual plugin.
const inlineScriptContents: Record<number, string> = {}
export function htmlScriptToVirtual(
  config: wxt.ResolvedConfig,
  getWxtDevServer: () => wxt.WxtDevServer | undefined,
): vite.PluginOption {
  const virtualInlineScript = `virtual:md-inline-script`
  const resolvedVirtualInlineScript = `\0${virtualInlineScript}`

  const server = getWxtDevServer?.()
  return [
    {
      name: `md:dev-html-prerender`,
      apply: `build`,
      async transform(code, id) {
        if (
          server == null
          || !id.endsWith(`.html`)
        ) {
          return
        }
        const { document } = parseHTML(code)
        // Replace inline script with virtual module served via dev server.
        // Extension CSP blocks inline scripts, so that's why we're pulling them out.
        const promises: Promise<void>[] = []
        const inlineScripts = document.querySelectorAll(`script[src^=http]`)
        inlineScripts.forEach(async (script) => {
          promises.push(new Promise<void>((resolve) => {
            const url = script.getAttribute(`src`) ?? ``
            doFetch(url).then((textContent) => {
              const hash = murmurHash(textContent)
              inlineScriptContents[hash] = textContent
              script.setAttribute(`src`, `${server.origin}/@id/${virtualInlineScript}?${hash}`)
              if (script.hasAttribute(`id`)) {
                script.setAttribute(`type`, `module`)
              }
              resolve()
            })
          }))
        })
        await Promise.all(promises)
        const newHtml = document.toString()
        config.logger.debug(`transform ${id}`)
        config.logger.debug(`Old HTML:\n${code}`)
        config.logger.debug(`New HTML:\n${newHtml}`)
        return newHtml
      },
    },
    {
      name: `md:virtualize-react-refresh`,
      apply: `serve`,
      resolveId(id) {
        // Resolve inline scripts
        if (id.startsWith(virtualInlineScript)) {
          return `\0${id}`
        }

        // Ignore chunks during HTML file pre-rendering
        if (id.startsWith(`/chunks/`)) {
          return `\0noop`
        }
      },
      load(id) {
        // Resolve virtualized inline scripts
        if (id.startsWith(resolvedVirtualInlineScript)) {
          // id="virtual:md-inline-script?<hash>"
          const hash = Number(id.substring(id.indexOf(`?`) + 1))
          return inlineScriptContents[hash]
        }

        // Ignore chunks during HTML file pre-rendering
        if (id === `\0noop`) {
          return ``
        }
      },
    },
  ]
}
```

然后需要把这个插件引入到 `wxt` 中，同样是刚才的 `build-extension.ts` 模块加入这个插件，

```ts
addViteConfig(wxt, () => ({
  plugins: [
    htmlScriptToVirtual(wxt.config, () => wxt.server),
  ],
}))
```

### 2.2 解决Vue DevTools引用地址错误的问题

在此之前以为浏览器插件形式下是无法使用`Vue DevTools` 的，也无从下手解决这个问题，一般是屏蔽掉这个插件。
```ts
import vueDevTools from 'vite-plugin-vue-devtools'
export default defineConfig({
	plugins: [
    	vueDevTools()
    ]
})
```

 自从了解了vite虚拟模块虚拟文件后，观察Vue DevTools引入后在html中的引用形式，
 
 ```html
<script type="module" src="/@id/virtual:vue-devtools-path:overlay.js"></script>
<script type="module" src="/@id/virtual:vue-inspector-path:load.js"></script>
```
解决方案就比较明显了，调整script引用链接，在url前加`${server.origin}` , 调整后html中的这两个脚本就是

```html
<script type="module" src="http://localhost:3000/@id/virtual:vue-devtools-path:overlay.js"></script>
<script type="module" src="http://localhost:3000/@id/virtual:vue-inspector-path:load.js"></script>
```
这样子才能发起http请求，走vite server的虚拟模块匹配逻辑。要不然，其实请求的最终地址是，类似
```markdown
chrome-extension://loflmllnppnghdegodmmlpdfkgjabkge/@id/virtual:vue-devtools-path:overlay.js
```
当然就无法处理的。

解决方案：用vite 插件做一个hack，

```ts
export function vueDevtoolsHack(
  config: wxt.ResolvedConfig,
  getWxtDevServer: () => wxt.WxtDevServer | undefined,
): vite.Plugin {
  const server = getWxtDevServer?.()
  return {
    name: `md:vue-devtools-hack`,
    apply: `build`,
    transformIndexHtml: {
      order: `post`,
      handler(html) {
        const { document } = parseHTML(html)
        const inlineScripts = document.querySelectorAll(`script[src^='/@id/virtual:']`)
        inlineScripts.forEach((script) => {
          const src = script.getAttribute(`src`)
          const newSrc = `${server?.origin}${src}`
          script.setAttribute(`src`, newSrc)
        })
        const newHtml = document.toString()
        config.logger.debug(`Old HTML:\n${html}`)
        config.logger.debug(`New HTML:\n${newHtml}`)
        return newHtml
      },
    },
  }
}

```

相关讨论，[how to use vue-devtools in chrome extension options page #727](https://github.com/vuejs/devtools/issues/727)

## 打包构建方案

### 3.1 虚拟文件持久化

经过同事的提醒，刚开始的调研方向是将虚拟文件打包进最终的产物，但是忽略了dev和build模式的区别。

在dev模式下，html页面中的script引用地址被修改为http请求虚拟文件地址，但是build模式下并不会直接沿用dev的这个过程，而且进一步分析，build模式下，script引用地址也不应该修改成http地址形式，应该是本地地址或者相对地址。

### 3.2 将CDN JS转换为本地文件
经过这一番分析后，还是决定针对build模式再重新写一个vite插件来处理，同时一并考虑保存地址及html引用地址，插件如下，

```ts
export function htmlScriptToLocal(
  wxt: wxt.Wxt,
): vite.Plugin {
  return {
    name: `md:build-html-prerender`,
    apply: `build`,
    transformIndexHtml: {
      order: `pre`,
      async handler(html) {
        const { document } = parseHTML(html)
        const promises: Promise<void>[] = []
        const httpScripts = document.querySelectorAll(`script[src^=http]`)
        if (httpScripts.length > 0) {
          httpScripts.forEach(async (script) => {
            /* eslint-disable no-async-promise-executor */
            promises.push(new Promise<void>(async (resolve) => {
              const url = script.getAttribute(`src`) ?? ``
              if (url?.startsWith(`http://localhost`)) {
                resolve()
                return
              }
              const textContent = await doFetch(url)
              const hash = murmurHash(textContent)
              const jsName = url.match(/\/([^/]+)\.js$/)?.[1] ?? `.js`
              const fileName = `${jsName.split(`.`)[0]}-${hash}.js`
              // write to file
              const outFile = path.resolve(wxt.config.outDir, `./${fileName}`)
              await writeFile(outFile, textContent, `utf8`)
              script.setAttribute(`src`, `/${fileName}`)
              // script.setAttribute(`type`, `module`)
              resolve()
            }))
          })
        }

        // Replace inline script with virtual module served via dev server.
        // Extension CSP blocks inline scripts, so that's why we're pulling them
        // out.
        const inlineScripts = document.querySelectorAll(`script:not([src])`)
        if (inlineScripts.length > 0) {
          inlineScripts.forEach(async (script) => {
            promises.push(new Promise<void>(async (resolve) => {
              // Save the text content for later
              const textContent = script.textContent ?? ``
              const hash = murmurHash(textContent)
              const fileName = `md-inline-${hash}.js`
              // write to file
              const outFile = path.resolve(wxt.config.outDir, `./${fileName}`)
              await writeFile(outFile, textContent, `utf8`)
              // Replace unsafe inline script
              const virtualScript = document.createElement(`script`)
              // virtualScript.type = `module`
              virtualScript.src = `/${fileName}`
              script.replaceWith(virtualScript)
              resolve()
            }),
            )
          })
        }
        await Promise.all(promises)
        const newHtml = document.toString()
        wxt.config.logger.debug(`Old HTML:\n${html}`)
        wxt.config.logger.debug(`New HTML:\n${newHtml}`)
        return newHtml
      },
    },
  }
}
```

相关讨论：[how to deal with cdn js files #1255](https://github.com/wxt-dev/wxt/discussions/1255)

最终引入自实现的插件配置如下，
```ts
addViteConfig(wxt, () => ({
  plugins: [
    htmlScriptToVirtual(wxt.config, () => wxt.server),
    vueDevtoolsHack(wxt.config, () => wxt.server),
    wxt.config.command === `build`
      ? htmlScriptToLocal(wxt)
      : undefined,
  ],
}))
```

## 解决打包体积过大问题

这是一个遗留了大半年的问题，通过vite等官方指引配置`build?.rollupOptions.output.manualChunks` 并没有效果，而且还可能导致build 出错。。
相关讨论[file of zip output is too large that expend firefox 4MB limit #765](https://github.com/wxt-dev/wxt/discussions/765)

其中打包后单文件体积超过4MB的是options这个chunk，意思就是 option entrypoint 把所有依赖都打包一起了。 本次研究出来cdn js本地化的方案过程中，对`WXT`框架源码增加了不少认识，经过一番hacking，算是找到了改变打包配置的`WXT` hook。

解决方案，选择哪些依赖单独分包，可以通过`npx wxt build --analyze`命令得到报告。
```ts
wxt.hook(`vite:build:extendConfig`, (_, config) => {
  if (config.build?.rollupOptions?.input && config.build?.rollupOptions?.input) {
    const input = config.build?.rollupOptions.input as Record<string, string>
    if (input.options) {
      const output = config.build?.rollupOptions.output as FakeRollupOptions
      output.manualChunks = (id) => {
        if (id.includes(`prettier`)) {
          return `prettier-chunk`
        }
        if (id.includes(`highlight.js`)) {
          return `highlight-chunk`
        }
      }
    }
  }
})
```

之所以不能在全局配置`manualChunks`，是由于`WXT`是多模块的打包方式，在`background`这个entrypoint打包的时候配置分包方式会由于不支持而报错。而针对option entrypoint单独修改，在`WXT`调用`vite#build`方法前修改（请在`WXT`源码中搜索关键方法：buildEntrypoints），就可以顺利完成分包。分包的效果是options chunk主包体积减少，其他分包`/chunks`目录，被options chunk引入。

## 结语

本次的改造保留了原有项目100%的功能，非常不容易，主要的举措是动态配置Entrypoint入口文件，利用Vite插件处理cdn js资源，hack Vue DevTools插件地址引用错误问题。对于dev和build两种模式区别处理cdn js资源的做法或许可以合并成一种，有待后续研究改进。

这款插件的最大好处就是引入了公众号素材库（图床）功能，在撰写公众号文章的时候不用使用其他图床，而且通过openapi形式与公众号素材库接口打通，粘贴图片即可上传并使用，包括预览效果都是支持的。

相关使用教程，请参考[公众号文章编辑器插件教程](https://mpmd.pages.dev/tutorial/)。新版本插件即将发布审核上架，敬请期待！

---

<center>
    <img src="https://honwhy.wang/_astro/qrcode_mp.Di5Nhs5-_Z1czA9A.webp" style="width: 140px;">
</center>
