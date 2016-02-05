---
layout: post
title:  "Navigating the Changes"
date:   "2015-01-07  2:00:00"

---

在发布 Beta 14 版本之前，我们花费了很长时间在 ionic 社区上搜集导航 UI 应该如何表现，不只是关于过渡动画，而是所有的元素怎么在一个 app 里整合起来。Beta 14 中在导航的结构和元素上我们遵循了很多非常棒并且已经很成熟的模式，这些模式在所有平台上的 app 里随处可见。我们赶脚这些更新可以让你很平滑的去构建导航的 UI。



<!-- more -->
###Nav buttons
在 Beta 14 版本中我们新增了一个新的navbar 集成结构（inheritance structure，翻译成这个咋这别扭...）。在`<ion-nav-buttons>` 或
`<ion-nav-title>` 下的任何元素的集合都会被默认的 header 或者顶级视图（一般是在 index.html 文件中）中的内容替换掉。

在很多情景下你可能只是需要一个 `<ion-nav-bar>`，然后你会在页面模板里添加一个合适的按钮。 

下面这个例子里我们定义了一个 nav-bar，右边有一个按钮，然后在新建一个有左边按钮的页面。当这个视图渲染出来之后，会同时包含这两个按钮，所以视图会继承已经在 nav-bar 里的任何按钮。

一个典型的结构可能是这样的：

```html
<!-- index.html -->
<ion-nav-bar>
  <ion-nav-buttons side="secondary">
    <button class="button" ng-click="doSomething()">
      Add
    </button>
  </ion-nav-buttons>
</ion-nav-bar>
<ion-nav-view>
</ion-nav-view>

<!-- some view template -->
<ion-view>
  <ion-nav-buttons side="primary">
    <button class="button" ng-click="doSomethingElse()">
      More
    </button>
  </ion-nav-buttons>
  <ion-content>
    Some super content here!
  </ion-content>
</ion-view>
```

###Navigation base

在发布 Beta 14 版本之前，我们先revisited 我们觉得导航 UI 应该如何表现的方式。像`ion-tab` 和 `ion-side-menus` 组件现在已经有了很多用途。就像他们对应的原生组件一样，已经成为了导航基础的部分。

让我们看看我们的 side-menus：


```html
<ion-side-menus enable-menu-with-back-views="false">

<ion-side-menu-content>
  <ion-nav-bar class="bar-positive">
    <ion-nav-back-button>
    </ion-nav-back-button>
    <ion-nav-buttons side="left">
      <button class="button button-icon button-clear ion-navicon" menu-toggle="left">
      </button>
    </ion-nav-buttons>
  </ion-nav-bar>
  <ion-nav-view name="menuContent"></ion-nav-view>
</ion-side-menu-content>

<ion-side-menu side="left">
  <ion-content>
    <ul class="list">
      <a href="#/event/check-in" class="item" menu-close>Check-in</a>
      <a href="#/event/attendees" class="item" menu-close>Attendees</a>
    </ul>
  </ion-content>
</ion-side-menu>

</ion-side-menus>
```
在左边的 side-menu 中，可能是你应用里其他页面的一个链接的列表，在 side-menu-content中你可能有一个 nav-view 来呈现这些页面。在 Beta 14 中，这些在 side-menu 中的链接将会触发导航折叠的变化。这个视图将会被当做导航的根节点，并且只有在子视图里才有过渡动画。 

现在结果就是，你可能没办法随意的在任何视图里使用 side-menus 了。使用一个菜单时候应该通过需要跳转的页面创建一个基础的导航。而如果内部再需要其他菜单时候开发者应该考虑像 [popover](http://codepen.io/ionic/pen/GpCst) 或 [modal](http://codepen.io/ionic/pen/gblny) 组件。

###Menu Toggle Visibility
在我们关于导航的调查中，我们发现已有的原生 SDK 里有一个和 side-menu 很类似的组件，导航折叠的时候会在子页面隐藏菜单开关（那个汉堡图标囧）；只有在根页面下才是可见的。我们赞同这种用户体验，并且在 `<ion-side-menus>`元素下通过 `enable-menu-with-back-views="false"` 实现了这种效果。[ionSideMenus Docs](http://ionicframework.com/docs/nightly/api/directive/ionSideMenus/).

###View LifeCycle

为了提升性能，我们改善了 ionic 缓存页面元素和数据的能力。一旦一个控制器被实例化之后将会持续整个 app 的生命周期；它只会被隐藏并且从监视周期（watch cycle 怎么翻译？）里移除。由于我们没有重建作用域，所以当我们再次进入到监视周期时候我们已经添加了相应地事件监听。（这句话怎么理解？Since we
aren’t rebuilding scope, we’ve added events for which we should listen when entering the watch cycle again.）


<table class="table">
<tr>
<td><code>$ionicView.loaded</code></td>
<td>视图已经被加载。该事件仅在视图被创建之后添加到 DOM 时发生。当离开一个已经被缓存的视图，下次再访问该视图时，事件不会被激活。loaded 事件适合添加一些对这个视图的设置代码，然而当一个视图被激活时并不推荐监听这个事件。</td>
</tr>
<tr>
<td><code>$ionicView.enter</code></td>
<td>已经完全进视图并且该视图已经被激活。不管首次加载还是访问一个缓存的视图，该事件都会发生。</td>
</tr>
<tr>
<td><code>$ionicView.leave</code></td>
<td>视图已经完全离开并且不在是激活状态的视图。不管是缓存还是销毁，该事件都会发生。</td>
</tr>
<tr>
<td><code>$ionicView.beforeEnter</code></td>
<td>视图即将进入并成为活动视图。</td>
</tr>
<tr>
<td><code>$ionicView.beforeLeave</code></td>
<td>视图即将离开并不再是活动视图。</td>
</tr>
<tr>
<td><code>$ionicView.afterEnter</code></td>
<td>视图已经完全进入并且现在是激活状态。</td>
</tr>
<tr>
<td><code>$ionicView.afterLeave</code></td>
<td>视图已经完全离开并且不再是激活状态</td>
</tr>
<tr>
<td><code>$ionicView.unloaded</code></td>
<td>视图控制器已经被销毁并且所有元素都已经被从 DOM 中移除</td>
</tr>
</table>

举个栗子:

```javascript
$scope.$on('$ionicView.afterEnter', function(){
  // Any thing you can think of
});
```



### Parting Words
通过这些更新，ionic 在导航和视图管理方面有了一个更为完善的结构。可以通过查看文档了解更多信息。

[ionNavView](http://ionicframework.com/docs/api/directive/ionNavView/)

[ionNavButtons](http://ionicframework.com/docs/api/directive/ionNavButtons/)

[ionSideMenus](http://ionicframework.com/docs/api/directive/ionSideMenus/)

[ionTabs](http://ionicframework.com/docs/api/directive/ionTabs/)


