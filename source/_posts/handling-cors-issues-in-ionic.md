title: 解决 ionic 中的 CORS 问题【译】 

date: 2015-02-28 20:17:56

tags: 

---

*译者注：本人翻译功力有限，所以文中难免有翻译不准确的地方，凑合看吧，牛逼的话你看[英文版](http://ionicframework.com/blog/handling-cors-issues-in-ionic/)的去，完事儿欢迎回来指正交流（^_^）*

如果你通过 `ionic serve` 或者 `ionic run` 命令使用或 live reload 或者访问过外部 API 结点，那么你肯定遇到过 CORS 问题，譬如下面这样：

```
XMLHttpRequest cannot load http://api.ionic.com/endpoint.
No 'Access-Control-Allow-Origin' header is present on the requested resource.
Origin 'http://localhost:8100' is therefore not allowed access.
```
那么问题来了，什么是 CORS 呢？又是什么导致了这个问题嘞？
<!-- more -->
# 什么是 CORS？
CORS=Cross origin resource sharing(跨域资源共享)

`origin` 就是你现在正在看的主站，你现在访问的是 `http://ionicframework.com/blog/handling-cors-issues-in-ionic`，那么 `origin` 就是 `ionicframework.com`。

如果说我们向 `http://cors.api.com/api` 发起一个 AJAX 请求，那么 host origin 会由被浏览器自动列入CORS请求的 Orgin header 指定好了*（原文：Say we send an AJAX request to http://cors.api.com/api, your host origin will be specified by the Origin header that is automatically included for CORS requests by the browser. ）* 由于 `ionicframework.com` 和 `api.com` 的主机并不匹配，所以在一个HTTP OPTIONS请求报头的 form 中我们所有从 `ionicframework.com` 发起的访问服务器资源的请求必修得到服务器的授权。

假如上面的请求出现错误(不被服务器允许)，那么我们是无法从服务器访问到(`api.com`上的)资源的。

让我们来看一下当你通过`ionic serve`， `ionic run`，`ionic run -l`来运行 app 的时候 `origin` 会是什么。

# 浏览器中的运行
当你运行 `ionic serve` 时发生了什么呢？

* 启动了一个本地 web 服务器
* 你的浏览器打开并定位到本地服务器地址

这让你看着你的应用加载到你电脑上一个浏览器里，地址是：`http://localhost:8100`（如果你选择了 localhost的话）。

你的 `origin` 就是 `localhost:8100`。

任何的发送到其他不是 `localhost:8100` 主机上的 AJAX 请求都会把`localhost:8100`作为他的 origin，这就会导致必须要经过一个 CORS 预检来看是否可以访问（非本机的）服务器资源。

# 设备上的运行
当你运行 `ionic run` 时发生了什么呢？

* app 所有的文件被拷贝到设备（或者模拟器）上。
* app 运行起来，触发手机/模拟器上的浏览器访问已经被拷贝上去的文件，比如：
  `file://some/path/www/index.html`。
  
因为你正在运行的 URI 是 `file://`，所以你的 `origin` 将不会存在，所以任何向外的请求都不再需要 CORS 请求。

# 在设备使用 livereload 运行

当你运行 `ionic run -l`时又发生了什么呢？

* 启动了一个本地服务器
* app 运行起来，触发手机/模拟器上的一个浏览器通过`http://192.168.1.1:8100`来运行文件(你的 本地 ip 可能是其他的)。

你的 `origin` 就会是 `192.168.1.1:8100`。

任何一个发送到不是`192.168.1.1:8100`的服务器上的 AJAX 请求都会需要进行 CORS 预检请求来看是否可以访问到该服务器上的资源。

# 在 ionic 中解决 CORS 问题
CORS 问题只有在我们通过 `ionic serve` 或者 `ionic run -l` 来运行或测试应用的时候才会遇到。

解决这个问题有两个办法：第一个，也是比较简单的一个，就是在你的 API 服务器端允许所有的 origin，然而我们并不能控制我们访问的所有的结点。我们需要的是一个不指定`origin`的请求。

我们可以通过使用代理服务器来解决这个问题。我们来看看 Ionic CLI 是怎样提供了一个易配置的代理服务器的。

# Ionic CLI代理服务器

关于代理的快速定义：

```
在计算机网络中，代理服务器就是一个服务器（计算机系统或者应用程序），是客户端发起的请求从其他服务器寻求资源的中间桥梁。
```
*原文：In computer networks, a proxy server is a server (a computer system or an application) that acts as an intermediary for requests from clients seeking resources from other servers.*

我们为了避开 CORS 问题需要做的就是有一个代理服务器，可以接收我们的请求，想 API 结点发出一个新的请求，接收结点响应，之后反馈给我们的应用，从而避开 CORS 问题。

Ionic CLI 就有给你提供一个代理服务器从而避开所有可能会遇到的 CORS 问题的能力。

由于代理服务器向你的目标主机发起了一个新的请求，所以就不会再有 `origin`，也就不再需要 CORS 了。要注意，在浏览器增加了 Origin header 是很重要的。


# 设置代理服务器

*注意，这些设置只有通过`ionic serve` 和 `ionic run -l` 运行应用才需要*

首先我们需要在 `ionic.project`文件中设置我们的代理，这会告诉我们的 Ionic 本地服务器监听这些地址，然后发送这些请求到我们的目标地址上。

在我们的应用中，当运行 `serve` 或 `run -l` 时候，我们需要把要访问的结点地址替换成代理服务器的地址。

使用gulp任务的 replace 模块来转换出口地址会简单一点。

建议的方法是设置一个 Angular Constant 来定位到我们试图代理的 API。

这就是我们下面要采用的方法。我们会同时设置一个 Angular Service 来从 API结点 获取数据。

# 设置代理路径

比如说我们想要访问 `http://cors.api.com/api`，但并不允许我们来自 `localhost`的 origin。

代理的设置包括两件事儿：在你本地 Ionic 服务器需要访问的 `path`，最终需要访问API的 `proxyUrl`。

在你的 `ionic.project` 中像这样设置：

```
{
  "name": "proxy-example",
  "app_id": "",
  "proxies": [
    {
      "path": "/api",
      "proxyUrl": "http://cors.api.com/api"
    }
  ]
}
```

通过`ionic serve`启动你的服务器。

像我们上面指定的这样，当你访问 Ionic 服务器地址 `http://localhost:8100/api` 的时候，它会以你的名义访问 `http://cors.api.com/api`。

这样，就不需要 CORS 了。

# 设置 Angular Constant
把你的 API结点设置成 Angular Constants是非常简单的一件事儿。

下面我们就来把API结点指定成为我们的代理 URL。

之后(发布时候)我们会把正式的地址作为 constant。

```
angular.module('starter', ['ionic', 'starter.controllers', 'starter.services'])
.constant('ApiEndpoint', {
  url: 'http://localhost:8100/api'
})
// For the real endpoint, we'd use this
// .constant('ApiEndpoint', {
//  url: 'http://cors.api.com/api'
// })
```

设置好之后你就能像下面这样在应用中引入`ApiEndpoint`依赖，随意调用这个constant了。

# 设置Angular Service
```
angular.module('starter.services', [])

//NOTE: We are including the constant `ApiEndpoint` to be used here.
.factory('Api', function($http, ApiEndpoint) {
  console.log('ApiEndpoint', ApiEndpoint)

  var getApiData = function() {
    return $http.get(ApiEndpoint.url + '/tasks')
      .then(function(data) {
        console.log('Got some data: ', data);
        return data;
      });
  };

  return {
    getApiData: getApiData
  };
})
```

# 通过 Gulp 自动转换地址
这个过程中，我们需要修改`gulpfile.js`来添加两个任务：添加代理和移除代理。

首先安装`replace`模块 - `npm install --save replace`

```
// `npm install --save replace`
var replace = require('replace');
var replaceFiles = ['./www/js/app.js'];

gulp.task('add-proxy', function() {
  return replace({
    regex: "http://cors.api.com/api",
    replacement: "http://localhost:8100/api",
    paths: replaceFiles,
    recursive: false,
    silent: false,
  });
})

gulp.task('remove-proxy', function() {
  return replace({
    regex: "http://localhost:8100/api",
    replacement: "http://cors.api.com/api",
    paths: replaceFiles,
    recursive: false,
    silent: false,
  });
})
```

# 结语
本教程向你展示了一个解决通过`ionic serve`或`ionic run -l`命令运行应用时候遇到的 CORS 问题的方法。

我们知道在`ionic serve`和`ionic run -l`之间转换 API 结点地址的时候可能会是个麻烦，比较建议的方法是启动一个 gulp 进程。

解决 CORS 问题最简单的方法是让 API 提供者允许所有的 hosts，然后这事儿有点儿不太现实。

使用 Angular constant 和 replace 模块可以给我们一个避开 CORS 的折中的办法。

如果你想看看完整的例子，可以看看这个[示例项目](http://github.com/driftyco/ionic-proxy-example)。

这就是你需要访问一个有 CORS 限制的 API 服务器时候需要了解的所有事儿了。

如果你还有什么疑问、问题或者想法，请在下面评论，或者在 [twitter](https://twitter.com/IonicChina) 或 [github](https://github.com/IonicChina) 上联系我们。



























