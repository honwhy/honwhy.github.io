---
title: "More on @crxjs/vite-plugin"
description: "After using @crxjs/vite-plugin to build chrome extension for while, I want to share some of my experience."
publishDate: "31 Dec 2023"
tags: ["chrome", "vite", "vue"]
---

## Get Started with Vue
[quick guide about using @crxjs/vite-plugin to build chrome extension](https://crxjs.dev/vite-plugin/getting-started/vue/create-project), after which you have to make some changes as to run the project smoothly.

### Vite and HMR
Only `npm run dev` may not be all, you have to set the hmr port,
```
  server: {
    port: 5173,
    strictPort: true,
    hmr: {
      clientPort: 5173,
    }
  }
```
### Use Typescript
remember import global definition is an important step. When building the project follow `@crxjs/vite-plugin` guide please choose Typescript option, after which import `@types/chrome` node_module.
```
npm i @types/chrome -D
```
some changes to `tsconfig.json`,
```
{
  "compilerOptions": {
    "types": ["@types/chrome"],
  }
}
```

## Use Vue in content_scripts
There is not direct way to do so, but you can mount Vue app to some newly created element which just appended to some existed element of websites, which content_scripts shall run at.
```
        {
            "js": [
                "src/contents/douban.ts"
            ],
            "matches": [
                "https://book.douban.com/subject/*"
            ]
        },
```
create element, and mounte Vue app,
```
import { createApp } from 'vue'
import App from './Douban.vue'

const aside = document.querySelector('.aside')
const el = document.createElement('div')
el.id = 'wl-ephohljfnflennmpegcmjnchjkghdddp'
aside?.prepend(el)
createApp(App).mount(el)

console.log('douban.ts loaded')
```

## Message Communication
The simple way of message communication is use `chrome.runtime` apis. Client send out message by `chrome.runtime.sendMessage`, and Server listen on `chrome.runtime.onMessage.addListener`. Normally message listeners run at background,
```
    "background": {
        "service_worker": "src/background/index.ts",
        "type": "module"
    },
```
but you can set message listeners in content_scripts, and some changes have to be make,
Client should specify the target browser active tab,
```

  chrome.tabs.query(
    {
      active: true,
      currentWindow: true,
    },
    (tabs) => {
      const tab = tabs[0] //这个就是当前活动的页面
      chrome.tabs.sendMessage(tab.id!, message, (res) => {
          console.log('popup=>queryBook', res)
          if (res != null) {
            isbn.value = res.isbn
            book.value = res.book
            source.value = res.from
          }
        })
    })
```
Another notice, messages do not fan out, just on copy, in other word, you should not set up two listeners in the same content_scripts, you should combine them all together.
```
// message hook
export function useMessage (callback: (message: any, sender: chrome.runtime.MessageSender, sendResponse: (response?: any) => void) => void) {
  chrome.runtime.onMessage.addListener(callback)
}
```
```
// Douban.vue
function onMessage(
  message: WeMessage<any>,
  sender: chrome.runtime.MessageSender,
  sendResponse: (response?: any) => void
) {
  // message.action === 'queryBook'
  // message.action === 'queryRecommends'
  console.log('receive request', sender, message)
  switch (message.action) {
    case 'queryBook':
      onQueryBook(sendResponse)
      break
    case 'queryRecommends':
      onQueryRecommends(sendResponse)
      break
    case 'contextMenuSearch':
      handleContextMenuSearch(message)
      sendResponse({ message: 'thanks', from: 'search-bar' })
      break
    default:
      return true
  }
}
```



## How to Parse Response
Be careful of the `cors` thing, for example, `Douban.vue` runs at `book.douban.com` website page, it can't just use `fetch` to fetch `szlib.org.cn` xhr responses, where that's background and message communications come to the stage. However, when only the HTML response we can get from target webiste, we would not parse them in background. Because background don't have `DOMParser`, and yeah content_script or popups have, the solution is obvious, messages to background, fetch and response back the origin html response, and the parse html response in content_script or popups by `DOMParser`.

## Use stores
Combines Vue app store and Chrome extension storage,
```
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useCommonStore = defineStore('common', () => {
  const city = ref('sz')
  const avatar = ref('')
  function setCity (val: string) {
    city.value = val
  }
  function setAvatar (val: string) {
    avatar.value = val
  }

  return { city, setCity, avatar, setAvatar }
})
```
```
const commonStore = useCommonStore()
onMounted(() => {
  initData()
})
function initData() {
  chrome.storage.local.get(
    ['city', 'avatar'],
    function (val: Record<string, any>) {
      console.log('chrome.storage.local.get', val)
      if (val != null) {
        console.log('city from storage', val)
        commonStore.city = val.city
        commonStore.avatar = val.avatar ?? ''
      }
    }
  )
}
```
    