---
title: "cloudflare workers 全栈化实战"
description: "cloudflare workers 全栈化实战。利用`cloudflare workers` 实现一个全栈化项目的部署，技术选型前端使用`Vue`，后端除了`cloudflare workers`全家桶，还是用`Hono`，`better-auth`以及`drizzle`"
publishDate: "10 Aug 2025"
tags: ["cloudflare workers", "d1", "better-auth", "drizzle"]
draft: true
---

> 经过多次的调研，本人打算利用`cloudflare workers` 实现一个全栈化项目的部署，技术选型前端使用`Vue`，后端除了`cloudflare workers`全家桶，还是用`Hono`，`better-auth`以及`drizzle`。

## 初始化框架

### 搭建全栈化项目

```
npm create cloudflare@latest -- psychology-cards --framework=vue
```