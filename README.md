# my-cfworker

利用CloudFlare的Workers结合Github仓库实现个人网站发布文章。

演示网站：[Tangm421's Personal Blog](https://myblog.luanhui.cf)

CF的Worker号称是全球边缘计算的无服务器应用，[You write code. We handle the rest.](https://workers.cloudflare.com/)，对于免费的CF用户来说，限制请求100000/天，最多可部署30个Worker，并且附带Workers.dev路由(也就是可以通过CF提供的域名访问)，当然也能使用自定义域名。

至于Github的话，全球最大的同性交友网站，能看到这里的网友自然不必多说了。

## 使用步骤

1. 首先得有一个属于自己的一级域名，关于怎么申请免费的一级域名，自己搜搜。

2. 绑定自己的域名到CF，也就是将自己申请的域名的nameserver设置为CF提供的地址，具体查看CF介绍以及申请域名时的托管商。

3. 如何使用worker及github

    这点具体参考[Akkariin github][Akkariin Meiko]中的说明，我就是将其仓库的workers-sakurafrp.js稍微修改后丢到Worker中。

完成以上步骤之后，就可以通过Worker提供的域名访问个人网站了，域名形式类似`withered-sun-d4bf.xxx.workers.dev`，`withered-sun-d4bf`的是默认生成的worker名称，`xxx`是子域名，这两者都可以自己修改。


## 如何使用自定义域名
绑定域名到CF后，一般2小时内就可以生效。一旦生效了，就按照以下步骤执行，例如绑定的域名是 `yyyy.com`

1. **进DNS添加CNAME**，起一个二级名称，比如`blog`，目标随便指向即可，比如workers.dev，勾上代理，也就是那个shi黄色的小云朵标志。

2. **进Workers添加路由**，按照提示添加路由：`blog.yyyy.com/*`（这里的二级域名必须要在前面一步DNS中增加过CNAME记录才行！），Worker选择之前保存和部署的那个Worker名称并保存。

大概过不了几分钟，就可以使用`https://blog.yyyy.com`自定义域名访问Worker了。



*由于本人对前端不熟悉，只是将Akkariin的js拿来测试，发布文章同时还需要更新list.json文件，相对操作起来比较麻烦也容易出错。对于会前端的同学来说这个worker大有可为*

[Akkariin Meiko]:https://github.com/kasuganosoras/cloudflare-worker-blog