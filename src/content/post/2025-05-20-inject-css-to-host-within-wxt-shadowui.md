---
title: "Inject CSS into Host Page from WXT ShadowRoot UI"
description: "With 3–4 years of browser extension development experience, I consider the ShadowRoot API the optimal approach for content scripts."
publishDate: "20 May 2025"
tags: ["browser extension", "wxt", "shadow dom", "css"]
---

> In browser extension development, isolating CSS is essential to avoid unintended style conflicts. The ShadowRoot API is an ideal tool for encapsulating your UI.

As a WXT framework practitioner with several years of browser extension experience, I’ve found the ShadowRoot API to be the best way to implement content scripts. It allows your UI to be injected without affecting—or being affected by—the host page’s styles.

However, in some real-world scenarios, you may need to intentionally break this encapsulation to modify the host page's appearance. This post explains how to do that cleanly using WXT.

## Creating a ShadowRoot UI in WXT

Here’s an example of using `createShadowRootUi` within a WXT content script:

```ts
// 1. Import your styles and components
import './style.css';
import { createApp } from 'vue';
import App from './App.vue';

export default defineContentScript({
  matches: ['<all_urls>'],
  cssInjectionMode: 'ui', // 2. Enable ShadowRoot UI mode

  async main(ctx) {
    // 3. Create the ShadowRoot UI
    const ui = await createShadowRootUi(ctx, {
      name: 'wechat-toc',
      position: 'inline',
      anchor: 'body',
      onMount: (container) => {
        const app = createApp(App);
        app.mount(container);
        return app;
      },
      onRemove: (app) => {
        app?.unmount();
      },
    });

    // 4. Mount it
    ui.mount();
  },
});
```

As expected, the imported styles will be scoped within the Shadow DOM.

![figure-1](https://wsrv.nl?url=https%3A%2F%2Fmmbiz.qpic.cn%2Fmmbiz_png%2F1bicgWjxTkWaV8s59EtIorIaTP4RxYN17wOaibaXMJpq8WT0VibOhAibyM6N8TAalSoicagwsoCyX0yLlbCAn64KmgQ%2F640%3Ffrom%3Dappmsg)

## Modifying Host Page Styles

Since styles imported into Vue SFCs or defined via `<style>` tags are scoped inside the ShadowRoot, applying global styles (e.g., changing the host page’s `font-family`) requires a different strategy.

To inject global styles, manually append a stylesheet to the host document:

```ts
const hostCssUrl = browser.runtime.getURL(`/injected.css`) // 确保路径正确

const hostLink = document.createElement(`link`)
hostLink.rel = `stylesheet`
hostLink.href = hostCssUrl
document.head.appendChild(hostLink)
```

The `injected.css` file might contain something like:

```css
body.wx_wap_page.wre-serif-fonts {
  font-family:
    SimSun,
    Times New Roman,
    Georgia,
    Merriweather,
    Playfair Display;
}
body.wx_wap_page.text-justify p {
  text-align: justify;
}
```

## Toggling Styles Dynamically with Storage Watchers

WXT provides a built-in [storage](https://wxt.dev/storage.html) API that supports reactive watching:

```ts
const unwatch = storage.watch<number>('local:counter', (newCount, oldCount) => {
  console.log('Count changed:', { newCount, oldCount });
});
```

You can use this to dynamically toggle the global font style based on user settings:

```ts
function handleSeriFont() {
  if (settings.value.useSerifFont === `1`) {
    if (!document.body.classList?.contains(`wre-serif-fonts`)) {
      addClass(document.body, `wre-serif-fonts`)
    }
  }
  else {
    removeClass(document.body, `wre-serif-fonts`)
  }
}
```

This way, even though your app UI is encapsulated in a ShadowRoot, you can still control global styles when needed.

## Further Reading & Inspiration

- [inject-script](https://github.com/wxt-dev/examples/blob/main/examples/inject-script/entrypoints/content.ts)
- [Re-use of css from options/pop-up page also in an injection script with ShadowRootUI?](https://github.com/wxt-dev/wxt/discussions/1548)