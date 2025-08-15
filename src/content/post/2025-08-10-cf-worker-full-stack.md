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
npm create cloudflare@latest -- cards --framework=vue
```

初始化后，整个工程已经配置好`vite.config.ts`， 规划好前端代码部分在`src`目录 以及后端代码部分在`server`目录。
整个项目可以看作以前端项目为主体，后端项目为辅的设计。

### 引入better-auth
这一步的目的是使用`better-auth`的账号及登录方面的表设计。主要涉及的表有 `user` `account` `verification`，另外和会话相关的我们开启次级存储，使用`cloudflare workers`的 `KV`。

配置`better-auth` 密钥，这一步和密码hash 有关，具体源码没有深究
```
BETTER_AUTH_SECRET=
```

如果是普通的数据库，不像`cloudflare workers` 这样子需要首先bindging，然后通过http 接口触发，再从context 中获取，那么配置使用`better-auth` 挺简单的。

配置好DB，
```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "@/db"; // your drizzle instance
 
export const auth = betterAuth({
    database: drizzleAdapter(db, {
        provider: "pg", // or "mysql", "sqlite"
    }),
});
```
使用cli 工具，`npx @better-auth/cli generate` 生成schema，`npx @better-auth/cli migrate` 在数据库中创建表。


由于`cloudflare workers` 的特殊性，这里采用网上的开源方案[cf-script](https://github.com/Thomascogez/cf-script)，先折衷搞到shema，

```ts
import { getAdapter } from "better-auth/db";
import { writeFile } from "node:fs/promises";
import { resolve } from "node:path";
import { initBetterAuth } from "../server/lib/auth";
import { generateDrizzleSchema } from "./_vendors/drizzle-generator";

export default async (env: unknown) => {
	const betterAuth = initBetterAuth(env);

	const output = await generateDrizzleSchema({
		adapter: await getAdapter(betterAuth.options),
		options: betterAuth.options,
		file: resolve(import.meta.dirname, "../db/schema/better-auth-schemas.ts")
	});

	await writeFile(output.fileName, output.code ?? "");

	console.log(`Better auth schema generated successfully at (${output.fileName} 🎉`);
};
```

然后初始化BetterAuth对象是在全局路由中处理，
```ts
// server/index.ts
type Bindings = {
  DB: D1Database; // Assuming your D1 binding is named 'DB'
  KV: KVNamespace; // Assuming your KV binding is named 'KV'
};

// 扩展 Context 的变量类型
interface Variables {
  db: ReturnType<typeof drizzle>;
}
const app = new Hono<{ Bindings: Env; Variables: Variables }>()

// 全局 middleware 示例
app.use('*', async (c, next) => {
  // 这里可以做一些全局处理，比如日志、鉴权等
  // console.log('Global middleware: 请求路径', c.req.path)
  const db = drizzle(c.env.DB)
  // console.log('Database connection established:', db)
  c.set('db', db) // 挂载 db 实例到 Context
  await next()
})
```
上面的处理，已经在全局开始设置了`db` 实例，然后再某些路由handler 中处理的时候，调用下面的方法就可以拿到`betterAuth`实例。

```ts
export const auth = (env: Env): ReturnType<typeof betterAuth> => {
  const db = drizzle(env.DB);

  return betterAuth({
    ...betterAuthOptions,
    database: drizzleAdapter(drizzle(env.DB), {
      provider: "sqlite",
      schema: {
        user: schema.userTable,
        account: schema.accountTable,
        session: schema.sessionTable,
        verification: schema.verificationTable,
      },
    }),
    secondaryStorage: {
			get(key) {
				return env.KV.get(key);
			},
			set(key, value, ttl) {
				return env.KV.put(key, value, { expirationTtl: ttl });
			},
			delete(key) {
				return env.KV.delete(key);
			}
		},
    baseURL: env.BETTER_AUTH_BASE_URL,
    secret: env.BETTER_AUTH_SECRET,
    emailAndPassword: {    
      enabled: true
    },
    // Additional options that depend on env ...
  });
};
```

比如说登录，

```ts
authRoutes.post('/login', async (c) => {
  const db = c.get('db') // 确保 db 实例已挂载到 Context
  const requestBody = await c.req.json();
  const JWT_SECRET = c.env.JWT_SECRET!; 
    try {
      const { email, password, username } = requestBody;
      
      // 支持email或username字段
      const loginIdentifier = email || username;
  
      // 验证输入
      if (!loginIdentifier || !password) {
        return c.json({
          success: false,
          message: '邮箱/用户名和密码都是必填项'
        }, 400);
      }
      const resp = await auth(c.env).api.signInEmail({
        body: {
          email: requestBody.email,
          password: requestBody.password,
        }
      })
      if (resp && resp.user) {
        const user = resp.user;
        // 生成JWT token
        const token = await generateToken(JWT_SECRET, user.id);
              // 获取新创建的用户
        const existingUser = await db.select().from(schema.userTable).where(eq(schema.userTable.id, user.id)).limit(1).get();
  
        return c.json({
          success: true,
          message: '登录成功',
          data: {
            user: {
              id: user.id,
              username: user.name,
              email: user.email,
            },
            token
          }
        });
      }
            return c.json({
        success: false,
        message: '用户名或密码错误'
      }, 400);
    } catch (error) {
      console.error('Login error:', error);
      return c.json({
        success: false,
        message: '登录失败，请稍后重试',
        error,
      }, 500);
    }
});
```

使用`betterAuth`实例的API方法比自己手写各类处理：校验账密、创建会话要简单多了。`betterAuth`提供了丰富的API，比如登录 `signInEmail`， 注册`signUpEmail` 等等。

## 参考文档

- [cloudflare workers](https://developers.cloudflare.com/)
- [cloudflare workers vue](https://developers.cloudflare.com/workers/framework-guides/web-apps/vue/)
- [cloudflare workers d1](https://developers.cloudflare.com/d1/)
- [cloudflare workers kv](https://developers.cloudflare.com/kv/)
- [Hono](https://hono.dev)
- [better-auth](https://www.better-auth.com/)
- [drizzle](https://orm.drizzle.team/)
- [cf-script](https://github.com/Thomascogez/cf-script)