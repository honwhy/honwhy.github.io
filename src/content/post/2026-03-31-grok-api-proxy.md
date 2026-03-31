---
title: "在国内网络丝滑访问grok服务"
description: "在国内网络丝滑访问grok服务，你可以像我这样子，通过cloudflare(赛博菩萨) 进行转发，在本地使用Chatbox APP进行访问。"
publishDate: "31 Mar 2026"
tags: ["grok", "chatbox", "x"]
---

# 在国内网络丝滑访问grok服务
>在国内网络丝滑访问grok服务，你可以像我这样子，通过cloudflare(赛博菩萨) 进行转发，在本地使用Chatbox APP进行访问。

需要准备的内容有，cloudflare账号，独立域名，grok服务，ChatBox APP。
## cloudflare 账号
### 注册并登录
建议自行搜索答案，或私下交流。
### 将域名托管到cloudflare
建议自行搜索答案，或私下交流。
### 搭建部署转发服务
在cloudflare，新建一个workers 服务，部署这段代码。
```
// worker.js - Cloudflare Worker 脚本，用于中转 xAI Grok API 请求
// 作者: w0x7ce
// 描述: 该脚本通过 Cloudflare Worker 提供 xAI Grok API 的代理服务，客户端需提供自己的 API 密钥。

// xAI Grok API 的官方端点
const XAI_API_URL = 'https://api.x.ai/v1/chat/completions';

/**
 * 处理传入的 HTTP 请求并转发到 xAI API
 * @param {Request} request - 客户端发来的请求对象
 * @returns {Response} - 转发后的响应或错误信息
 */
async function handleRequest(request) {
  // 从请求头中提取客户端提供的 Authorization（API 密钥）
  const clientAuth = request.headers.get('Authorization');
  
  // 检查是否提供了有效的 Bearer 密钥
  if (!clientAuth || !clientAuth.startsWith('Bearer ')) {
    return new Response(
      JSON.stringify({
        error: 'Missing or invalid Authorization header. Please provide a valid Bearer token.',
      }),
      {
        status: 401, // 未授权
        headers: { 'Content-Type': 'application/json' },
      }
    );
  }

  // 复制客户端的请求头，并确保 Content-Type 为 JSON
  const headers = new Headers(request.headers);
  headers.set('Content-Type', 'application/json');

  // 获取请求体（JSON 格式）
  const body = await request.text();

  // 构造转发请求，保持原始方法、头和体
  const proxyRequest = new Request(XAI_API_URL, {
    method: request.method,
    headers: headers,
    body: body,
  });

  // 转发请求到 xAI API 并获取响应
  const response = await fetch(proxyRequest);

  // 检查请求是否要求流式响应
  let stream = false;
  try {
    const requestData = JSON.parse(body);
    stream = requestData.stream || false;
  } catch (e) {
    // 如果 JSON 解析失败，默认非流式
    console.log('Failed to parse request body:', e);
  }

  // 根据 stream 参数返回流式或非流式响应
  if (stream) {
    // 流式响应：直接返回原始响应流
    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: response.headers,
    });
  } else {
    // 非流式响应：等待完整响应后返回
    const responseData = await response.text();
    return new Response(responseData, {
      status: response.status,
      statusText: response.statusText,
      headers: { 'Content-Type': 'application/json' },
    });
  }
}

// 监听 Cloudflare Worker 的 fetch 事件
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});
```
这是一个开源项目，我fork了到个人仓库了，[grok-api-proxy](https://github.com/honwhy/grok-api-proxy)

<del>这个项目只实现了/chat/completions 接口，没有实现/models 接口，不过问题不大。</del>
### 配置转发域名
比如你的独立域名是 `xxx.wang`，你可以取一个`grok.xxx.wang`的域名，添加到上一步的workers 服务中。cloudflare 会自动帮你做好域名解析，生效也很快的。
如果没有独立域名，你可以拿到cloudflare 分配的一个域名，类似`grok-123.account.workers.dev`，但是这样的域名并不好用，解析可能被污染了。
>通过自己的独立域名进行访问就没有这个担忧。

配置截图，
![](https://wsrv.nl?url=http://mmbiz.qpic.cn/sz_mmbiz_png/iaQoVF64VAQfc94E6eLJj8ZDv4DN7rgTJVMzhaqcbSuiaWTx6WmmOcYmhbkWJwiaOkgPmD0m1UDHWnDvFBRticEteXLFa8TSxeWe9XI7zIGS69w/0?from=appmsg)

## grok 服务
登录https://console.x.ai/ 服务，添加api key，

![](https://wsrv.nl?url=http://mmbiz.qpic.cn/sz_mmbiz_png/iaQoVF64VAQfPkk33JfbmkP2k0bhk7jCIPzTZnzysS39icfWeW4ljxqt2MjoCeoNDABrkLaZrVHRbNSOR4hfAAgERVM6HkLQctYJBuko2ibYmY/0?from=appmsg)
这一步获取到api key要记得备份保存起来。
配置信用卡进行支付，先试水购买 $5 。
<del>你可能需要一张信用卡（貌似有香港的信用卡会更好，私信交流哈）</del>

## Chatbox
Chatbox 的官方是这个，[Chatbox](https://chatboxai.app/zh)， 大陆app store 不确认是否官方上架的。虽然但是我已经用起来了。
*--免责声明--*
![](https://wsrv.nl?url=http://mmbiz.qpic.cn/mmbiz_jpg/iaQoVF64VAQfVHcq9YTcKVQO7zonQYvRCMcbuEN1Gsop27oWWpBzZvrpks65q33zvyich6eXiauajYQ8GtrJF0nJua18lRTVPAvfjDFvoyzpK4/0?from=appmsg)
*--免责声明--*
### 配置模型
点击APP 的左上角弹出左侧菜单，选择模型提供方，添加一个自己的模型
- 名称，随意填
- API模型，**OpenAI API 兼容**
- API密钥，grok 服务添加api key的时候获得
- API主机，类似`https://grok.xxx.wang/v1`
- 改善网络兼容性（勾选上）
手工添加一个模型，模型ID是`grok-4-1-fast`
![](https://wsrv.nl?url=http://mmbiz.qpic.cn/sz_mmbiz_png/iaQoVF64VAQcqF4goTa8qfW4zeIMqqLnuTTGle0s1qqibNy7c61g4H7AzANApk7hXibzbiblQgwZ24octCicXG05T8I5pj3sHqjPNDSsSfZUHMIw/0?from=appmsg)
### 测试效果

![](https://wsrv.nl?url=http://mmbiz.qpic.cn/sz_mmbiz_jpg/iaQoVF64VAQeyibha3uicOKyNYFF3KpDH8n4PnHu20BHicsOID0uRA8nf59tSCqMzjXrGQSuUAdhqcXjerUSN2r4PKdC0TlIpZBKb2XVYjWXqrw/0?from=appmsg)
由于grok和X的联系，你可以这么问，（3-28发问可以最新的知识是3-27的，已经是很实时的了）

![](https://wsrv.nl?url=http%3A%2F%2Fmmbiz.qpic.cn%2Fmmbiz_jpg%2FiaQoVF64VAQdHVy0RvbbHkzdac0BZ5ZPJApBLS1Uh1qDYzkkFRKM1vYt7hysPiboyZF3p07yhuJicBzL7ric7tRcU2nqYojZ4RO3cfcUmy0eLLI%2F0%3Ffrom%3Dappmsg)

