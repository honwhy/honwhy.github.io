---
title: "微信 Markdown 编辑器部署Cloudflare Workers简单方式"
description: "这可能是最简单部署方式了，只需要一行命令即可搞定。注册登录cloudflare后，打开md项目，只需要执行pnpm web wrangler:deploy命令即可完成部署"
publishDate: "06 Oct 2025"
tags: ["微信", "公众号", "cloudflare", "workers"]

---
# 微信 Markdown 编辑器部署Cloudflare Workers简单方式

> 这可能是最简单部署方式了，只需要一行命令即可搞定。

![](https://fastly.jsdelivr.net/gh/bucketio/img14@main/2025/10/06/1759731375335-3e70409b-3076-4b95-9c91-9b574d2ae69b.png)

### fork 或者 clone 项目
前往[doocs/md](https://github.com/doocs/md) 项目进行fork 或者clone 到本地。

### 创建Cloudflare 账号
前端[cloudflare/dash](https://dash.cloudflare.com/login) 注册账号，如已经有账号了，这一步跳过。

### 登录Cloudflare 账号

使用命令行操作，进入到 md项目目录，然后执行登录
```bash
cd /path/to/md
npx wrangler login
```

`npx wrangler login`会 引导打开浏览器进行登录。

### 编辑及部署

建议使用`pnpm` 打包命令。
还是在原来的目录。
```bash
pnpm install # 安装依赖
pnpm web wrangler:deploy # 部署
```

最重要的命令是 `deploy`，执行成功后可以看到类似下面的输出，


![](https://fastly.jsdelivr.net/gh/bucketio/img9@main/2025/10/06/1759727426751-3cca9a93-c00a-4a40-ac37-492bdaf8dec7.png)

可以看到deploy 成功后，cloudflare workers已经分配好了一个访问域名，比如`https://md.honwhy-wang.workers.dev`，通过这个域名就可以使用编辑器了。

### 使用建议

既然采用了cloudflare workers进行项目部署，建议使用公众号图床管理自己的图片资源，不再担心图床资源收费和不够额度问题。

#### 配置图床

**如何获取appID和appsecret 后文会有讲解。**
![](https://fastly.jsdelivr.net/gh/bucketio/img6@main/2025/10/06/1759728495461-b1f89459-a7d2-4e84-ba3c-adebb369658a.png)

#### 切换默认图床

![](https://fastly.jsdelivr.net/gh/bucketio/img10@main/2025/10/06/1759728522342-c5c6ab22-fae3-452e-8fd8-393ae4a1403b.png)


### 更新

如果md 项目更新迭代了，只需要在原来的目录执行，
```bash
git pull
pnpm web wrangler:deploy
```
重新部署后，就可以使用上迭代后的功能了。

### 题外话

之所以适配`cloudflare workers`的部署方式，目的是为了使用它的后端服务能力，进一步说是为了解决对接微信公众号图床的问题。

#### 独立域名

虽然cloudflare workers 提供了二级*.workers.dev 域名，但是在不使用代理的情况，该域名可能无法解析到正确的IP。

最好的解决办法就是配置独立域名。你需要
- 购买域名
- 域名交给cloudflare 进行管理

配置独立域名方式很简单，登录cloudflare 控制台，进入我们新建的`md` workers，打开配置tab，


![](https://fastly.jsdelivr.net/gh/bucketio/img1@main/2025/10/06/1759727977762-52c35ef1-900e-403e-ac78-e58c08394858.png)

在【域和路由】中点击添加，在右侧抽屉中选择

![](https://fastly.jsdelivr.net/gh/bucketio/img11@main/2025/10/06/1759728036727-4c8834b8-cabf-432f-96c2-be355e103a53.png)

输入，比如`md.honwhy.wang`

![](https://fastly.jsdelivr.net/gh/bucketio/img8@main/2025/10/06/1759728094741-0cdf5fb7-be94-4c99-9130-6d10dd88babd.png)

配置生效后（大约10分钟内），即可通过新的域名进行访问。

#### 关于对接公众号图床

公众号后他地址：`https://mp.weixin.qq.com/`

1、使用了公众号的openapi进行对接
2、需要开通公众号开发者账号

登录公众号后台，访问 设置与开发>基本配置，启用开发者，并走账号登录认证流程获取AppSecret，注意保存AppSecret。

![](https://fastly.jsdelivr.net/gh/bucketio/img19@main/2025/10/06/1759728364616-a3b5ff3e-60b7-4fbd-b38e-864c28c8ae76.png)

3、需要配置IP白名单
由于微信做了安全限制，从cloudflare workers后台访问微信openapi 服务需要做IP地址放行。

打开网页控制台（F12）,选择公众号图床，配置好账号密钥，尝试上传一张图片。如果有看到相关IP被限制的提示，则将这个IP复制下来。

配置白名单

通过 设置与开发>基本配置的IP白名单， 或者 设置与开发 > 安全中心 进行配置


![](https://fastly.jsdelivr.net/gh/bucketio/img8@main/2025/10/06/1759728739903-5e12769d-7cac-40a9-abb3-8bb34a73f7e2.png)


![](https://fastly.jsdelivr.net/gh/bucketio/img13@main/2025/10/06/1759729065968-e253cadf-116c-4678-a100-9c91b6720967.png)

注意：
- IP至多可输入200个
- 多个IP配置请用换行分割
- IP段仅支持 /8、/16、/24 网段，不支持其他网段
- 不支持IP:端口形式

#### 原理解释

前端不直接访问公众号openapi接口，首先访问workers的后台接口，比如部署访问网址是`https://md.honwhy.wang`，那么上传图片访问接口是
`https://md.honwhy-wang/cgi-bin/media/uploadimg`。这个workers后台接口收到请求转发到`https://api.weixin.qq.com/cgi-bin/media/uploadimg` 接口。

为什么要这么处理，原因是从`https://md.honwhy.wang` 直接访问`https://api.weixin.qq.com` 存在跨域问题。
