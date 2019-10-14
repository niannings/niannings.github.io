---
title: pwa-guide
tags:
  - PWA
date: 2019-10-14 17:18:38
---
# PWA初探

[TOC]

## PWA是什么?

> PWA（Progressive web apps，渐进式 Web 应用）运用现代的 Web API 以及传统的渐进式增强策略来创建跨平台 Web 应用程序。这些应用无处不在、功能丰富，使其具有与原生应用相同的用户体验优势。

## PWA的优势(Progressive web app advantages)
> PWA 是可被发现、易安装、可链接、独立于网络、渐进式、可重用、响应性和安全的。

- 什么是渐进式（Progressive）？  
可以用**渐进增强**来描述，就是要为现代功能强大的浏览器提供最优质的体验和最炫酷的效果，同时也能为较弱的浏览器提供还能接受的体验效果。

上面的内容都可以在[MDN相关文档](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Advantages)找到详细介绍。

## Web App Manifest

详情参考[MDN Web App Manifest](https://developer.mozilla.org/zh-CN/docs/Web/Manifest)

## PWA是如何独立于网络的

详情参考[MDN Service Worker API](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API)

> Service workers 本质上充当Web应用程序与浏览器之间的代理服务器，也可以在网络可用时作为浏览器和网络间的代理。它们旨在（除其他之外）使得能够创建有效的离线体验，拦截网络请求并基于网络是否可用以及更新的资源是否驻留在服务器上来采取适当的动作。他们还允许访问推送通知和后台同步API。

- 特点
    - 细粒度地缓存资源
    - 不会阻塞主线程：Service worker运行在worker上下文，因此它不能访问DOM。相对于驱动应用的主JavaScript线程，它运行在其他线程中，所以不会造成阻塞。它设计为完全异步，同步API（如XHR和localStorage）不能在service worker中使用。
    - 为了安全，只能是HTTPS

- 注册
> 使用 [ServiceWorkerContainer.register()](https://developer.mozilla.org/zh-CN/docs/Web/API/ServiceWorkerContainer/register) 方法首次注册service worker。如果注册成功，service worker就会被下载到客户端并尝试安装或激活（见下文），这将作用于整个域内用户可访问的URL，或者其特定子集。

### 最佳实践
- service worker 的基本架构是什么？
- 怎么注册一个 service worker？
- 一个新  service worker 的 install 及 activation 过程？
- 怎么更新 service worker？
- 它的缓存控制和自定义响应？

详情参考[MDN 使用 Service Workers](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers)

1. 请求和响应流只能被读取一次

#### fetch事件
> 每次任何被 service worker 控制的资源被请求到时，都会触发 fetch 事件，这些资源包括了指定的 scope 内的文档，和这些文档内引用的其他任何资源

一个来自MDN的例子：

```js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // 尝试读取缓存
    caches.match(event.request).then(function(resp) {
      // 读取到了直接返回，否则发起网络请求
      return resp || fetch(event.request).then(function(response) {
        return caches.open('v1').then(function(cache) {
          // 将response 拷贝并缓存
          cache.put(event.request, response.clone());
          return response;
        });
      });
    }).catch(function() {
      // 当请求没有匹配到缓存中的任何资源的时候，以及网络不可用的时候，
      // 我们的请求依然会失败
      return caches.match('/sw-test/gallery/myLittleVader.jpg');
    })
  );
});
```

#### 版本控制(更新你的service worker)
> 如果你的 service worker 已经被安装，但是刷新页面时有一个新版本的可用，新版的 service worker 会在后台安装，但是还没激活。当不再有任何已加载的页面在使用旧版的 service worker 的时候，新版本才会激活。一旦再也没有更多的这样已加载的页面，新的 service worker 就会被激活。

修改```install```事件

```js
self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open('v2').then(function(cache) {
      return cache.addAll([
        '/sw-test/',
        …

        // include other new resources for the new version...
      ]);
    })
  );
});
```

当安装发生的时候，前一个版本依然在响应请求，新的版本正在后台安装，我们调用了一个新的缓存 v2，所以前一个 v1 版本的缓存不会被扰乱。

当没有页面在使用当前的版本的时候，这个新的 service worker 就会激活并开始响应请求。

#### 清理旧缓存

```js
self.addEventListener('activate', function(event) {
  var cacheWhitelist = ['v2'];

  // 传给 waitUntil() 的 promise 会阻塞其他的事件，直到它完成。
  // 所以你可以确保你的清理操作会在你的的第一次 fetch 事件之前会完成。
  event.waitUntil(
    caches.keys().then(function(keyList) {
      return Promise.all(keyList.map(function(key) {
        if (cacheWhitelist.indexOf(key) === -1) {
          return caches.delete(key);
        }
      }));
    })
  );
});
```

## 一个来自MDN的完整service worker例子

```js
// sw.js
self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open('v1').then(function(cache) {
      return cache.addAll([
        '/sw-test/',
        '/sw-test/index.html',
        '/sw-test/style.css',
        '/sw-test/app.js',
        '/sw-test/image-list.js',
        '/sw-test/star-wars-logo.jpg',
        '/sw-test/gallery/bountyHunters.jpg',
        '/sw-test/gallery/myLittleVader.jpg',
        '/sw-test/gallery/snowTroopers.jpg'
      ]);
    })
  );
});

self.addEventListener('fetch', function(event) {
  event.respondWith(caches.match(event.request).then(function(response) {
    // caches.match() always resolves
    // but in case of success response will have value
    if (response !== undefined) {
      return response;
    } else {
      return fetch(event.request).then(function (response) {
        // response may be used only once
        // we need to save clone to put one copy in cache
        // and serve second one
        let responseClone = response.clone();
        
        caches.open('v1').then(function (cache) {
          cache.put(event.request, responseClone);
        });
        return response;
      }).catch(function () {
        return caches.match('/sw-test/gallery/myLittleVader.jpg');
      });
    }
  }));
});
```

## 下面我们尝试编写一个sw.js并观察它的运行情况

查看我的[例子](http://note.youdao.com/noteshare?id=22653cbf313e22732143a578fe771837)

# 让我们的PWA应用程序可以安装

详情参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/Progressive_web_apps/Installable_PWAs)

通过前面的工作我们已经可以实现web的离线访问了，但是离原生的app还是有些差距：安装到本地、更容易访问、全屏运行、没有浏览器界面，最终看起来更像一个本地应用。

如何做到这些？

## 要求：

- 一份网页清单，填好 正确的字段
- 网站的域必须是安全（HTTPS）的
- 一个本设备上代表应用的图标
- 一个注册好的service worker，可以让应用离线工作（这仅对于安卓设备上的Chrome浏览器是必需的）

### 一份清单(Manifest)

参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/Manifest)

把它放到head标签中

```html
<link rel="manifest" href="js13kpwa.webmanifest">
```

> 注意：过去有一些常用的扩展名用于清单：manifest.webapp 在Firefox OS应用清单中很流行，许多人使用manifest.json作为网页清单因为内容是JSON格式的。但是，.webmanifest 扩展名是在W3C清单规范 中显示指定的，所有应该坚持这种做法。

一份基本的清单：```js13kpwa.webmanifest```

```json
{
    "name": "js13kGames Progressive Web App",
    "short_name": "js13kPWA",
    "description": "Progressive Web App that lists games submitted to the A-Frame category in the js13kGames 2017 competition.",
    "icons": [
        {
            "src": "icons/icon-32.png",
            "sizes": "32x32",
            "type": "image/png"
        },
        // ...
        {
            "src": "icons/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ],
    "start_url": "/pwa-examples/js13kpwa/index.html",
    "display": "fullscreen",
    "theme_color": "#B12A34",
    "background_color": "#B12A34"
}
```
然后将我们的项目部署到github，用手机打开试试

可以打开[https://esop-fed.github.io/ani-css/](https://esop-fed.github.io/ani-css/)看看。
