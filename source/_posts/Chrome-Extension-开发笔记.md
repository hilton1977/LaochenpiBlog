---
title: Chrome Extension 开发笔记
tags: []
categories: [java]
toc: false
date: 2020-01-17 15:22:32
---

> 之前一会有用 Chrome　扩展程序但是一直不知道原因和开发，想着平时逛逛 Nga 各种不想看的栏目和广告背景想着自己做一个插件来提升下使用体验

![](/images/chrome-extension.png)

## Chrome Extension 
谷歌扩展程序是由 html、css、js 等资源组成的一个`.crx` 资源包，可以依附于浏览器修改页面修改自定义页面`css`样式，注入或调用页面的`js`达到一些特殊功能。

谷歌插件提供了丰富的API
- 书签控制
- 网络请求控制监听，可以自定义修改请求头
- 下载控制
- 弹窗功能
- 通信功能
- 云同步存储 `chrome.stroage`

## 项目结构
### manifest.json
`manifest.json`是谷歌插件的核心配置中心，包含了插件的各种配置，详细可以参考[官方文档](https://developer.chrome.com/extensions/manifest)

``` json
{
  "manifest_version": 2,
  "name": "Nga大漩涡",
  "version": "1.0",
  "description": "Nga 论坛广告屏蔽,浏览增强",
  "author": "李时珍老陈皮",
  "icons": {
    "48": "img/icon.png",
    "128": "img/icon.png"
  },
  "browser_action": {
    "default_icon": "img/icon.png",
    "default_popup": "popup.html"
  },
  "content_scripts":
  [
    {
      // "<all_urls>" 表示匹配所有地址
      "matches": ["https://ngabbs.com/*","https://bbs.nga.cn/*"],
      // 多个JS按顺序注入
      "js": ["js/jquery-1.8.3.js", "js/nga.js"],
      // JS的注入可以随便一点，但是CSS的注意就要千万小心了，因为一不小心就可能影响全局样式
      "css": ["css/nga.css"],
      // 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle
      "run_at": "document_end"
    }
  ],
  "permissions": [
    // 右键菜单
    "contextMenus",
    // 标签
    "tabs",
    // 通知
    "notifications",
    // web请求
    "webRequest",
    // 阻塞式web请求
    "webRequestBlocking",
    // 插件本地存储
    "storage",
    // 可以通过executeScript或者insertCSS访问的网站
    "http://*/*",
    // 可以通过executeScript或者insertCSS访问的网站
    "https://*/*"
  ],
  // 后台运行脚本页
  "background": {
    "page": "background.html"
  }, 
  // web 访问的资源列表
  "web_accessible_resources": [
    "js/inject/nga.inject.js"
  ]
}
```


#### popup
`popup`是在点击插件图标弹出页面，当鼠标失去焦点后立即关闭生命周期较短所以用于做一些临时交互例如数据的展示查询等

![翻译插件](/images/translationCrx.png)

#### content_scripts
`content_scripts` 我们可以通过配置向指定页面或者所有页面注入自定义`css`和`js`，`matches`用于配置需要注入的页面，`<all_urls>`表示注入所有无限制，`js`配置脚本列表会根据__顺序__注入，`run_at`配置注入的时间可选`document_start`、 `document_end`、 `document_idle`，默认值为`document_idle`。

> 注意：`content_scripts` 可以访问到页面所有的`DOM`对象，但是无法访问页面`js`对象和函数，它的运行环境与页面运行不在同一环境位面，当我们需要访问页面的函数或者对象时可在`content_scripts`通过使用`DOM`操作像源页面注入脚本。
``` javascript
function injectJs(path) {
    let temp = document.createElement('script');
    temp.setAttribute('type', 'text/javascript');
    temp.src = chrome.extension.getURL(path);
    document.head.appendChild(temp);
}
```
可以通过`F12`查看页面元素找到我们注入的脚本与原始页面属于同一运行环境即访问调用同一环境下的所有函数和对象。
> 注意：注入脚本需要配置声明访问权限 `web_accessible_resources`，否则无法访问

#### background
`background` 后台脚本生命周期是所有脚本最长的，一直运行在浏览器后台当浏览器关闭时才结束，常用与一些需要长时间运行的脚本，通过`background`配置`page`页面或者`scripts`。

> `background` 可以访问几乎所有的 Api 且能__无限跨域__！，鉴于`backgroud`生命周期过长，长时间占用资源可能导致性能问题，可配置`persistent`为`false`，在被需求依赖时才加载空闲时消亡

#### contextMenus
右键菜单可以更丰富的操作习惯，例如翻译插件选中一段文字右键可以进行选中翻译，主要通过`chrome.centextMenus` Api 实现，我就想想在逛论坛遇到喜欢的傻吊图右键一键保存岂不是方便多了，代码如下
``` javascript
// background.js
chrome.contextMenus.create({
	type: 'normal'， // 类型，可选：["normal", "checkbox", "radio", "separator"]，默认 normal
	title: '菜单的名字', // 显示的文字，除非为“separator”类型否则此参数必需，如果类型为“selection”，可以使用%s显示选定的文本
	contexts: ['page'], // 上下文环境，可选：["all", "page", "frame", "selection", "link", "editable", "image", "video", "audio"]，默认page
	onclick: function(){}, // 单击时触发的方法
	parentId: 1, // 右键菜单项的父菜单项ID。指定父菜单项将会使此菜单项成为父菜单项的子菜单
	documentUrlPatterns: 'https://*.baidu.com/*' // 只在某些页面显示此右键菜单
});
// 删除某一个菜单项
chrome.contextMenus.remove(menuItemId)；
// 删除所有自定义右键菜单
chrome.contextMenus.removeAll();
// 更新某一个菜单项
chrome.contextMenus.update(menuItemId, updateProperties);
```
这里只是简单列举一些常用的，完整API参见：https://developer.chrome.com/extensions/contextMenus


