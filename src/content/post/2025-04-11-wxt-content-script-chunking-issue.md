---
title: "WXT Content Script Chunking Issue"
description: "Large content scripts in WXT browser extensions can cause problems like slow updates and crashes, especially on Windows. Firefox also limits file sizes."
publishDate: "11 Apr 2025"
tags: ["chrome", "wxt", "content script", "CustomEvent"]
---

## Intro

[WXT](https://wxt.dev), which is `An open source tool that makes web extension development faster than ever before`. In this article, we'll examine code splitting challenges specific to content scripts and present an optimized approach to using WXT's injectScript utility.

This technical deep dive adapts key concepts from the original Chinese article [浏览器插件ContentScript分包问题](https://mp.weixin.qq.com/s/Tt-cJRANVxymrj1LcBco0w)

## Summary

This article addresses the challenge of managing large dependencies in WXT-based browser extension content scripts. Large content script sizes can negatively impact development, particularly with HMR (Hot Module Replacement) and on Windows, potentially causing crashes.  Firefox also has file size limits.

The author explains that unlike extension options pages, content scripts lack a straightforward entry point like an HTML file, making ESM implementation difficult.  The suggested workaround involves manual code splitting using WXT's `injectScript` to load dependencies, such as `ajax-hook`, into the web page's context.  The article highlights the problem of accessing injected script variables from the content script (demonstrating that simply assigning to `window` or using `document.body.setAttribute` is insufficient).

The recommended solution is to use `CustomEvent` for communication between the injected script and the content script. The injected script listens for events, performs operations (like Markdown rendering with `marked`, `mermaid`, and `katex`), and then dispatches a new event with the result back to the content script.  The content script listens for this result event.  This approach allows the content script to use the functionality of the injected, pre-bundled code.

## Key Points

- Large content scripts can cause performance issues and crashes in WXT, and exceed Firefox limits.

- WXT's `injectScript` can load code into the page context, but direct variable access is problematic.

- `CustomEvent` provides a reliable communication channel between injected scripts and content scripts in WXT.

- The article provides a concrete example of rendering Markdown in a content script using this method.

## Simplified English Translation

Large content scripts in WXT browser extensions can cause problems like slow updates and crashes, especially on Windows. Firefox also limits file sizes.

Regular web pages have an HTML entry point, but content scripts don't, making it hard to use modern JavaScript modules (ESM). A workaround is to split the code: use `injectScript` to load some code separately.

This separate code can't easily share variables with the main content script.  The article shows that setting variables on `window` or using `document.body.setAttribute` doesn't work well.

## Show Codes

[md-mermaid](https://github.com/honwhy/md-mermaid)

split chunks,
```ts
// injected.ts
import { marked } from 'marked'
import mermaid from 'mermaid'
import markedKatex from 'marked-katex-extension'
export default defineUnlistedScript(() => {
  console.log(`Hello from the injected script!`)
  marked.use({
    renderer: {
      code: code => code.lang === `mermaid`
        ? `<pre class="mermaid">${code.text}</pre>`
        : `<pre><code>${code.text}</code></pre>`,
    },
  },
  )
  marked.use(markedKatex({
    throwOnError: false
  }))
  window.addEventListener(`marked-request`, (s) => {
    const cs = s as CustomEvent
    const html = marked(cs.detail as string)
    window.dispatchEvent(new CustomEvent(`marked-response`, {detail: html}))
  })
  mermaid.initialize({ startOnLoad: false, theme: `dark`, darkMode: true })
  window.addEventListener(`mermaid-run`, () => {
    mermaid.run()
  })
})
```

add injected.js into web page,

```ts
// entrypoints/example-ui.content/index.ts
import { createApp } from 'vue'
import App from './popup/App.vue'

export default defineContentScript({
  matches: [`https://github.com/*`],

  async main(ctx) {
    await injectScript(`/injected.js`, {
      keepInDom: false
    })
    const ui = createIntegratedUi(ctx, {
      position: `inline`,
      anchor: `body`,
      onMount: (container) => {
        // Create the app and mount it to the UI container
        const app = createApp(App)
        app.mount(container)
        return app
      },
      onRemove: (app) => {
        // Unmount the app when the UI is removed
        app?.unmount()
      },
    })

    // Call mount to add the UI to the DOM
    ui.mount()
  },
})
```

don't forget the manifest.json changes,

```ts
// wxt.config.ts
...
web_accessible_resources: [
  {
    resources: [`*.png`, `*.svg`, `injected.js`],
    matches: [`<all_urls>`],
  },
],
...
```