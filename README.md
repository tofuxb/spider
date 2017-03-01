前段时间在对博客进行了Vue前后端分离改造，导致了一个问题，网站内容无法得到搜索引擎的收录。。。这困扰了我一段时间，由于工作比较繁忙，也没有对此问题进行探究，巧在最近帮老婆写个爬虫脚本的时候，接触到了PhantomJS，顿时开窍，我可以用PhantomJS来解决呀~
> PhantomJS is a headless WebKit scriptable with a JavaScript API. It has **fast **and **native **support for various web standards: DOM handling, CSS selector, JSON, Canvas, and SVG.

于是Google一番，发现这种方案原来很多人在用了，天哪~真是凹凸曼了，科普一下，这种解决方案其实是一种旁路机制，原理就是通过Nginx配置，将搜索引擎的爬虫请求转发到一个node server，再通过PhantomJS来解析完整的HTML。废话不说，直接上代码吧！

# 准备一个PhantomJS任务脚本
首先，我们需要一个叫spider.js的文件，用于phantomjs 解析网站。
```javascript
// spider.js
"use strict";

// 单个资源等待时间，避免资源加载后还需要加载其他资源
var resourceWait = 500;
var resourceWaitTimer;

// 最大等待时间
var maxWait = 5000;
var maxWaitTimer;

// 资源计数
var resourceCount = 0;

// PhantomJS WebPage模块
var page = require('webpage').create();

// NodeJS 系统模块
var system = require('system');

// 从CLI中获取第二个参数为目标URL
var url = system.args[1];

// 设置PhantomJS视窗大小
page.viewportSize = {
    width: 1280,
    height: 1014
};

// 获取镜像
var capture = function(errCode){

    // 外部通过stdout获取页面内容
    console.log(page.content);

    // 清除计时器
    clearTimeout(maxWaitTimer);

    // 任务完成，正常退出
    phantom.exit(errCode);

};

// 资源请求并计数
page.onResourceRequested = function(req){
    resourceCount++;
    clearTimeout(resourceWaitTimer);
};

// 资源加载完毕
page.onResourceReceived = function (res) {

    // chunk模式的HTTP回包，会多次触发resourceReceived事件，需要判断资源是否已经end
    if (res.stage !== 'end'){
        return;
    }

    resourceCount--;

    if (resourceCount === 0){

        // 当页面中全部资源都加载完毕后，截取当前渲染出来的html
        // 由于onResourceReceived在资源加载完毕就立即被调用了，我们需要给一些时间让JS跑解析任务
        // 这里默认预留500毫秒
        resourceWaitTimer = setTimeout(capture, resourceWait);

    }
};

// 资源加载超时
page.onResourceTimeout = function(req){
    resouceCount--;
};

// 资源加载失败
page.onResourceError = function(err){
    resourceCount--;
};

// 打开页面
page.open(url, function (status) {

    if (status !== 'success') {

        phantom.exit(1);

    } else {

        // 当改页面的初始html返回成功后，开启定时器
        // 当到达最大时间（默认5秒）的时候，截取那一时刻渲染出来的html
        maxWaitTimer = setTimeout(function(){

            capture(2);

        }, maxWait);

    }

});
```
让我们来运行一下吧~
`$ phantomjs spider.js 'https://wj.qq.com/'`

可在终端看到渲染后的HTML结构啦！棒棒哒！

# 命令服务化
要做到响应搜索引擎爬虫的请求，我们需要将此命令服务化，通过node起个简单的web服务。
```javascript
// server.js
// ExpressJS调用方式
var express = require('express');
var app = express();

// 引入NodeJS的子进程模块
var child_process = require('child_process');

app.get('*', function(req, res){

    // 完整URL
    var url = req.protocol + '://'+ req.hostname + req.originalUrl;

    // 预渲染后的页面字符串容器
    var content = '';

    // 开启一个phantomjs子进程
    var phantom = child_process.spawn('phantomjs', ['spider.js', url]);

    // 设置stdout字符编码
    phantom.stdout.setEncoding('utf8');

    // 监听phantomjs的stdout，并拼接起来
    phantom.stdout.on('data', function(data){
        content += data.toString();
    });

    // 监听子进程退出事件
    phantom.on('exit', function(code){
        switch (code){
            case 1:
                console.log('加载失败');
                res.send('加载失败');
                break;
            case 2:
                console.log('加载超时: '+ url);
                res.send(content);
                break;
            default:
                res.send(content);
                break;
        }
    });

});

app.listen(3000, function () {
  console.log('Spider app listening on port 3000!');
});

```
运行`node server.js`，此时我们已经有了一个预渲染的web服务啦，接下来的工作便是将搜索引擎爬虫的请求转发到这个web服务，最终将渲染结果返回给爬虫。
为了防止node进程挂掉，可以使用nohup来启动，`nohup node server.js &`。

通过Nginx配置，我们可以轻松的解决这个问题。
```conf
upstream spider_server {
  server localhost:3000;
}

server {
    listen       80;
    server_name  example.com;
    
    location / {
      proxy_set_header  Host            $host:$proxy_port;
      proxy_set_header  X-Real-IP       $remote_addr;
      proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;

      if ($http_user_agent ~* "Baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest|slackbot|vkShare|W3C_Validator|bingbot|Sosospider|Sogou Pic Spider|Googlebot|360Spider") {
        proxy_pass  http://spider_server;
      }
    }
}
```
大功告成！

注：本文代码主要参考以下文章的分享，鄙人略作修改后的可用版本，感谢分享！
> 参考链接：
> https://www.mxgw.info/t/phantomjs-prerender-for-seo.html
> http://imweb.io/topic/560b402ac2317a8c3e08621c
> https://icewing.cc/linux-install-phantomjs.html
